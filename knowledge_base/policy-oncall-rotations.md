---
title: "On-Call Rotation Policy & Escalation Matrix"
author: "SRE Team"
tags: ["on-call", "policy", "pagerduty", "escalation", "sre", "incident", "rotation", "sev1"]
last_updated: "2024-01-05"
version: "1.6.3"
---

# On-Call Rotation Policy & Escalation Matrix

> **Audience:** All engineers at AetherFlow who participate in on-call rotations.
> **Purpose:** Define how on-call coverage works, what is expected of on-call engineers, and how alerts escalate.
> **Related Docs:** `runbook-incident-response.md`, `policy-code-review.md`, `reference-service-catalog.md`

On-call is one of the most direct connections between engineering work and customer experience. This document exists so that every engineer who carries a pager understands their responsibilities clearly, knows exactly what to do when an alert fires, and feels supported — not alone.

---

## Table of Contents

1. [On-Call Philosophy](#1-on-call-philosophy)
2. [Rotation Structure](#2-rotation-structure)
3. [PagerDuty Configuration](#3-pagerduty-configuration)
4. [On-Call Responsibilities](#4-on-call-responsibilities)
5. [Escalation Matrix](#5-escalation-matrix)
6. [Alert Triage Guide](#6-alert-triage-guide)
7. [On-Call Handoff Protocol](#7-on-call-handoff-protocol)
8. [Compensation & Time-Off Policy](#8-compensation--time-off-policy)
9. [Improving the On-Call Experience](#9-improving-the-on-call-experience)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. On-Call Philosophy

AetherFlow's on-call model is based on the principle of **team ownership**: the engineers who build a service are best positioned to operate it in production. On-call is not a separate ops function — it is part of engineering.

Equally important: **on-call should not be a burden that burns people out.** An on-call rotation that pages every night is a signal that something is broken in the system, not in the people. Engineers are encouraged — and expected — to file tickets for every noisy or actionable alert, and to push back when alert quality degrades.

---

## 2. Rotation Structure

AetherFlow uses a **tiered rotation model** with two layers:

### Tier 1 — Service On-Call (Primary Responder)

Each product service team maintains its own rotation. The Tier 1 on-call is the first person paged for alerts in their service domain.

| Rotation | Service Scope | Team | Rotation Length |
|----------|--------------|------|----------------|
| **Ingestion On-Call** | `ingestion-api`, `stream-processor` | Backend — Ingestion | 1 week |
| **Query On-Call** | `query-api`, `query-cache` | Backend — Query | 1 week |
| **Auth & Tenant On-Call** | `auth-service`, `tenant-service` | Backend — Platform | 1 week |
| **Billing On-Call** | `billing-service`, `payment-processor` | Backend — Billing | 1 week |
| **Infrastructure On-Call** | Kafka cluster, PostgreSQL, Kubernetes, networking | SRE | 1 week |

### Tier 2 — Engineering Manager Escalation

Each Tier 1 rotation has an associated Engineering Manager on-call. They are paged only if:
- A SEV-1 is unresolved after 30 minutes
- A Tier 1 responder is unreachable (no acknowledgment within 5 minutes)
- Customer data exposure is confirmed or suspected

### Tier 3 — Executive Escalation

VP of Engineering and CTO are in a Tier 3 escalation pool. They are paged only for:
- SEV-1 unresolved after 60 minutes
- Any confirmed data breach or compliance-impacting event
- Complete multi-region service outage

---

## 3. PagerDuty Configuration

All AetherFlow alerting routes through PagerDuty. The canonical service name in PagerDuty is `aetherflow-production`.

### Escalation Policies

```
Ingestion On-Call Policy
  ├── Level 1: Current Ingestion On-Call (5-min ack timeout)
  ├── Level 2: Secondary Ingestion On-Call (10-min ack timeout)
  └── Level 3: Engineering Manager On-Call (30-min from incident open)

Infrastructure On-Call Policy
  ├── Level 1: Current SRE On-Call (5-min ack timeout)
  ├── Level 2: Secondary SRE On-Call (10-min ack timeout)
  └── Level 3: Engineering Manager On-Call (30-min from incident open)
```

### Notification Rules

Personal notification rules are configured per-engineer in PagerDuty. The recommended configuration:

| Time Since Alert | Notification Method |
|-----------------|---------------------|
| 0 minutes | Push notification (PagerDuty app) + SMS |
| 2 minutes (no ack) | Phone call |
| 5 minutes (no ack) | Escalate to Level 2 |

> ⚠️ **You must install the PagerDuty mobile app and enable critical alerts.** iOS users: grant PagerDuty permission to bypass Do Not Disturb. Android users: set PagerDuty as a priority app. On-call is not optional overnight — if you are on primary, your phone must be reachable.

### Override Requests

If you need to swap on-call coverage (vacation, illness, conflict):
1. Find a willing swap partner from your team
2. Create a PagerDuty override for the affected window via the PagerDuty UI or Slack bot: `/pd override @person from [datetime] to [datetime]`
3. Post the override confirmation in `#oncall-schedule`

**Do NOT swap without creating the PagerDuty override.** "I told them verbally" has led to coverage gaps in the past.

---

## 4. On-Call Responsibilities

### During Your On-Call Week

**You are expected to:**
- Acknowledge PagerDuty alerts within **5 minutes** (day or night)
- Respond to SEV-1 and SEV-2 incidents per `runbook-incident-response.md`
- Triage SEV-3 and SEV-4 alerts and create Jira tickets within 4 business hours
- Be reachable by phone, Slack, and PagerDuty at all times
- Review the `#alerts-prod` channel at the start of each business day for overnight anomalies
- Not schedule deep-focus work (e.g., architectural design sprints) during your on-call week
- Hand off context explicitly at the end of your shift (see Section 7)

**You are NOT expected to:**
- Fix every problem yourself — escalate and pull in experts freely
- Work on feature development during an active incident
- Have deep expertise in every service — use the `reference-service-catalog.md` to find the owning team
- Stay awake for 24 hours — if you are handling a prolonged overnight incident, you are entitled to take the following morning off

### Overnight Alerts

AetherFlow does not expect engineers to work full business hours the day after a significant overnight incident. If you handled a SEV-1 between midnight and 6:00 UTC, inform your manager and take compensatory time as described in Section 8.

---

## 5. Escalation Matrix

Use this matrix during an active incident to determine who to loop in and how.

| Condition | Escalate To | Method | Expected Response |
|-----------|-------------|--------|-------------------|
| Alert fires, you need help diagnosing | Secondary on-call for your rotation | Slack DM + PagerDuty manual page | 10 min |
| SEV-1 declared | Engineering Manager On-Call | PagerDuty escalation (automatic at 30 min) or manual trigger | 15 min |
| SEV-1 unresolved at 60 min | VP of Engineering | Phone call | 10 min |
| Data loss confirmed | VP of Engineering + Legal Counsel | Phone call + email to `legal@aetherflow.com` | Immediate |
| Security breach suspected | `@security-team` in `#security-incidents` | Slack + phone call to Security Lead | Immediate |
| Kafka cluster issue | SRE Infrastructure On-Call | Slack DM + PagerDuty manual page | 10 min |
| PostgreSQL primary failure | SRE Infrastructure On-Call | PagerDuty manual page | 5 min |
| Payment processing failure | Billing On-Call + VP Engineering | PagerDuty + phone | 10 min |
| Third-party provider (AWS, Cloudflare) suspected | SRE On-Call | Slack — check provider status pages first | 15 min |

### Quick Contact Reference

> ⚠️ Actual phone numbers and personal contacts are in the **AetherFlow Engineering** 1Password vault under `on-call/contacts`. Do not store phone numbers in the wiki.

The current on-call schedule is always visible at: `https://aetherflow.pagerduty.com/schedules`

Slack aliases for on-call roles (always point to the current on-call engineer):
- `@oncall-ingestion` → Current Ingestion On-Call
- `@oncall-infra` → Current SRE On-Call
- `@oncall-manager` → Current EM On-Call

---

## 6. Alert Triage Guide

Not every PagerDuty page requires the same response. Use this guide to calibrate urgency.

### High-Signal Alerts (Wake up, act immediately)

These alerts have a strong track record of indicating real customer impact. Do not delay:

| Alert Name | Likely Cause | First Action |
|------------|-------------|--------------|
| `IngestionAPIDown` | Pod crash loop, bad deploy, network partition | Check pod status; roll back if recent deploy |
| `KafkaConsumerLagCritical` | Consumer crash, slow consumer, partition skew | See `runbook-kafka-operations.md` |
| `PostgresPrimaryUnreachable` | DB failure, network issue | See `runbook-database-failover.md` |
| `SLOBurnRateCritical` | Error rate spike consuming error budget rapidly | Check Datadog APM for error spike source |
| `CertificateExpiryImmediate` | TLS cert expiring within 24 hours | Alert Vault team; check cert-manager logs |

### Medium-Signal Alerts (Investigate promptly, not necessarily immediately)

| Alert Name | Likely Cause | First Action |
|------------|-------------|--------------|
| `KafkaConsumerLagHigh` | Processing slowdown | Check consumer metrics; may self-recover |
| `PgBouncerConnectionsHigh` | Connection pool pressure | Check active connections; consider restarting PgBouncer |
| `P99LatencyHigh` | Slow query, GC pressure, noisy neighbor | Check Datadog APM traces for the tail latency source |
| `PodRestartLoopWarning` | OOM kill, failing health check | Check pod logs; look for OOMKilled in events |

### Low-Signal / Known-Flappy Alerts

These alerts have historically fired without customer impact. Investigate during business hours; do not lose sleep:

| Alert Name | Known Issue | Status |
|------------|-------------|--------|
| `DatadogSyntheticLatencySpike-APAC` | APAC PoP has elevated baseline latency | Tracked: INFRA-884 |
| `RedisEvictionRateHigh` | Fires during large tenant batch uploads; self-clears | Tracked: PLAT-3021 |
| `EKSNodeNotReady` | Fires briefly during routine node rotation | Expected behavior; ignore if < 3 min |

> 📝 **If you encounter a flappy alert not on this list, document it here.** The next person on-call deserves the context you just earned the hard way.

---

## 7. On-Call Handoff Protocol

Handoffs happen every Monday at **09:00 UTC**. The outgoing on-call engineer is responsible for initiating the handoff — do not rely on the incoming engineer to ask.

### Handoff Checklist

Post the following in `#oncall-handoff` at the end of your shift:

```
ON-CALL HANDOFF — [ROTATION NAME]
Outgoing: @your-handle
Incoming: @their-handle
Period Covered: [Mon DD] 09:00 UTC → [Mon DD] 09:00 UTC

=== Incidents This Week ===
[ ] List any SEV-1/SEV-2 incidents with Jira PIR link
[ ] List any SEV-3s that are still open with Jira ticket

=== Ongoing Issues / Watch Items ===
[ ] Any degraded-but-not-alerting conditions (e.g., elevated P99 on query-api)
[ ] Known flapping alerts and their status
[ ] Any deployments planned in the next 24 hours

=== Alerts I Filed Tickets For ===
[ ] List new Jira tickets created from on-call activity this week

=== Tooling / Runbook Gaps ===
[ ] Anything missing from runbooks that made your week harder
[ ] Any alert that fired but lacked a runbook reference

=== Recommendations for Incoming On-Call ===
[ ] Anything they should watch closely in the first 24 hours
```

A handoff that is skipped or phoned in is a failure mode — the incoming engineer is flying blind. If you are unable to complete a proper handoff due to an active incident, flag it in `#oncall-schedule` and provide the handoff as soon as the incident is resolved.

---

## 8. Compensation & Time-Off Policy

AetherFlow's on-call compensation policy is documented in the HR system (Rippling). Key points for engineers:

### Weekday On-Call

On-call during business hours (Mon–Fri, 09:00–17:00 local) is considered part of normal engineering responsibilities. No additional compensation applies for carrying the pager during these hours.

### Evening & Weekend On-Call

Engineers carrying the pager outside business hours (evenings, weekends) receive an **on-call stipend** per week. Exact amounts are in Rippling under "On-Call Compensation."

### Incident Response Time-Off

If you handle a production incident outside of business hours that results in more than **2 hours of active work**, you are entitled to take equivalent time off during the following business week. Notify your manager via Slack — no formal approval process required for incidents < 4 hours. For longer incidents, log the time in Rippling under "On-Call Recovery."

### Opting Out of On-Call

On-call participation is an expectation for senior (L4+) and above engineers on product teams. Exceptions (medical, life circumstances) are handled confidentially with your manager. No engineer should feel unable to ask for a temporary reprieve.

---

## 9. Improving the On-Call Experience

On-call quality is a systemic property. The most impactful individual contribution you can make is to act on the problems you encounter while on-call.

### The Alert Quality Feedback Loop

After every page that required your attention, ask:
1. **Was this alert actionable?** If not, should the threshold be adjusted or the alert removed?
2. **Was this alert well-documented?** If not, update the runbook or this document.
3. **Could this have been prevented?** If yes, file a Jira ticket in the `RELIABILITY` project.

### On-Call Improvement Tickets

Tag improvement tickets with `on-call-improvement` in Jira. The SRE team reviews these in the weekly reliability sync every Thursday. High-impact improvements (alert noise reduction, runbook gaps) are prioritized in the SRE sprint.

### Reliability Reviews

After any week where on-call resulted in more than 3 pages outside business hours, the SRE team will schedule a **Reliability Review** — a 30-minute meeting with the owning team to identify systemic causes. This is not punitive. It is a systemic feedback loop.

---

## 10. Tribal Knowledge & Gotchas

- **The Slack `@oncall-*` aliases take ~2 minutes to update in PagerDuty.** At the moment of a handoff (Monday 09:00 UTC), there is a brief window where the alias may still point to the outgoing engineer. For critical pages at handoff time, check `https://aetherflow.pagerduty.com/schedules` directly.

- **PagerDuty phone calls come from rotating numbers.** Do not save the PagerDuty phone number in your contacts as a single entry — the number rotates. Save it as a label (e.g., "PagerDuty On-Call") and rely on the caller ID name, not the number.

- **The `#alerts-prod` channel is high-volume.** Do not rely on it as your sole alerting mechanism during your on-call week — many alerts auto-resolve before you could act, and the signal-to-noise ratio is poor. Your PagerDuty mobile app is the authoritative alert channel.

- **Never acknowledge-and-snooze a SEV-1 alert.** PagerDuty allows you to snooze alerts, but this suppresses escalation. If you acknowledge an alert, you own it until it is resolved or you explicitly reassign it in PagerDuty.

- **AWS us-east-1 is our primary region; us-west-2 is DR.** If an alert appears to be AWS-infrastructure related and you see widespread service disruption, check the AWS Service Health Dashboard *before* assuming it's our code. The `#infra-ops` Slack channel also surfaces AWS events automatically via the CloudWatch–Slack integration.

- **On weekends, Slack is slower.** If you need to pull in another engineer urgently and they don't respond to Slack within 5 minutes, call them via PagerDuty's manual page or use the phone numbers in 1Password. We don't expect engineers to be on Slack all weekend — we do expect them to have PagerDuty notifications enabled.

---

*Document Owner: SRE Team | Review Cycle: Quarterly | Next Review: 2024-04-05*
*See also: `runbook-incident-response.md`, `reference-service-catalog.md`*
*On-call schedule live at: https://aetherflow.pagerduty.com/schedules*
