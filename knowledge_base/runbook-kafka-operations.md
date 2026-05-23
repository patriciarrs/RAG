---
title: "Apache Kafka Cluster Operations"
author: "Platform Team"
tags: ["runbook", "kafka", "operations", "consumer-lag", "strimzi", "streaming", "sre", "platform"]
last_updated: "2024-02-10"
version: "1.1.0"
---

# Apache Kafka Cluster Operations

> **Audience:** SRE engineers and Platform engineers on-call.
> **Purpose:** Day-to-day operational procedures for AetherFlow's Apache Kafka cluster — consumer lag management, partition rebalancing, broker health, topic management, and disaster recovery.
> **Related Docs:** `adr-007-messaging-kafka.md`, `runbook-incident-response.md`, `reference-service-catalog.md`, `reference-infra-terraform.md`

> ⚠️ **During an active incident involving Kafka**, jump to [Section 3: Consumer Lag Emergency Response](#3-consumer-lag-emergency-response) or [Section 4: Broker Failure Response](#4-broker-failure-response). Read background sections afterward.

---

## Table of Contents

1. [Cluster Architecture Reference](#1-cluster-architecture-reference)
2. [Daily Health Checks](#2-daily-health-checks)
3. [Consumer Lag Emergency Response](#3-consumer-lag-emergency-response)
4. [Broker Failure Response](#4-broker-failure-response)
5. [Topic Management](#5-topic-management)
6. [Partition Rebalancing](#6-partition-rebalancing)
7. [Schema Registry Operations](#7-schema-registry-operations)
8. [Scaling the Cluster](#8-scaling-the-cluster)
9. [Monitoring & Alert Reference](#9-monitoring--alert-reference)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. Cluster Architecture Reference

AetherFlow runs a **3-broker Kafka cluster** managed by the Strimzi Kafka Operator on EKS, dedicated Kafka node group (`kafka-dedicated` taint). See `adr-007-messaging-kafka.md` for the architectural decision context.

```
┌────────────────────────────────────────────────────────────────┐
│                  AetherFlow Kafka Cluster                       │
│                  (Strimzi, Kubernetes namespace: kafka)         │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Broker 1   │  │   Broker 2   │  │   Broker 3   │         │
│  │ us-east-1a   │  │ us-east-1b   │  │ us-east-1c   │         │
│  │ m6i.2xlarge  │  │ m6i.2xlarge  │  │ m6i.2xlarge  │         │
│  │ 2TB gp3 EBS  │  │ 2TB gp3 EBS  │  │ 2TB gp3 EBS  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  ZooKeeper 1 │  │  ZooKeeper 2 │  │  ZooKeeper 3 │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                 │
│  ┌───────────────────────────────────────────┐                 │
│  │         Schema Registry (x2, HA)          │                 │
│  └───────────────────────────────────────────┘                 │
└────────────────────────────────────────────────────────────────┘
```

### Key Configuration Values

| Parameter | Value | Notes |
|-----------|-------|-------|
| Kafka version | 3.6.0 | |
| Default replication factor | 3 | All 3 brokers hold a copy |
| `min.insync.replicas` | 2 | Writes require 2/3 brokers to ack |
| Default retention | 7 days | Override per-topic for replay topics |
| Default partition count | 6 | Matches stream-processor replica count |
| Max message size | 10MB | `message.max.bytes` |
| Bootstrap servers | `kafka-bootstrap.kafka.svc:9092` (internal) | |

### Topic Inventory (Current)

| Topic | Partitions | Retention | Producers | Consumers |
|-------|-----------|-----------|-----------|-----------|
| `aetherflow.ingestion.raw-event.v1` | 6 | 7 days | ingestion-api | stream-processor |
| `aetherflow.ingestion.enriched-event.v1` | 6 | 7 days | stream-processor | metrics-aggregator, billing-service |
| `aetherflow.analytics.aggregated-metric.v2` | 3 | 30 days | stream-processor | query-api |
| `aetherflow.billing.usage-record.v1` | 3 | 90 days | billing-service | (external billing system) |
| `aetherflow.tenants.config.changelog` | 1 | Infinite (compacted) | tenant-service | tenant-service (cache invalidation) |
| `aetherflow.notifications.failed.v1` | 1 | 30 days | notification-service | (manual review) |

### Accessing the Cluster

```bash
# All kafka-* commands require the bootstrap server flag.
# Use the alias defined in ~/.bashrc on bastion hosts:
alias kfk='kafka-topics.sh --bootstrap-server kafka-bootstrap.kafka.svc:9092'

# Or from your local machine via kubectl port-forward:
kubectl port-forward -n kafka svc/aetherflow-kafka-kafka-bootstrap 9092:9092 &
export KAFKA_BOOTSTRAP="localhost:9092"

# Verify connectivity
kafka-topics.sh --bootstrap-server $KAFKA_BOOTSTRAP --list
```

---

## 2. Daily Health Checks

The on-call engineer should run this health check at the start of each business day. It takes under 5 minutes and surfaces slow-burning issues before they page.

```bash
#!/bin/bash
# scripts/kafka-daily-healthcheck.sh

BOOTSTRAP="kafka-bootstrap.kafka.svc:9092"
echo "=== Kafka Daily Health Check — $(date -u '+%Y-%m-%d %H:%M UTC') ==="

# 1. Broker availability
echo ""
echo "[1/5] Broker availability"
kafka-broker-api-versions.sh --bootstrap-server $BOOTSTRAP 2>&1 \
  | grep -E "^kafka-broker-[0-9]" | awk '{print "  "$1, $NF}'
# Expected: all 3 brokers respond with API versions

# 2. Under-replicated partitions (URP)
echo ""
echo "[2/5] Under-replicated partitions"
URP=$(kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --describe --under-replicated-partitions 2>/dev/null | wc -l)
if [ "$URP" -gt 0 ]; then
  echo "  ⚠️  WARNING: $URP under-replicated partition(s) detected"
  kafka-topics.sh --bootstrap-server $BOOTSTRAP \
    --describe --under-replicated-partitions
else
  echo "  ✅ No under-replicated partitions"
fi

# 3. Consumer lag across all groups
echo ""
echo "[3/5] Consumer group lag (non-zero groups only)"
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --describe --all-groups 2>/dev/null \
  | awk 'NR==1 || ($5 != "0" && $5 != "LAG" && $5 != "-")' \
  | column -t
# Healthy state: all groups show LAG = 0 or single-digit values

# 4. Disk usage per broker
echo ""
echo "[4/5] Broker disk usage"
kubectl exec -n kafka aetherflow-kafka-kafka-0 -- df -h /var/lib/kafka/data \
  | awk 'NR>1 {print "  Broker-0: "$5" used ("$3" / "$2")"}'
kubectl exec -n kafka aetherflow-kafka-kafka-1 -- df -h /var/lib/kafka/data \
  | awk 'NR>1 {print "  Broker-1: "$5" used ("$3" / "$2")"}'
kubectl exec -n kafka aetherflow-kafka-kafka-2 -- df -h /var/lib/kafka/data \
  | awk 'NR>1 {print "  Broker-2: "$5" used ("$3" / "$2")"}'
# Alert manually if any broker > 70% — Prometheus alert fires at 80%

# 5. Schema Registry health
echo ""
echo "[5/5] Schema Registry"
curl -sf http://schema-registry.kafka.svc:8081/subjects | \
  python3 -c "import sys,json; schemas=json.load(sys.stdin); \
  print(f'  ✅ {len(schemas)} schemas registered')" \
  || echo "  ❌ Schema Registry unreachable"

echo ""
echo "=== Health check complete ==="
```

---

## 3. Consumer Lag Emergency Response

**Alert:** `KafkaConsumerLagCritical` fires when any consumer group exceeds **100,000 messages** of lag sustained for 5 minutes. This is a SEV-2. If lag exceeds 500,000 messages or is growing without bound, escalate to SEV-1.

### Step 1: Identify the Lagging Group and Root Cause

```bash
BOOTSTRAP="kafka-bootstrap.kafka.svc:9092"

# Get a full picture of all consumer groups and their lag
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --describe --all-groups 2>/dev/null \
  | sort -k5 -rn \
  | head -30

# Focus on the lagging group (e.g., stream-processor)
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --describe --group stream-processor

# Sample output to interpret:
# GROUP             TOPIC                               PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# stream-processor  aetherflow.ingestion.raw-event.v1   0          45823001        45923001        100000
# stream-processor  aetherflow.ingestion.raw-event.v1   1          45800000        45900000        100000
# ...
```

**Diagnosing the root cause — ask these questions in order:**

| Question | How to Check | Implication |
|----------|-------------|-------------|
| Are consumer pods running? | `kubectl get pods -n production -l app=stream-processor` | If pods are down: restart them |
| Is lag growing or stable? | Watch the LAG column over 60s | Growing = active problem; Stable = throughput matched, catching up |
| Is one partition lagging more? | Compare LAG across partition rows | Skew = hot partition or stuck consumer assignment |
| Did a deployment just happen? | Check `#deployments` | Rebalance storm from rolling restart |
| Is producer throughput normal? | Check `LOG-END-OFFSET` growth rate in Grafana | If offset not growing: problem is upstream |

### Step 2: Scale Up the Consumer (Most Common Fix)

If the consumer pods are healthy but simply outnumbered by messages:

```bash
# Check current replica count
kubectl get deployment stream-processor -n production \
  -o jsonpath='{.spec.replicas}'

# Scale up — do NOT exceed the partition count (currently 6)
# Replicas > partitions = idle replicas with no work
kubectl scale deployment/stream-processor --replicas=6 -n production

# Watch lag reduce in real-time (poll every 30s)
watch -n 30 "kafka-consumer-groups.sh \
  --bootstrap-server kafka-bootstrap.kafka.svc:9092 \
  --describe --group stream-processor \
  | awk '{print \$5}' | grep -v LAG | awk '{sum+=\$1} END {print \"Total lag: \"sum}'"
```

### Step 3: Handle a Consumer Rebalance Storm

If you see consumers repeatedly joining and leaving the group (visible as rapidly changing `CONSUMER-ID` entries), a rebalance storm is in progress.

```bash
# Check the Kafka coordinator logs for rebalance events
kubectl logs -n kafka aetherflow-kafka-kafka-0 \
  --tail=100 | grep -i "rebalance\|LeaveGroup\|JoinGroup"

# Common cause: pod rolling restart where too many pods were replaced at once
# Check if a deployment is in progress
kubectl rollout status deployment/stream-processor -n production

# If a bad deployment caused the storm, roll it back:
kubectl rollout undo deployment/stream-processor -n production

# Wait for the rebalance to settle (watch CONSUMER-ID stabilize)
watch -n 10 "kafka-consumer-groups.sh \
  --bootstrap-server kafka-bootstrap.kafka.svc:9092 \
  --describe --group stream-processor | grep stream-processor"
```

> ⚠️ **The simultaneous restart trap:** Never delete more than 1 stream-processor pod at a time. Rolling restarts (`kubectl rollout restart`) replace pods one at a time — safe. Mass deletion (`kubectl delete pods -l app=stream-processor`) triggers a full rebalance — do not do this. See `runbook-incident-response.md` §9.

### Step 4: Emergency — Reset Consumer Offset (Last Resort)

Only use this if: the consumer is stuck on a poison-pill message it cannot process, AND you have VP of Engineering approval, AND you understand that **resetting the offset forward skips messages permanently**.

```bash
# First: identify the stuck partition and offset
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --describe --group stream-processor

# Option A: Skip forward by 1 message (surgical)
# Set offset to CURRENT-OFFSET + 1 on the stuck partition
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --group stream-processor \
  --topic aetherflow.ingestion.raw-event.v1:2 \  # partition 2
  --reset-offsets \
  --shift-by 1 \
  --execute

# Option B: Reset to latest (skip ALL backlog — nuclear option, requires VP approval)
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
  --group stream-processor \
  --topic aetherflow.ingestion.raw-event.v1 \
  --reset-offsets \
  --to-latest \
  --execute
# ☠️  This permanently discards all unprocessed messages in the backlog.
```

---

## 4. Broker Failure Response

**Alert:** `KafkaBrokerDown` fires when a broker pod is not Running for > 2 minutes. This is a SEV-2 (one broker down, cluster still functional with `min.insync.replicas=2`). If two brokers are simultaneously down, escalate to SEV-1 immediately — writes will be rejected.

### Step 1: Assess Scope

```bash
# Check pod status across all Kafka brokers
kubectl get pods -n kafka -l strimzi.io/name=aetherflow-kafka-kafka

# Expected: all 3 Running. If one is CrashLoopBackOff or Pending:
kubectl describe pod aetherflow-kafka-kafka-<N> -n kafka | tail -30
kubectl logs aetherflow-kafka-kafka-<N> -n kafka --tail=100
```

### Step 2: Under-Replicated Partitions (URPs)

When a broker is down, its partitions lose one replica and become under-replicated:

```bash
# How many URPs are there?
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --describe --under-replicated-partitions

# Monitor URP count — should decrease as partitions catch up after broker returns
watch -n 15 "kafka-topics.sh --bootstrap-server kafka-bootstrap.kafka.svc:9092 \
  --describe --under-replicated-partitions | wc -l"
```

**Critically:** While URPs exist, writes to affected topics require `min.insync.replicas=2` to be met by the 2 remaining brokers. **Writes still succeed** — do not panic and trigger a consumer reset. The cluster is degraded but functional.

### Step 3: Attempt Broker Recovery

```bash
# Strimzi will automatically attempt to restart a crashed broker pod.
# Check if it's in a restart loop:
kubectl get pod aetherflow-kafka-kafka-<N> -n kafka \
  --watch -o 'jsonpath={.status.containerStatuses[0].restartCount}{"\n"}'

# If restart count is growing rapidly (OOMKill loop):
# 1. Check memory usage — Kafka JVM heap is set to 6GB per broker
kubectl top pod aetherflow-kafka-kafka-<N> -n kafka

# 2. If OOMKilled: the broker needs more memory or the heap is fragmented
#    Temporary fix: delete the pod to force a clean restart
kubectl delete pod aetherflow-kafka-kafka-<N> -n kafka
# Strimzi will immediately replace it as part of the StatefulSet

# 3. If pod is Pending (node issue):
kubectl describe pod aetherflow-kafka-kafka-<N> -n kafka | grep -A5 Events
# Common: node not ready, EBS volume not detaching from old node
```

### Step 4: EBS Volume Stuck on Old Node

This is a known AWS issue that can delay broker recovery by 10–20 minutes:

```bash
# Check if the EBS volume is still attached to the old node
VOLUME_ID=$(kubectl get pvc -n kafka \
  data-0-aetherflow-kafka-kafka-<N> \
  -o jsonpath='{.spec.volumeName}' \
  | xargs kubectl get pv -o jsonpath='{.spec.csi.volumeHandle}')

aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --query 'Volumes[0].Attachments' \
  --region us-east-1

# If still attached to a terminated node, force-detach:
aws ec2 detach-volume \
  --volume-id $VOLUME_ID \
  --force \
  --region us-east-1

# Then delete the stuck pod to allow rescheduling
kubectl delete pod aetherflow-kafka-kafka-<N> -n kafka
```

### Step 5: Post-Recovery — Verify Partition Reassignment

After the broker recovers, it must catch up on replication for all partitions it leads:

```bash
# Wait for URPs to reach 0 (may take 5-20 minutes depending on how long broker was down)
watch -n 30 "kafka-topics.sh \
  --bootstrap-server kafka-bootstrap.kafka.svc:9092 \
  --describe --under-replicated-partitions | wc -l"

# Once URPs = 0, the cluster is fully healthy
# Verify preferred replica election has completed
kafka-leader-election.sh \
  --bootstrap-server $BOOTSTRAP \
  --election-type PREFERRED \
  --all-topic-partitions
```

---

## 5. Topic Management

### Creating a New Topic

All topic creation goes through Terraform + Strimzi `KafkaTopic` resources — **never use `kafka-topics.sh --create` in production**. Manual topic creation bypasses our governance and Terraform state tracking.

```yaml
# Add to infra/environments/production/kafka-topics.tf
resource "kubernetes_manifest" "topic_my_new_topic" {
  manifest = {
    apiVersion = "kafka.strimzi.io/v1beta2"
    kind       = "KafkaTopic"
    metadata = {
      name      = "aetherflow.myservice.my-event.v1"
      namespace = "kafka"
      labels = {
        "strimzi.io/cluster" = "aetherflow-kafka"
      }
    }
    spec = {
      partitions = 6        # Match consumer replica count
      replicas   = 3        # Always 3 in production
      config = {
        "retention.ms"           = "604800000"  # 7 days default
        "min.insync.replicas"    = "2"
        "cleanup.policy"         = "delete"     # or "compact" for changelogs
        "compression.type"       = "snappy"
      }
    }
  }
}
```

### Topic Naming Convention

```
aetherflow.<domain>.<event-type>.<version>

Examples:
  aetherflow.ingestion.raw-event.v1
  aetherflow.analytics.aggregated-metric.v2
  aetherflow.tenants.config.changelog       ← compacted topics omit version
```

Rules:
- Lowercase only; hyphens to separate words; dots to separate segments
- Always include a version suffix — enables non-breaking topic evolution
- Compacted (changelog) topics use the suffix `.changelog` instead of a version

### Inspecting Topic Contents

```bash
# Read the last 10 messages from a topic (raw bytes decoded as string)
kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP \
  --topic aetherflow.ingestion.raw-event.v1 \
  --from-beginning \
  --max-messages 10

# Read with Avro deserialization (requires Schema Registry access)
kafka-avro-console-consumer \
  --bootstrap-server $BOOTSTRAP \
  --topic aetherflow.ingestion.raw-event.v1 \
  --from-beginning \
  --max-messages 5 \
  --property schema.registry.url=http://schema-registry.kafka.svc:8081

# Inspect message count per partition
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --bootstrap-server $BOOTSTRAP \
  --topic aetherflow.ingestion.raw-event.v1 \
  --time -1  # -1 = latest offsets
```

---

## 6. Partition Rebalancing

Over time, partition leadership can drift unevenly across brokers — one broker may lead 4 partitions while another leads 1. This causes uneven load.

### Check Current Leadership Distribution

```bash
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --describe \
  | awk '/^Topic:/{topic=$2} /Partition:/{
      match($0, /Leader: ([0-9]+)/, arr);
      leaders[arr[1]]++
    } END {
      for (b in leaders) print "Broker " b ": " leaders[b] " leader partitions"
    }'
```

### Trigger Preferred Replica Election

If a broker recovered from failure, its formerly-led partitions may now be led by other brokers. Trigger a preferred replica election to rebalance:

```bash
# Elect preferred replicas for all topics (safe — no data movement)
kafka-leader-election.sh \
  --bootstrap-server $BOOTSTRAP \
  --election-type PREFERRED \
  --all-topic-partitions

# Verify leadership is now balanced
kafka-topics.sh --bootstrap-server $BOOTSTRAP --describe \
  | grep "Leader:" | sort | uniq -c | sort -rn
```

### Partition Reassignment (Rare — Requires SRE Lead Approval)

Full partition reassignment moves data between brokers and incurs significant I/O. Only necessary when adding or permanently removing a broker:

```bash
# Step 1: Generate a reassignment plan
kafka-reassign-partitions.sh \
  --bootstrap-server $BOOTSTRAP \
  --topics-to-move-json-file topics.json \
  --broker-list "0,1,2" \
  --generate > reassignment-plan.json

# Step 2: Review the plan carefully before executing
cat reassignment-plan.json

# Step 3: Execute (this is I/O intensive — schedule during low-traffic hours)
kafka-reassign-partitions.sh \
  --bootstrap-server $BOOTSTRAP \
  --reassignment-json-file reassignment-plan.json \
  --execute \
  --throttle 50000000  # 50MB/s throttle to limit broker I/O impact

# Step 4: Monitor progress
kafka-reassign-partitions.sh \
  --bootstrap-server $BOOTSTRAP \
  --reassignment-json-file reassignment-plan.json \
  --verify
```

---

## 7. Schema Registry Operations

### Viewing Registered Schemas

```bash
# List all registered subjects (one subject per topic per key/value)
curl -s http://schema-registry.kafka.svc:8081/subjects | python3 -m json.tool

# Get all versions of a specific schema
curl -s http://schema-registry.kafka.svc:8081/subjects/aetherflow.ingestion.raw-event.v1-value/versions

# Get the latest schema definition
curl -s http://schema-registry.kafka.svc:8081/subjects/aetherflow.ingestion.raw-event.v1-value/versions/latest \
  | python3 -m json.tool
```

### Compatibility Levels

All AetherFlow schemas use `BACKWARD` compatibility by default. This means:
- New consumers (using a new schema) can read messages written with old schemas
- Old consumers **cannot** read messages written with a new schema if fields were added without defaults

```bash
# Check compatibility level for a subject
curl -s http://schema-registry.kafka.svc:8081/config/aetherflow.ingestion.raw-event.v1-value

# Test if a new schema is compatible before registering it
curl -s -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d @new-schema.json \
  http://schema-registry.kafka.svc:8081/compatibility/subjects/aetherflow.ingestion.raw-event.v1-value/versions/latest
# Returns: {"is_compatible":true} or {"is_compatible":false}
```

> ⚠️ **Never delete a schema version in production** unless you are certain no active consumer references it. Schema deletion is irreversible and will break consumers that depend on the schema ID. Contact the Platform Team before deleting any schema.

---

## 8. Scaling the Cluster

### Adding Topics or Partitions

Adding **partitions to an existing topic** is safe but has one gotcha: messages with the same key may be routed to a different partition after the increase, breaking ordering guarantees for key-based consumers.

```bash
# Only safe if consumers are stateless with respect to partition assignment
# For stream-processor: do NOT increase raw-event.v1 partitions without also
# scaling stream-processor replicas and verifying tenant-level ordering requirements

# Add partitions to a topic (Terraform preferred; CLI for emergencies only)
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --alter \
  --topic aetherflow.ingestion.raw-event.v1 \
  --partitions 9  # Current: 6 → New: 9
```

### Vertical Scaling (Broker Instance Type)

Broker instance changes are managed via Terraform and require a rolling pod restart. Each broker is restarted one at a time — cluster remains available throughout, but URPs will temporarily appear during each restart.

Update `infra/environments/production/main.tf` → `module.eks.node_groups.kafka.instance_types` and apply via the standard Terraform workflow. The Strimzi operator handles the rolling restart automatically when node capacity changes.

### Horizontal Scaling (Adding Brokers)

Adding a fourth broker requires:
1. Terraform change to increase `module.eks.node_groups.kafka.min_size` and `desired_size`
2. Strimzi `Kafka` resource `spec.kafka.replicas` incremented from 3 to 4
3. Partition reassignment plan generated and executed to distribute load to the new broker
4. SRE lead approval and a maintenance window reservation

This procedure is documented in detail in the SRE playbook (Confluence: `Kafka Horizontal Scaling`).

---

## 9. Monitoring & Alert Reference

| Alert | Threshold | Severity | Dashboard |
|-------|-----------|----------|-----------|
| `KafkaBrokerDown` | Any broker unreachable > 2 min | SEV-2 | `d/kafka-cluster` |
| `KafkaUnderReplicatedPartitions` | URP > 0 for > 5 min | SEV-2 | `d/kafka-cluster` |
| `KafkaConsumerLagHigh` | Any group lag > 10,000 for > 5 min | Warning | `d/kafka-consumers` |
| `KafkaConsumerLagCritical` | Any group lag > 100,000 for > 5 min | SEV-2 | `d/kafka-consumers` |
| `KafkaDiskUsageHigh` | Any broker disk > 80% | Warning | `d/kafka-cluster` |
| `KafkaDiskUsageCritical` | Any broker disk > 90% | SEV-1 | `d/kafka-cluster` |
| `KafkaZooKeeperSessionExpired` | ZooKeeper session lost | SEV-2 | `d/kafka-cluster` |
| `SchemaRegistryDown` | Schema Registry unreachable > 1 min | SEV-2 | `d/kafka-cluster` |

Key Grafana dashboards:
- **Kafka Cluster Health:** `https://grafana.aetherflow.internal/d/kafka-cluster`
- **Consumer Lag:** `https://grafana.aetherflow.internal/d/kafka-consumers`
- **Producer Throughput:** `https://grafana.aetherflow.internal/d/kafka-producers`

Key Prometheus queries for manual inspection:

```promql
# Consumer lag per group per topic partition
kafka_consumergroup_lag{job="kafka-exporter"}

# Broker disk usage percentage
(kubelet_volume_stats_used_bytes{persistentvolumeclaim=~"data-0-aetherflow-kafka-kafka-.*"} /
 kubelet_volume_stats_capacity_bytes{persistentvolumeclaim=~"data-0-aetherflow-kafka-kafka-.*"}) * 100

# Messages in rate per topic
sum(rate(kafka_topic_partition_current_offset{job="kafka-exporter"}[5m])) by (topic)
```

---

## 10. Tribal Knowledge & Gotchas

- **The rebalance storm from rolling restart is real and well-documented.** When `kubectl rollout restart deployment/stream-processor` is triggered, pods are replaced one at a time. Each replacement triggers a rebalance (the leaving pod surrenders its partitions; the new pod acquires them). With 6 pods, that's 6 rebalances in sequence — each taking 3–10 seconds. Total blackout window: up to 60 seconds. This is expected. Do not escalate a lag spike immediately after a deployment.

- **ZooKeeper is still in the loop (for now).** We are running Kafka 3.6 in ZooKeeper mode, not KRaft mode. ZooKeeper is the metadata store for broker registration and topic configuration. If ZooKeeper quorum is lost (2 of 3 ZooKeeper nodes unavailable), Kafka brokers continue serving existing connections but cannot accept new consumer group joins or topic changes. KRaft migration is tracked in `PLAT-3350`.

- **Schema Registry availability is load-path-critical.** The `schema-enforcer` service caches Schema Registry results for 5 minutes. If Schema Registry goes down, `schema-enforcer` continues validating from cache for up to 5 minutes before falling back to passthrough mode (no schema validation). This is a deliberate availability trade-off. If Schema Registry is down for > 5 minutes, events will be written to Kafka without schema validation until it recovers.

- **Disk usage is our most dangerous slow-burn risk.** At 85k events/sec with Snappy compression, we write roughly 12GB/hour across the cluster (4GB per broker). With 2TB per broker and 7-day retention, our steady-state disk usage is ~672GB per broker — about 33% of capacity. If retention is extended for any topic (e.g., replay topics at 30 days), this math changes fast. Always `infracost` a retention change before applying it.

- **`--dry-run` is not available for offset reset.** The `kafka-consumer-groups.sh --reset-offsets` command does have a `--dry-run` flag that prints what it would do without executing. **Always use `--dry-run` first** before any offset reset. The output shows exactly which partition and offset it will change to. Many engineers skip this and make irreversible mistakes.

- **Strimzi rolling updates are slow by design.** When you update the Strimzi `Kafka` custom resource (e.g., bump the Kafka version or change a broker config), Strimzi performs a rolling update one broker at a time, waiting for the cluster to reach full health before proceeding to the next broker. For a 3-broker cluster this can take 15–30 minutes. Do not cancel a Strimzi rolling update mid-way — partial broker version mixes can cause split-brain partition leadership.

---

*Document Owner: Platform Team | Review Cycle: Quarterly | Next Review: 2024-05-10*
*See also: `adr-007-messaging-kafka.md`, `runbook-incident-response.md`, `reference-service-catalog.md`*
