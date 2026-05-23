---
title: "PostgreSQL Failover & Recovery Runbook"
author: "SRE Team"
tags: ["runbook", "postgresql", "database", "failover", "rds", "disaster-recovery", "pgbouncer", "sre"]
last_updated: "2023-09-18"
version: "1.2.4"
---

# PostgreSQL Failover & Recovery Runbook

> **Audience:** SRE on-call engineers and senior backend engineers.
> **Purpose:** Step-by-step procedures for handling PostgreSQL primary failures, forced failovers, read replica promotion, and post-failover recovery.
> **Related Docs:** `runbook-incident-response.md`, `runbook-production-deployment.md`, `reference-service-catalog.md`, `reference-infra-terraform.md`

> ⚠️ **If you are reading this during an active incident**, jump directly to [Section 3: Emergency Failover Procedure](#3-emergency-failover-procedure). Read the context sections after the incident.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Failure Detection & Symptoms](#2-failure-detection--symptoms)
3. [Emergency Failover Procedure](#3-emergency-failover-procedure)
4. [Controlled Failover (Planned Maintenance)](#4-controlled-failover-planned-maintenance)
5. [Read Replica Failure](#5-read-replica-failure)
6. [PgBouncer Recovery](#6-pgbouncer-recovery)
7. [Post-Failover Validation](#7-post-failover-validation)
8. [Point-in-Time Recovery (PITR)](#8-point-in-time-recovery-pitr)
9. [Monitoring & Alerting Reference](#9-monitoring--alerting-reference)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. Architecture Overview

```
                        ┌─────────────────────────────────┐
                        │        AWS RDS Multi-AZ          │
                        │                                  │
Application Pods        │  ┌──────────────┐               │
(via PgBouncer)  ──────▶│  │   PRIMARY    │  us-east-1a   │
                        │  │  (r6g.2xl)   │               │
                        │  └──────┬───────┘               │
                        │         │ synchronous replication │
                        │  ┌──────▼───────┐               │
                        │  │   STANDBY    │  us-east-1b   │
                        │  │  (r6g.2xl)   │               │
                        │  └──────────────┘               │
                        └─────────────────────────────────┘
                                  │
                        ┌─────────▼─────────┐
                        │   READ REPLICA     │  us-east-1c
                        │   (r6g.xlarge)     │  (async replication)
                        │   query-api only   │
                        └───────────────────┘

Connection path:
  App pods → PgBouncer (transaction pooling) → RDS CNAME endpoint
  RDS endpoint: postgres-primary.aetherflow.internal (CNAME → RDS DNS)
  Replica endpoint: postgres-replica-query.aetherflow.internal
```

### Key Endpoints

| Endpoint | DNS | Purpose |
|----------|-----|---------|
| Primary write | `postgres-primary.aetherflow.internal` | All writes; read by most services |
| Primary via PgBouncer | `pgbouncer.aetherflow.internal:5432` | Connection-pooled path (used by app pods) |
| Query read replica | `postgres-replica-query.aetherflow.internal` | query-api reads only |
| RDS native DNS | `aetherflow-prod-primary.xxxx.us-east-1.rds.amazonaws.com` | Direct RDS access (SRE use only) |

> **Critical:** Application pods connect through PgBouncer, not directly to RDS. During failover, the key question is always: *has PgBouncer reconnected to the new primary?* See Section 6.

### Connection Limits

| Layer | Max Connections | Current Steady-State |
|-------|----------------|---------------------|
| RDS instance (`max_connections`) | 900 | ~180 (PgBouncer server-side pool) |
| PgBouncer server-side pool | 180 | ~140 active |
| PgBouncer client-side (app pods) | 500 | ~320 active |

If client connections exceed 500, PgBouncer queues new connections. If server connections approach 180, reduce PgBouncer pool size before scaling app pods.

---

## 2. Failure Detection & Symptoms

### Alert: `PostgresPrimaryUnreachable`

This is a **SEV-1 alert**. Page fires when the RDS primary endpoint has been unreachable for > 60 seconds from at least 2 of 3 monitoring probes.

**Symptoms you will observe:**

- Services logging `pq: could not connect to server` or `connection refused`
- PgBouncer logs: `closing because: server conn crashed?` or `ERROR: no more connections allowed`
- ingestion-api: 500s on all write endpoints
- query-api: 500s on non-cached queries
- stream-processor: consumer lag growing (write path to enriched-event metadata blocked)
- Grafana: database connection pool utilization drops to 0 and then spikes as reconnects storm

### Alert: `PostgresReplicationLagHigh`

Fires when the read replica replication lag exceeds **30 seconds**. This does NOT indicate a primary failure — it means the replica is falling behind. query-api reads may return stale data.

**Common causes:**
- Heavy write load on primary (large ingestion burst)
- Replica instance under CPU/IO pressure
- Vacuum storms on large tables (`events`, `metrics`)

### Alert: `PgBouncerConnectionsHigh`

Fires when PgBouncer client connections exceed 450 (90% of the 500 limit). Not a primary failure — see Section 6.

---

## 3. Emergency Failover Procedure

> **This section assumes the RDS primary is unresponsive and RDS Multi-AZ automatic failover has either not triggered or has triggered but services have not recovered.**

### Step 0: Declare SEV-1

```bash
# Post in #war-room immediately:
# "SEV-1 declared: PostgreSQL primary unreachable.
#  IC: @you. Investigating failover status."
```

### Step 1: Determine RDS Failover Status

```bash
# Check the RDS event log for failover events (last 30 minutes)
aws rds describe-events \
  --source-identifier aetherflow-prod-primary \
  --source-type db-instance \
  --duration 30 \
  --region us-east-1 \
  --query 'Events[*].[Date,Message]' \
  --output table

# Also check current instance status
aws rds describe-db-instances \
  --db-instance-identifier aetherflow-prod-primary \
  --region us-east-1 \
  --query 'DBInstances[0].{Status:DBInstanceStatus,AZ:AvailabilityZone,Endpoint:Endpoint.Address}' \
  --output table
```

**Interpret the output:**

- If you see event `"Multi-AZ instance failover completed"` → RDS has already failed over. The DNS endpoint now points to the former standby. Proceed to Step 3 (PgBouncer reconnection).
- If you see `"Multi-AZ instance failover started"` but NOT completed → RDS failover is in progress (typically 60–120 seconds). Wait and re-poll every 30 seconds.
- If NO failover event and instance status is `"available"` → The RDS instance may be healthy but unreachable from our VPC. Check security groups and VPC routing. Do NOT force a failover yet.
- If instance status is `"failed"` or `"storage-full"` → Proceed to Step 2.

### Step 2: Force RDS Failover (if automatic failover has not triggered)

> Only execute this step if: (a) the primary is confirmed unresponsive for > 3 minutes AND (b) automatic failover has not triggered.

```bash
# Force a Multi-AZ failover — promotes the standby to primary
# This causes ~60-120 seconds of additional downtime while the promotion completes
aws rds reboot-db-instance \
  --db-instance-identifier aetherflow-prod-primary \
  --force-failover \
  --region us-east-1

# Monitor the failover event
watch -n 10 "aws rds describe-events \
  --source-identifier aetherflow-prod-primary \
  --source-type db-instance \
  --duration 10 \
  --region us-east-1 \
  --query 'Events[*].[Date,Message]' \
  --output table"
```

### Step 3: Force PgBouncer to Reconnect

After RDS failover completes, the RDS DNS endpoint CNAME updates automatically. However, **PgBouncer caches the resolved IP address** of the primary and will NOT pick up the new IP until its server-side connections are recycled.

```bash
# Step 3a: Identify PgBouncer pods
kubectl get pods -n production -l app=pgbouncer

# Step 3b: Force a rolling restart of PgBouncer
# This is safe — app pods will briefly queue new connections (< 5s per pod)
kubectl rollout restart deployment/pgbouncer -n production

# Step 3c: Watch the restart
kubectl rollout status deployment/pgbouncer -n production --timeout=3m

# Step 3d: Verify PgBouncer is connecting to the NEW primary IP
# The new primary will be in a different AZ than the old one
kubectl exec -n production deploy/pgbouncer -- \
  psql -h pgbouncer -p 5432 -U aetherflow_admin -d pgbouncer \
  -c "SHOW SERVERS;" | grep -v idle
```

### Step 4: Verify Application Recovery

```bash
# Check that ingestion-api can write
kubectl exec -n production deploy/ingestion-api -- \
  /bin/sh -c 'psql $DATABASE_URL -c "SELECT 1;"'

# Check error rates across services
# Open: https://grafana.aetherflow.internal/d/service-health
# Confirm error rates are returning to baseline

# Check PgBouncer pool statistics
kubectl exec -n production deploy/pgbouncer -- \
  psql -h pgbouncer -p 5432 -U aetherflow_admin -d pgbouncer \
  -c "SHOW POOLS;"
```

### Step 5: Post-Failover Immediate Actions

```bash
# 1. Confirm the new primary's AZ (for awareness, not action)
aws rds describe-db-instances \
  --db-instance-identifier aetherflow-prod-primary \
  --region us-east-1 \
  --query 'DBInstances[0].{AZ:AvailabilityZone}' \
  --output text

# 2. Note: after Multi-AZ failover, the instance is temporarily running
#    WITHOUT a synchronous standby. AWS will provision a new standby
#    in the original AZ automatically — this takes 5-15 minutes.
#    Monitor for the "Multi-AZ instance created" event.
aws rds describe-events \
  --source-identifier aetherflow-prod-primary \
  --source-type db-instance \
  --duration 30 \
  --region us-east-1 \
  --query 'Events[*].[Date,Message]' \
  --output table

# 3. Once the standby is re-established, we are back to full HA.
#    Until then, we are running on a single node — treat this as elevated risk.
```

---

## 4. Controlled Failover (Planned Maintenance)

For RDS version upgrades, instance class changes, or AZ rebalancing, use a controlled failover during a maintenance window.

```bash
# Step 1: Notify in #deployments and #war-room
# "Planned PostgreSQL failover beginning at HH:MM UTC. Expect ~90s of DB unavailability."

# Step 2: Put ingestion-api into degraded mode (queues events in Redis temporarily)
# This feature is toggled via LaunchDarkly flag: "db-degraded-mode"
# Contact @priya.nair or @sara.lindqvist to enable

# Step 3: Trigger controlled failover
aws rds reboot-db-instance \
  --db-instance-identifier aetherflow-prod-primary \
  --force-failover \
  --region us-east-1

# Step 4: Watch and time the failover
time aws rds wait db-instance-available \
  --db-instance-identifier aetherflow-prod-primary \
  --region us-east-1

# Step 5: Restart PgBouncer (same as Section 3, Step 3b)
kubectl rollout restart deployment/pgbouncer -n production

# Step 6: Disable degraded mode in LaunchDarkly
# Step 7: Validate (Section 7)
```

---

## 5. Read Replica Failure

The read replica (`postgres-replica-query.aetherflow.internal`) is used exclusively by query-api. Its failure does NOT affect write operations or any other service.

### Symptoms of Read Replica Failure

- `PostgresReplicaUnreachable` alert fires
- query-api logs: `pq: could not connect to server` on replica connection
- query-api P99 latency degrades if it falls back to the primary for reads

### Recovery Options

**Option A: Wait for automatic recovery** (preferred for transient failures)

AWS RDS will attempt to automatically restart or replace a failed replica. Monitor the RDS event log. If the replica recovers within 5 minutes with no customer impact, no further action is needed.

**Option B: Redirect query-api to primary read replica** (if Option A fails)

```bash
# Update the query-api DATABASE_REPLICA_URL environment variable
# to point at the primary (temporary measure — primary will take on extra load)
kubectl set env deployment/query-api \
  DATABASE_REPLICA_URL="$(kubectl get secret postgres-primary-credentials \
    -n production -o jsonpath='{.data.url}' | base64 -d)" \
  -n production

# Monitor primary connection count — should not exceed 300
kubectl exec -n production deploy/pgbouncer -- \
  psql -h pgbouncer -p 5432 -U aetherflow_admin -d pgbouncer \
  -c "SHOW POOLS;" | grep query
```

**Option C: Recreate the read replica via Terraform**

```bash
cd infra/environments/production

# The replica is defined as module.postgres_replica_query
# To recreate: taint the resource and apply
terraform taint module.postgres_replica_query.aws_db_instance.this
terraform plan -target=module.postgres_replica_query -out=tfplan
# Review plan — confirm it only recreates the replica
terraform apply tfplan
```

> ⚠️ Option C causes ~15–20 minutes of replica unavailability while the new instance is provisioned and initial sync completes.

---

## 6. PgBouncer Recovery

PgBouncer is the connection pooler sitting between application pods and RDS. Most "database issues" that aren't actual RDS failures are PgBouncer problems.

### Diagnosing PgBouncer Issues

```bash
# Connect to the PgBouncer admin console
kubectl exec -it -n production deploy/pgbouncer -- \
  psql -h 127.0.0.1 -p 5432 -U pgbouncer -d pgbouncer

# Useful admin commands once connected:
SHOW POOLS;     -- Current pool state, wait counts, idle counts
SHOW CLIENTS;   -- All client connections and their states
SHOW SERVERS;   -- Server-side (RDS) connections
SHOW CONFIG;    -- Active PgBouncer configuration
SHOW STATS;     -- Aggregate request and latency statistics
```

### Scenario: PgBouncer pods restarting in a loop

```bash
# Check the last 50 log lines for the failing pod
kubectl logs -n production deploy/pgbouncer --tail=50

# Common causes:
# 1. RDS credentials changed — check the Kubernetes secret
kubectl get secret postgres-primary-credentials -n production -o yaml

# 2. RDS endpoint changed (post-failover) — verify via dig
dig postgres-primary.aetherflow.internal

# 3. Wrong pg_hba.conf — check Kubernetes ConfigMap
kubectl get configmap pgbouncer-config -n production -o yaml
```

### Scenario: PgBouncer zombie process (known issue PLAT-2847)

Approximately 1 in 10 PgBouncer restarts results in a zombie process that holds port 5432 open on the pod without actually serving connections. Signs: `kubectl get pods` shows the pod Running, but `SHOW POOLS` returns no active pools.

```bash
# Identify the zombie
kubectl exec -n production <pgbouncer-pod-name> -- \
  ps aux | grep pgbouncer

# Kill the zombie process (note: this requires the pod to have kill permissions)
kubectl exec -n production <pgbouncer-pod-name> -- \
  kill -9 <PID>

# PgBouncer will restart via its supervisor (s6-overlay)
# Verify recovery within 10 seconds
kubectl exec -n production <pgbouncer-pod-name> -- \
  psql -h 127.0.0.1 -p 5432 -U pgbouncer -d pgbouncer -c "SHOW POOLS;"
```

If kill permissions are not available, delete the pod and let Kubernetes reschedule it:

```bash
kubectl delete pod <pgbouncer-pod-name> -n production
# Kubernetes will immediately replace it from the Deployment
```

---

## 7. Post-Failover Validation

Run this checklist after **any** failover, emergency or planned, before declaring the incident resolved.

```bash
#!/bin/bash
# scripts/db-post-failover-validate.sh

echo "=== PostgreSQL Post-Failover Validation ==="

# 1. Confirm primary is available
echo "[1/6] RDS primary status..."
STATUS=$(aws rds describe-db-instances \
  --db-instance-identifier aetherflow-prod-primary \
  --region us-east-1 \
  --query 'DBInstances[0].DBInstanceStatus' \
  --output text)
echo "  Status: $STATUS"
[ "$STATUS" = "available" ] && echo "  ✅" || echo "  ❌ FAIL"

# 2. Confirm Multi-AZ standby is being provisioned
echo "[2/6] Multi-AZ standby status..."
MULTIAZ=$(aws rds describe-db-instances \
  --db-instance-identifier aetherflow-prod-primary \
  --region us-east-1 \
  --query 'DBInstances[0].MultiAZ' \
  --output text)
echo "  Multi-AZ: $MULTIAZ"
[ "$MULTIAZ" = "True" ] && echo "  ✅" || echo "  ⚠️  Standby not yet restored — monitor"

# 3. PgBouncer can reach primary
echo "[3/6] PgBouncer connectivity..."
kubectl exec -n production deploy/pgbouncer -- \
  psql -h pgbouncer -U aetherflow_admin -d aetherflow \
  -c "SELECT pg_is_in_recovery();" 2>&1
# Expected: f (false = not a replica = we are on primary)

# 4. Write test
echo "[4/6] Write path test..."
kubectl exec -n production deploy/ingestion-api -- \
  /bin/sh -c 'psql $DATABASE_URL -c \
  "INSERT INTO _healthcheck (ts) VALUES (NOW()) RETURNING id;"'

# 5. Read replica replication lag
echo "[5/6] Read replica lag..."
aws rds describe-db-instances \
  --db-instance-identifier aetherflow-prod-replica-query \
  --region us-east-1 \
  --query 'DBInstances[0].{Status:DBInstanceStatus}' \
  --output table

# 6. Application error rates
echo "[6/6] Check Grafana for error rate normalization:"
echo "  https://grafana.aetherflow.internal/d/service-health"
echo "  All services should be back within SLO within 5 minutes of PgBouncer recovery."
```

---

## 8. Point-in-Time Recovery (PITR)

> **Warning:** PITR creates a new RDS instance. It does NOT overwrite the existing primary. Use PITR only for data recovery (accidental deletion, corruption), never for availability recovery.

PITR requires VP of Engineering approval before execution — it is destructive to application state if misapplied.

```bash
# Restore to a specific point in time (up to 30 days back, per our backup retention)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier aetherflow-prod-primary \
  --target-db-instance-identifier aetherflow-prod-pitr-recovery-$(date +%Y%m%d) \
  --restore-time "2023-09-17T14:30:00Z" \  # Adjust to target time (UTC)
  --db-instance-class db.r6g.2xlarge \
  --no-multi-az \                           # Not HA — recovery instance only
  --region us-east-1

# Monitor restoration progress
aws rds wait db-instance-available \
  --db-instance-identifier aetherflow-prod-pitr-recovery-$(date +%Y%m%d) \
  --region us-east-1

# Connect to the recovery instance to inspect/export data
# DO NOT point application traffic at this instance without SRE lead approval
```

After PITR recovery, work with Data Engineering and the owning team to identify the specific rows/tables affected and apply targeted SQL to reconcile the production instance. Full instance swap is extremely rare and requires a separate runbook.

---

## 9. Monitoring & Alerting Reference

| Metric | Source | Alert Threshold | Dashboard |
|--------|--------|----------------|-----------|
| RDS primary reachability | Prometheus blackbox | Unreachable > 60s → SEV-1 | `d/postgres-health` |
| PgBouncer client connections | Prometheus `pgbouncer_exporter` | > 450 → warning, > 480 → SEV-2 | `d/pgbouncer` |
| PgBouncer server connections | Prometheus | > 160 → warning | `d/pgbouncer` |
| Replication lag (replica) | CloudWatch `ReplicaLag` | > 30s → warning, > 120s → SEV-2 | `d/postgres-health` |
| RDS FreeStorageSpace | CloudWatch | < 100GB → warning, < 20GB → SEV-1 | `d/postgres-health` |
| RDS CPU | CloudWatch | > 80% sustained 5m → warning | `d/postgres-health` |
| Long-running transactions | Prometheus `pg_locks` | Transaction > 5 min → warning | `d/postgres-health` |

Key Grafana dashboards:
- **PostgreSQL Health:** `https://grafana.aetherflow.internal/d/postgres-health`
- **PgBouncer:** `https://grafana.aetherflow.internal/d/pgbouncer`
- **RDS Storage & IOPS:** `https://grafana.aetherflow.internal/d/rds-storage`

---

## 10. Tribal Knowledge & Gotchas

- **DNS TTL after failover is the silent killer.** The RDS CNAME endpoint updates immediately post-failover, but individual pods (and PgBouncer) cache the resolved IP. PgBouncer caches it until its server connections are recycled. App pods that hold open connections to PgBouncer will see failures until PgBouncer reconnects. The rolling restart in Step 3 is the fastest path to recovery — do not skip it expecting things to self-heal.

- **`pg_is_in_recovery()` is your ground truth.** After a failover, always run `SELECT pg_is_in_recovery();` against the endpoint you believe is primary. It returns `f` (false) if you are connected to the actual primary. If it returns `t` (true), you are connected to a replica — something is wrong with your routing.

- **`max_connections` on RDS r6g.2xlarge is 900, not unlimited.** At peak we use ~180 server-side connections from PgBouncer. If connection count approaches 500+, RDS will start refusing new connections. PgBouncer's transaction pooling is what keeps us well below this ceiling — do not remove the pooler.

- **Vacuum storms on `events` and `metrics` tables.** These two tables accumulate >10M rows/day. Autovacuum runs aggressively on them and can spike CPU and I/O on the primary for 10–15 minute windows. These show up in CloudWatch as CPU spikes and sometimes trigger the `P99LatencyHigh` alert. Before escalating an apparent slowness incident, check the RDS Performance Insights for `autovacuum` workers. It's a false alarm if vacuum is the cause.

- **PITR to the same region only.** AWS RDS PITR cannot restore cross-region. Our DR plan (us-west-2) uses a separate RDS instance with logical replication, not PITR. For cross-region recovery, contact the SRE lead and reference the DR runbook (currently in draft: `runbook-disaster-recovery.md`).

- **The read replica has async replication — there is no durability guarantee.** During a primary failure, the read replica may not have received the last N seconds of writes. Never use the read replica as a primary substitute for writes. It is strictly read-only and may be slightly behind.

---

*Document Owner: SRE Team | Review Cycle: Quarterly | Next Review: 2023-12-18*
*See also: `runbook-incident-response.md`, `reference-infra-terraform.md`, `reference-service-catalog.md`*
