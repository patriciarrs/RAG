---
title: "AetherFlow Microservices Catalog"
author: "Platform Team"
tags: ["service-catalog", "microservices", "architecture", "reference", "dependencies", "ports", "ownership"]
last_updated: "2024-03-01"
version: "4.0.1"
---

# AetherFlow Microservices Catalog

> **Audience:** All engineers.
> **Purpose:** Single source of truth for all production microservices — ownership, runtime details, dependencies, SLOs, and quick-start debugging links.
> **Related Docs:** `adr-009-api-grpc.md`, `adr-007-messaging-kafka.md`, `runbook-incident-response.md`, `reference-infra-terraform.md`

This catalog is updated as part of every service launch checklist. If a service is missing or a field is stale, open a PR — catalog drift is a reliability risk.

---

## Quick Reference Index

| Service | Owner | Language | Interface | SLO (Availability) | On-Call Alias |
|---------|-------|----------|-----------|--------------------|---------------|
| [ingestion-api](#ingestion-api) | Backend — Ingestion | Go | gRPC + REST (external) | 99.95% | `@oncall-ingestion` |
| [stream-processor](#stream-processor) | Backend — Ingestion | Go | Kafka consumer | 99.9% | `@oncall-ingestion` |
| [query-api](#query-api) | Backend — Query | Go | gRPC + REST (external) | 99.95% | `@oncall-query` |
| [auth-service](#auth-service) | Backend — Platform | Go | gRPC | 99.99% | `@oncall-platform` |
| [tenant-service](#tenant-service) | Backend — Platform | Go | gRPC | 99.95% | `@oncall-platform` |
| [billing-service](#billing-service) | Backend — Billing | Go | gRPC + Stripe webhooks | 99.9% | `@oncall-billing` |
| [notification-service](#notification-service) | Backend — Platform | Go | Kafka consumer + SendGrid | 99.5% | `@oncall-platform` |
| [dashboard-bff](#dashboard-bff) | Frontend | TypeScript (Node) | REST/JSON (external) | 99.9% | `@oncall-platform` |
| [schema-enforcer](#schema-enforcer) | Platform | Go | Kafka consumer | 99.9% | `@oncall-ingestion` |
| [metrics-aggregator](#metrics-aggregator) | Data Engineering | Python | Kafka consumer + TimescaleDB | 99.5% | `@oncall-infra` |

---

## Service Entries

---

### ingestion-api

The front door for all customer data. Accepts events via REST (external customers) and forwards them to Kafka after schema validation.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/ingestion-api` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Ingestion (`#team-ingestion`) |
| **Tech Lead** | @priya.nair |
| **SLO** | 99.95% availability / P99 latency < 150ms (measured at edge) |
| **External Endpoint** | `https://ingest.aetherflow.com/v1/events` (REST/JSON, customer-facing) |
| **Internal gRPC** | `ingestion-api-grpc.internal.aetherflow.com:50051` |
| **Health Check** | `GET /healthz` → HTTP 200 |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/ingestion-api |
| **Traces** | Datadog APM → service: `ingestion-api` |
| **Logs** | `kubectl logs -n production -l app=ingestion-api` |
| **Replicas (prod)** | 12 (autoscaled 8–20 via KEDA based on inbound request rate) |
| **Resources (per pod)** | Request: 500m CPU / 512Mi RAM · Limit: 2000m CPU / 1Gi RAM |

**Dependencies:**

```
ingestion-api
├── → Kafka (producer): aetherflow.ingestion.raw-event.v1
├── → auth-service (gRPC): ValidateToken on every request
├── → schema-enforcer (gRPC): ValidateSchema before publishing
├── → Redis: rate limiting (per-tenant, per-second)
└── → PostgreSQL: tenant configuration lookup (cached 60s in Redis)
```

**Key Configuration (environment variables):**

| Variable | Description |
|----------|-------------|
| `MAX_EVENT_SIZE_BYTES` | Maximum accepted event payload size (default: 1MB) |
| `RATE_LIMIT_DEFAULT_RPS` | Default per-tenant rate limit in requests/second (default: 1000) |
| `SCHEMA_VALIDATION_ENABLED` | Toggle schema enforcement; `true` in production always |
| `KAFKA_BROKERS` | Broker addresses (injected from Kubernetes secret) |

**Known Issues / Quirks:**

- Has a 90-second graceful shutdown period — do not reduce `terminationGracePeriodSeconds` below 120s. See `runbook-production-deployment.md` §9.
- The `ValidateToken` call to auth-service adds ~2–4ms to every request. If auth-service P99 degrades, ingestion-api P99 degrades in lockstep.
- Event ordering within a tenant is guaranteed only within the same Kafka partition. Events from a single tenant always map to the same partition via `tenant_id` as the partition key.

---

### stream-processor

Kafka consumer that reads raw events from `aetherflow.ingestion.raw-event.v1`, applies enrichment and transformation, and writes to downstream topics.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/stream-processor` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Ingestion (`#team-ingestion`) |
| **Tech Lead** | @priya.nair |
| **SLO** | Consumer lag < 10,000 messages sustained; P99 processing latency < 500ms |
| **Internal Interface** | Kafka consumer only (no HTTP or gRPC server) |
| **Health Check** | `GET /healthz` → HTTP 200 (exposes metrics + health on :8084) |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/stream-processor |
| **Replicas (prod)** | 6 (manually scaled; matches Kafka partition count for `raw-event.v1`) |
| **Resources (per pod)** | Request: 1000m CPU / 1Gi RAM · Limit: 4000m CPU / 2Gi RAM |

**Dependencies:**

```
stream-processor
├── ← Kafka (consumer): aetherflow.ingestion.raw-event.v1
├── → Kafka (producer): aetherflow.ingestion.enriched-event.v1
├── → Kafka (producer): aetherflow.analytics.aggregated-metric.v2
├── → tenant-service (gRPC): GetTenantConfig for enrichment rules
└── → PostgreSQL: write enriched event metadata (async batch, 1s flush)
```

**⚠️ Critical Operational Note:**
Replica count MUST match or be less than the partition count of the source topic (`aetherflow.ingestion.raw-event.v1`, currently 6 partitions). Over-scaling causes idle replicas with no partitions assigned. Under-scaling causes lag accumulation. See `runbook-kafka-operations.md` for scaling procedures.

**Known Issues / Quirks:**

- Restarting more than 2 pods simultaneously triggers a full consumer group rebalance (3–5 minute processing pause). Always use `kubectl rollout restart`, never `kubectl delete pods`. See `runbook-incident-response.md` §9.
- The enrichment step makes a synchronous gRPC call to tenant-service per event batch. If tenant-service is degraded, stream-processor throughput drops proportionally.

---

### query-api

Serves all customer-facing read queries — event searches, metric aggregations, and the Stream Replay feature.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/query-api` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Query (`#team-query`) |
| **Tech Lead** | @elena.vasquez |
| **SLO** | 99.95% availability / P99 latency < 200ms (simple queries) / P99 < 5s (aggregation queries) |
| **External Endpoint** | `https://api.aetherflow.com/v1/query` (REST/JSON, customer-facing) |
| **Internal gRPC** | `query-api-grpc.internal.aetherflow.com:50053` |
| **Health Check** | `GET /healthz` → HTTP 200 |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/query-api |
| **Replicas (prod)** | 8 (autoscaled 4–16 via HPA on CPU + custom Prometheus metric) |
| **Resources (per pod)** | Request: 500m CPU / 1Gi RAM · Limit: 2000m CPU / 4Gi RAM |

**Dependencies:**

```
query-api
├── → auth-service (gRPC): ValidateToken
├── → tenant-service (gRPC): GetTenantConfig
├── → PostgreSQL (read replica): event metadata queries
├── → TimescaleDB: metric aggregation queries (via metrics-aggregator writes)
├── → Redis: query result cache (TTL: 30s for aggregations, 5s for raw event queries)
└── → Kafka (consumer): stream-replay consumer group (for Stream Replay feature only)
```

**Key Operational Notes:**

- Uses a **dedicated PostgreSQL read replica** (`postgres-replica-query.aetherflow.internal`). Never point query-api at the primary — it will saturate connection pools.
- Aggregation queries on large time windows (> 30 days) can take 10–30 seconds for large tenants. These are expected and within SLO. Alert only if P99 for *simple* queries exceeds 200ms.
- The Stream Replay feature uses a separate Kafka consumer group (`query-api-replay`). Lag on this group does not affect the main query SLO — monitor separately.

---

### auth-service

Central authentication and authorization service. Every other service that handles customer requests calls auth-service to validate bearer tokens.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/auth-service` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Platform (`#team-platform`) |
| **Tech Lead** | @sara.lindqvist |
| **SLO** | **99.99% availability** / P99 latency < 20ms — this is the strictest SLO in the system |
| **Internal gRPC** | `auth-service-grpc.internal.aetherflow.com:50052` |
| **Health Check** | `GET /healthz` → HTTP 200 |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/auth-service |
| **Replicas (prod)** | 6 (manually maintained minimum; HPA rarely triggers) |
| **Resources (per pod)** | Request: 250m CPU / 256Mi RAM · Limit: 1000m CPU / 512Mi RAM |

**Dependencies:**

```
auth-service
├── → PostgreSQL (primary): token storage and user session records
├── → Redis: token validation cache (TTL: 60s; reduces DB load by ~95%)
└── (no Kafka dependency)
```

**⚠️ Critical Note — Cascading Failure Risk:**
Auth-service is a **universal dependency**. If it degrades, every service that validates tokens degrades simultaneously. The Redis cache provides 60s of resilience against a PostgreSQL failure. If auth-service P99 exceeds 20ms, page the Platform On-Call immediately — do not wait for the SLO burn rate alert.

**Known Issues / Quirks:**

- Token validation results are cached in Redis with a 60s TTL. Token revocation (e.g., when a user logs out or API key is deleted) takes up to 60s to propagate across the fleet. This is a known and accepted trade-off documented in the original design doc.
- gRPC is the only supported interface — there is no REST fallback for auth-service. Services on the legacy REST path (pre-ADR-009 migration) must use the temporary REST adapter at `auth-service-rest.internal.aetherflow.com:8081`, which is deprecated and will be removed in Q2 2024.

---

### tenant-service

Manages tenant configuration, feature flags, and subscription metadata. Called frequently by other services for tenant context.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/tenant-service` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Platform (`#team-platform`) |
| **Tech Lead** | @james.whitfield |
| **SLO** | 99.95% availability / P99 latency < 30ms |
| **Internal gRPC** | `tenant-service-grpc.internal.aetherflow.com:50054` |
| **Health Check** | `GET /healthz` → HTTP 200 |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/tenant-service |
| **Replicas (prod)** | 4 |
| **Resources (per pod)** | Request: 250m CPU / 256Mi RAM · Limit: 1000m CPU / 512Mi RAM |

**Dependencies:**

```
tenant-service
├── → PostgreSQL (primary): tenant configuration storage
├── → Redis: configuration cache (TTL: 120s)
├── ← Kafka (consumer): aetherflow.tenants.config.changelog (for cache invalidation)
└── → LaunchDarkly SDK: feature flag evaluation
```

**Known Issues / Quirks:**

- The Kafka changelog consumer (`aetherflow.tenants.config.changelog`) is used for cache invalidation. If it falls behind, configuration changes (e.g., rate limit updates, feature flag overrides) may take up to 120s (the Redis TTL) to propagate. This is expected.
- Tenant creation is a **synchronous operation** that triggers 4 downstream side effects (provisioning storage, setting up Kafka ACLs, creating billing records, sending welcome email). If any step fails, the whole creation is rolled back via a saga pattern. Partial failures are logged to `#alerts-prod` with the tag `tenant-provision-failure`.

---

### billing-service

Manages usage metering, invoice generation, and Stripe integration.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/billing-service` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Billing (`#team-billing`) |
| **Tech Lead** | @marcus.chen |
| **SLO** | 99.9% availability (lower than others; Stripe provides async retry guarantees) |
| **Internal gRPC** | `billing-service-grpc.internal.aetherflow.com:50055` |
| **External Webhook** | `https://api.aetherflow.com/webhooks/stripe` (Stripe event receiver) |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/billing-service |
| **Replicas (prod)** | 3 |

**Dependencies:**

```
billing-service
├── → PostgreSQL (primary): usage records, invoice history
├── ← Kafka (consumer): aetherflow.ingestion.enriched-event.v1 (for usage metering)
├── → Stripe API: invoice creation, payment processing
└── → notification-service (gRPC): SendInvoiceEmail
```

**⚠️ Deployment Note:**
Billing-service deployment is in the **production blackout window** for the last 3 business days of each month (billing cycle close). See `runbook-production-deployment.md` §2.

---

### notification-service

Handles all outbound customer notifications: email (SendGrid), Slack webhooks, and PagerDuty integrations configured by tenants.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/notification-service` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Backend — Platform |
| **SLO** | 99.5% availability / best-effort delivery (fire-and-forget from callers' perspective) |
| **Internal Interface** | Kafka consumer + gRPC server (for synchronous send requests) |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/notification-service |
| **Replicas (prod)** | 3 |

**Key Note:** Notification delivery failures do NOT surface as errors to the caller. Failed sends are retried 3× with exponential backoff, then written to the `aetherflow.notifications.failed.v1` topic for manual review. The `#alerts-notification-failures` Slack channel is fed from this topic.

---

### dashboard-bff

Backend-for-Frontend serving the AetherFlow web dashboard. Aggregates data from multiple backend services into frontend-optimized payloads.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/dashboard` |
| **Language / Runtime** | TypeScript, Node.js 20 LTS |
| **Team** | Frontend (`#team-frontend`) |
| **Tech Lead** | @yuki.tanaka |
| **SLO** | 99.9% availability / P99 < 500ms |
| **External Endpoint** | `https://app.aetherflow.com/api/*` (REST/JSON, authenticated) |
| **Health Check** | `GET /api/health` → HTTP 200 |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/dashboard-bff |
| **Replicas (prod)** | 4 |

**Note:** dashboard-bff is the only TypeScript service in the fleet. It is deployed via the same ArgoCD pipeline but uses a separate Dockerfile and Node.js base image. Frontend engineers manage its Helm chart independently.

---

### schema-enforcer

Validates event schemas against the Confluent Schema Registry before events are accepted into the main pipeline.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/aetherflow` → `/cmd/schema-enforcer` |
| **Language / Runtime** | Go 1.21 |
| **Team** | Platform |
| **SLO** | P99 latency < 10ms (called synchronously in the ingestion hot path) |
| **Internal gRPC** | `schema-enforcer-grpc.internal.aetherflow.com:50056` |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/schema-enforcer |
| **Replicas (prod)** | 4 |

**Critical:** This service is in the synchronous hot path of ingestion-api. Any degradation here directly impacts ingestion P99 latency. Schema Registry results are cached locally in schema-enforcer with a 5-minute TTL to reduce Schema Registry load.

---

### metrics-aggregator

Python-based Kafka consumer that reads enriched events and writes pre-aggregated metric rollups to TimescaleDB for the query-api to serve.

| Field | Value |
|-------|-------|
| **Repository** | `github.com/aetherflow/metrics-aggregator` |
| **Language / Runtime** | Python 3.11 |
| **Team** | Data Engineering (`#team-data`) |
| **Tech Lead** | @fatima.al-rashid |
| **SLO** | 99.5% / aggregation lag < 60 seconds |
| **Metrics Dashboard** | https://grafana.aetherflow.internal/d/metrics-aggregator |
| **Replicas (prod)** | 3 |

**Note:** This is the only Python service in the fleet. It uses `confluent-kafka-python` for Kafka consumption. The Data Engineering team owns its deployment independently via a separate ArgoCD application (`metrics-aggregator-prod`).

---

## Adding a New Service to This Catalog

When launching a new production service, open a PR adding its entry to this document before the service goes to production. Use the template below:

```markdown
### <service-name>

<One sentence description.>

| Field | Value |
|-------|-------|
| **Repository** | |
| **Language / Runtime** | |
| **Team** | |
| **Tech Lead** | |
| **SLO** | |
| **Internal gRPC / External Endpoint** | |
| **Health Check** | |
| **Metrics Dashboard** | |
| **Replicas (prod)** | |
| **Resources (per pod)** | |

**Dependencies:**
\`\`\`
<service-name>
├── → ...
└── ← ...
\`\`\`
```

---

*Document Owner: Platform Team | Review Cycle: Monthly or on new service launch*
*Last full audit: 2024-03-01 | Next audit: 2024-04-01*
*See also: `adr-009-api-grpc.md`, `adr-007-messaging-kafka.md`, `reference-infra-terraform.md`*
