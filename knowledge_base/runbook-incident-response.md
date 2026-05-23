---
title: "SEV-Level Incident Response Runbook"
author: "SRE Team"
tags: ["incident-response", "on-call", "sev1", "sev2", "runbook", "war-room", "pagerduty"]
last_updated: "2024-01-15"
version: "2.3.1"
---

# SEV-Level Incident Response Runbook

> **Audience:** All on-call engineers, SREs, and engineering managers.
> **Purpose:** Standardize how AetherFlow detects, escalates, mitigates, and post-mortems production incidents.
> **Related Docs:** `policy-oncall-rotations.md`, `runbook-database-failover.md`, `runbook-kafka-operations.md`

---

## Table of Contents

1. [Severity Definitions](#1-severity-definitions)
2. [Alerting & Detection](#2-alerting--detection)
3. [Incident Commander Protocol](#3-incident-commander-protocol)
4. [Response Procedures by SEV Level](#4-response-procedures-by-sev-level)
5. [The War Room](#5-the-war-room)
6. [Escalation Matrix](#6-escalation-matrix)
7. [Mitigation Playbooks](#7-mitigation-playbooks)
8. [Post-Incident Review (PIR) Process](#8-post-incident-review-pir-process)
9. [Tribal Knowledge & Gotchas](#9-tribal-knowledge--gotchas)

---

## 1. Severity Definitions

| Level | Name | Customer Impact | Response SLA | Example |
|-------|------|----------------|--------------|---------|
| **SEV-1** | Critical | Total service outage or data loss affecting >10% of customers | Page immediately; acknowledge in **5 min** | Ingestion API returning 5xx for all tenants |
| **SEV-2** | Major | Significant degradation affecting a subset of customers or a key feature | Acknowledge in **15 min** | Kafka consumer lag >500k messages |
| **SEV-3** | Minor | Non-critical feature broken; workaround exists | Address within **4 hours** | Dashboard export timing out for specific report |
| **SEV-4** | Low | Cosmetic issues; logging anomalies | Address next business day | Stale tooltip text in UI |

> ⚠️ **Golden Rule:** When in doubt, declare a higher severity. Downgrading is cheap. Under-responding to a SEV-1 is catastrophic.

---

## 2. Alerting & Detection

### Alert Sources

AetherFlow uses a layered alerting stack:

1. **Prometheus + Alertmanager** → fires on SLO burn rates, latency P99, and error rate thresholds
2. **Datadog Synthetics** → external black-box health checks every 30 seconds from 5 global PoPs
3. **PagerDuty** → aggregation and on-call routing (see `policy-oncall-rotations.md`)
4. **Customer-reported** → Zendesk tickets tagged `production-incident` trigger a webhook into `#alerts-prod`

### Key Alerting Thresholds (as of v2.3.1)

```yaml
# Prometheus alert excerpt — prometheus/rules/aetherflow-slos.yaml
- alert: IngestionAPIErrorRateHigh
  expr: |
    sum(rate(http_requests_total{job="ingestion-api", status=~"5.."}[5m]))
    / sum(rate(http_requests_total{job="ingestion-api"}[5m])) > 0.01
  for: 2m
  labels:
    severity: sev2
  annotations:
    summary: "Ingestion API error rate > 1% for 2 minutes"
    runbook_url: "https://wiki.aetherflow.internal/runbook-incident-response"

- alert: IngestionAPIDown
  expr: up{job="ingestion-api"} == 0
  for: 1m
  labels:
    severity: sev1
  annotations:
    summary: "Ingestion API is completely down"
```

---

## 3. Incident Commander Protocol

Every SEV-1 and SEV-2 **must** have a designated **Incident Commander (IC)**. The IC is NOT necessarily the person fixing the problem — they coordinate communication and decision-making.

### IC Responsibilities

- [ ] Declare the incident and set the severity in PagerDuty
- [ ] Open the `#war-room` Slack channel (see [Section 5](#5-the-war-room))
- [ ] Assign a **Scribe** to maintain the live timeline in the incident doc
- [ ] Assign a **Communications Lead** for external customer updates
- [ ] Make the go/no-go call on rollbacks, failovers, and escalations
- [ ] Drive the incident to resolution or hand off cleanly with a written brief

### IC Handoff Script

When handing off IC duties between engineers:

```
IC HANDOFF — [DATE TIME UTC]
Outgoing IC: @person-a
Incoming IC: @person-b

Current Status: [One sentence]
Active Hypotheses: [Bullet list]
Actions In-Flight: [Who is doing what right now]
Blockers: [What needs a decision]
Stakeholders Notified: [Yes/No, last update at HH:MM]
```

---

## 4. Response Procedures by SEV Level

### SEV-1 Response (0–5 minutes)

```bash
# Step 1: Acknowledge the PagerDuty alert immediately
# Step 2: Join #war-room in Slack
# Step 3: Pull the current service health dashboard
open https://grafana.aetherflow.internal/d/service-health

# Step 4: Check recent deployments — was anything pushed in the last 2 hours?
# Check the deployment log in #deployments channel or run:
kubectl rollout history deployment/ingestion-api -n production

# Step 5: If a bad deploy is suspected, rollback FIRST, investigate SECOND
kubectl rollout undo deployment/ingestion-api -n production
```

### SEV-2 Response (0–15 minutes)

- Acknowledge in PagerDuty
- Post in `#incidents` with: severity, affected service, initial hypothesis
- Begin investigation — check Datadog APM traces for the affected time window
- If no resolution in 30 minutes, **upgrade to SEV-1**

### SEV-3 / SEV-4 Response

- No war room required
- Create a Jira ticket tagged `severity:3` or `severity:4`
- Assign to the owning team's queue
- Update the ticket at least once per day until resolved

---

## 5. The War Room

> **The `#war-room` Slack channel is the ONLY source of truth during a SEV-1 incident.** Do not use DMs, email, or side conversations for incident coordination — everything must be visible in the channel.

### War Room Etiquette

- **Pin the incident doc** at the start (template: `https://wiki.aetherflow.internal/incident-doc-template`)
- Post a status update **every 15 minutes**, even if it just says "Still investigating, no new findings"
- Use **thread replies** for technical deep-dives to keep the main channel clean
- Tag stakeholders explicitly with `@person` — do not assume people are watching
- When the incident is resolved, post: `✅ RESOLVED — [TIME UTC] — [One-line summary]`

### Mandatory War Room Posts

| Timing | Content |
|--------|---------|
| T+0 | Incident declared. IC: @name. Severity: SEV-X. Affected: [service]. |
| T+15 | Status update or hypothesis update |
| T+30 | Escalation status (are VPs/customers aware?) |
| Resolution | Resolution summary + next steps |

### Customer Status Page

External status updates go to `status.aetherflow.com` via Statuspage.io. Only the Communications Lead and engineering managers have write access. **Never post resolution updates to the status page before the system is actually stable** — premature all-clears are a repeat offense in our history.

---

## 6. Escalation Matrix

| If... | Then escalate to... | Via... |
|-------|--------------------|----|
| SEV-1 unresolved after 30 min | Engineering Manager on-call | PagerDuty escalation policy |
| Data loss confirmed or suspected | VP of Engineering + Legal | Phone call, not Slack |
| SEV-1 unresolved after 60 min | CTO | Phone call |
| Customer data exposed (any severity) | Security team + VP of Eng immediately | `#security-incidents` + phone |
| Payment processing affected | VP of Engineering + Finance | Phone call |

---

## 7. Mitigation Playbooks

### Playbook A: High Kafka Consumer Lag

See `runbook-kafka-operations.md` for full detail. Quick steps:

```bash
# Check lag across all consumer groups
kafka-consumer-groups.sh --bootstrap-server kafka-broker-1.aetherflow.internal:9092 \
  --describe --all-groups | grep -v "0$"

# If lag is growing on 'stream-processor' group, scale up consumers:
kubectl scale deployment/stream-processor --replicas=8 -n production
```

### Playbook B: Database Connection Pool Exhaustion

See `runbook-database-failover.md`. Quick steps:

```bash
# Check active connections on primary
psql -h postgres-primary.aetherflow.internal -U aetherflow_admin -c \
  "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# If > 450 connections (our pool max is 500), restart PgBouncer on affected nodes
kubectl rollout restart deployment/pgbouncer -n production
```

### Playbook C: Ingestion API Rollback

```bash
# Find the last known-good image tag from #deployments history
# Then pin the deployment to that image
kubectl set image deployment/ingestion-api \
  ingestion-api=gcr.io/aetherflow-prod/ingestion-api:<LAST_GOOD_TAG> \
  -n production

# Watch the rollout
kubectl rollout status deployment/ingestion-api -n production
```

---

## 8. Post-Incident Review (PIR) Process

A PIR (also called a postmortem) is **mandatory** for all SEV-1 incidents and **recommended** for SEV-2s.

### PIR SLAs

- Draft published: **within 24 hours** of incident resolution
- Review meeting: **within 72 hours**
- Action items tracked in Jira: **within 1 week**

### PIR Template Sections

1. **Incident Summary** — What happened, when, for how long, customer impact
2. **Timeline** — Minute-by-minute log (pull from `#war-room` thread)
3. **Root Cause Analysis** — The actual technical root cause (5 Whys encouraged)
4. **Contributing Factors** — What made this worse or harder to detect
5. **What Went Well** — Genuinely celebrate good responses
6. **What Went Poorly** — Honest, blameless assessment
7. **Action Items** — Each item needs an owner, a Jira ticket, and a due date

> **Blameless Culture:** PIRs are about systems and processes, not individuals. The AetherFlow postmortem process follows the Google SRE blameless postmortem model. Naming individuals as root causes is explicitly prohibited.

---

## 9. Tribal Knowledge & Gotchas

> This section captures hard-won lessons that aren't in any formal spec. Update it after every major incident.

- **The Kafka rebalance trap:** If you restart more than 2 stream-processor pods simultaneously, you'll trigger a full consumer group rebalance that can take 3–5 minutes and cause a processing blackout. Always use `kubectl rollout restart` (which does rolling restarts), never `kubectl delete pods`.

- **PgBouncer restart flakiness:** PgBouncer pods in production have a known issue where approximately 1 in 10 restarts results in a zombie process that holds the port. If connections don't recover within 90 seconds post-restart, SSH into the node and `kill -9` the old process manually. Tracked in JIRA: `PLAT-2847`.

- **Datadog lag:** Datadog metrics in our environment have an observed 45–90 second ingestion lag during high-volume periods. When an incident is actively unfolding, **prefer Prometheus/Grafana** for real-time data — Datadog is better for historical correlation after the fact.

- **The `#war-room` channel gets archived:** After a SEV-1 is resolved, the channel is archived automatically after 7 days by our Slack automation bot. Export the thread to the PIR doc **before** resolution if you want to preserve it.

- **Status page lag:** Changes to `status.aetherflow.com` can take up to 4 minutes to propagate to CDN edges. Factor this into communication timing.

- **EKS node IMDS race condition:** On rare occasions after an EKS node replacement, pods on the new node fail to acquire IAM credentials from the instance metadata service for up to 60 seconds. If you see `NoCredentialProviders` errors immediately after a node comes online, wait before escalating.

---

*Document Owner: SRE Team | Review Cycle: Quarterly | Next Review: 2024-04-15*
*See also: `policy-oncall-rotations.md`, `runbook-database-failover.md`, `runbook-kafka-operations.md`*
