---
title: "ADR-010: Adopt OpenTelemetry for Unified Observability"
author: "SRE Team"
tags: ["adr", "observability", "opentelemetry", "otel", "tracing", "metrics", "logging", "architecture", "current"]
last_updated: "2024-02-28"
version: "1.0.0"
---

# ADR-010: Adopt OpenTelemetry for Unified Observability

> ✅ **CURRENT DECISION** — Active as of 2024-02-28. This ADR establishes OpenTelemetry (OTel) as the standard instrumentation framework across all AetherFlow services.
> Migration from vendor-specific SDKs (Datadog tracer, custom Prometheus client wrappers) to the OTel SDK is in progress on the schedule in Section 7.

---

## Status

**Accepted**

| Field | Value |
|-------|-------|
| Date Proposed | 2024-01-22 |
| Date Accepted | 2024-02-28 |
| Proposers | @elena.vasquez (SRE), @priya.nair (Platform) |
| Stakeholders | SRE, Platform, all Backend teams |
| Supersedes | Nothing formally — this establishes a new standard where none existed |

---

## Context

AetherFlow's observability stack grew organically, with each team adopting whatever tooling solved their immediate problem. By late 2023, we had:

- **Tracing:** Direct Datadog APM SDK (`gopkg.in/DataDog/dd-trace-go.v1`) imported in every service — 11 services with Datadog-specific trace propagation
- **Metrics:** A mix of direct `prometheus/client_golang` with non-standard label conventions across teams, plus Datadog custom metrics for billing-related counters
- **Logging:** Structured JSON via `go.uber.org/zap`, but with inconsistent field names (`tenantID` vs `tenant_id` vs `tid` across different services)

### Problems This Caused

**Problem 1: Vendor lock-in with real cost.** Datadog APM pricing scales with ingested spans. At 85k events/sec, our tracing volume had grown to cost **$28,000/month** in Datadog APM alone. Migrating to a different tracing backend (e.g., self-hosted Tempo or Jaeger for most traffic, Datadog only for sampled high-value traces) required touching every service's import graph because the Datadog SDK was directly embedded.

**Problem 2: No correlation between signals.** A trace in Datadog had a `dd_trace_id`. A log line in Loki had a `request_id`. A metric in Prometheus had a `pod` label. Correlating all three during an incident required manual copy-paste of IDs across three UIs — a workflow that slows down every investigation.

**Problem 3: Inconsistent instrumentation quality.** Some services had excellent span coverage; others had a single root span and no child spans. There was no standard for what *must* be instrumented. Engineers instrumented based on intuition rather than a shared contract.

**Problem 4: Brittle collector topology.** Each service emitted telemetry directly to Datadog's intake endpoint. There was no buffering, no sampling layer, and no ability to route telemetry to alternative backends for cost or compliance reasons without modifying application code.

---

## Decision

**AetherFlow will adopt OpenTelemetry (OTel) as the standard instrumentation framework for all telemetry — traces, metrics, and logs — across all services.**

Specifically:
- All services will use the **OTel Go SDK** for instrumentation
- An **OTel Collector** fleet (deployed as a DaemonSet) will receive all telemetry from application pods and route it to backends
- **Traces:** OTel → Collector → Tempo (self-hosted, for 100% retention at 7 days) + sampled to Datadog (5% of traces for Datadog dashboards)
- **Metrics:** OTel → Collector → Prometheus (existing, unchanged) via the Prometheus exporter
- **Logs:** Structured JSON via `zap` with OTel trace context injected, scraped by Loki/Promtail
- A **semantic conventions standard** (Section 5) will be enforced via a lint rule in CI

### Why OpenTelemetry Specifically

| Requirement | OTel | Vendor SDK |
|-------------|------|-----------|
| Vendor-neutral (can switch backends without code changes) | ✅ | ❌ |
| Single SDK for traces + metrics + logs | ✅ | ❌ (Datadog APM + statsd + logrus = 3 SDKs) |
| Automatic instrumentation for HTTP, gRPC, DB | ✅ (contrib packages) | ✅ |
| W3C TraceContext propagation (standard) | ✅ | ❌ (Datadog uses B3/proprietary) |
| Collector-based routing and sampling | ✅ | ❌ |
| CNCF project (long-term viability) | ✅ | ❌ |

### Why Tempo for Trace Storage

Grafana Tempo is a horizontally scalable, object-storage-backed distributed tracing backend:
- **Cost:** Tempo stores traces in S3 at ~$0.023/GB. At our volume, 100% trace retention for 7 days costs ~$400/month vs. $28,000/month for Datadog APM
- **Integration:** Native Grafana datasource — trace correlation with Loki logs and Prometheus metrics in a single Grafana query
- **Operational:** Deployed on EKS via the `grafana/tempo-distributed` Helm chart; SRE already operates Grafana

---

## Implementation Standard

### Go SDK Setup

All services must initialize the OTel SDK using the shared `pkg/telemetry` package. **Do not import `go.opentelemetry.io/otel` directly in service code** — always go through `pkg/telemetry` to ensure consistent configuration.

```go
// go/pkg/telemetry/telemetry.go
// Standard AetherFlow OTel initialization package.
// All services call telemetry.Init() in main() before starting servers.
package telemetry

import (
    "context"
    "fmt"
    "os"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/exporters/prometheus"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
    "go.uber.org/zap"
)

type Config struct {
    ServiceName    string
    ServiceVersion string
    Environment    string  // "production", "staging", "development"
    OTLPEndpoint   string  // OTel Collector endpoint
}

// Init initializes the global OTel TracerProvider and MeterProvider.
// Returns a shutdown function that must be called on service exit.
func Init(ctx context.Context, cfg Config) (shutdown func(context.Context) error, err error) {
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(cfg.ServiceName),
            semconv.ServiceVersion(cfg.ServiceVersion),
            semconv.DeploymentEnvironment(cfg.Environment),
            // AetherFlow-specific resource attributes
            attribute.String("aetherflow.team", os.Getenv("SERVICE_TEAM")),
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create OTel resource: %w", err)
    }

    // Trace exporter — sends to OTel Collector via gRPC
    traceExporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint(cfg.OTLPEndpoint),
        otlptracegrpc.WithInsecure(), // mTLS handled by service mesh
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create trace exporter: %w", err)
    }

    // Tracer provider with head-based sampling
    // 100% sampling in dev/staging; 10% in production (Collector tail-samples the rest)
    var sampler sdktrace.Sampler
    if cfg.Environment == "production" {
        sampler = sdktrace.TraceIDRatioBased(0.10)
    } else {
        sampler = sdktrace.AlwaysSample()
    }

    tracerProvider := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(traceExporter,
            sdktrace.WithBatchTimeout(5*time.Second),
            sdktrace.WithMaxExportBatchSize(512),
        ),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sampler),
    )
    otel.SetTracerProvider(tracerProvider)

    // W3C TraceContext + Baggage propagation (replaces Datadog B3 propagation)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    // Metric exporter — Prometheus pull-based (no change to existing Prometheus setup)
    metricExporter, err := prometheus.New()
    if err != nil {
        return nil, fmt.Errorf("failed to create metric exporter: %w", err)
    }
    meterProvider := metric.NewMeterProvider(
        metric.WithReader(metricExporter),
        metric.WithResource(res),
    )
    otel.SetMeterProvider(meterProvider)

    shutdown = func(ctx context.Context) error {
        if err := tracerProvider.Shutdown(ctx); err != nil {
            zap.L().Error("trace provider shutdown error", zap.Error(err))
        }
        return meterProvider.Shutdown(ctx)
    }
    return shutdown, nil
}
```

```go
// cmd/ingestion-api/main.go — standard initialization pattern
func main() {
    ctx := context.Background()

    shutdown, err := telemetry.Init(ctx, telemetry.Config{
        ServiceName:    "ingestion-api",
        ServiceVersion: buildVersion, // injected at build time via ldflags
        Environment:    os.Getenv("AETHERFLOW_ENV"),
        OTLPEndpoint:   os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT"), // points to DaemonSet collector
    })
    if err != nil {
        log.Fatalf("failed to initialize telemetry: %v", err)
    }
    defer shutdown(ctx)

    // ... rest of service initialization
}
```

### Automatic Instrumentation (gRPC and HTTP)

Use the OTel contrib packages for automatic span creation on inbound and outbound gRPC and HTTP calls. Manual span creation should be reserved for business-logic operations.

```go
// gRPC server instrumentation — add to all gRPC servers
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

grpcServer := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

// gRPC client instrumentation — add to all gRPC client connections
conn, err := grpc.Dial(address,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    // ... other dial options
)

// HTTP server instrumentation (for dashboard-bff and external REST endpoints)
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

mux.Handle("/v1/events", otelhttp.NewHandler(
    http.HandlerFunc(handleEvents),
    "ingestion.handle_events",
))
```

### Manual Span Creation (Business Logic)

```go
// Standard pattern for manual spans on business-critical operations
import "go.opentelemetry.io/otel"

func (s *IngestionService) processEvent(ctx context.Context, event *Event) error {
    tracer := otel.Tracer("ingestion-api")

    ctx, span := tracer.Start(ctx, "ingestion.process_event",
        trace.WithAttributes(
            attribute.String("aetherflow.tenant_id", event.TenantID),
            attribute.String("aetherflow.event_type", event.EventType),
            attribute.Int("aetherflow.payload_bytes", len(event.Payload)),
        ),
    )
    defer span.End()

    if err := s.validateSchema(ctx, event); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "schema validation failed")
        return fmt.Errorf("schema validation: %w", err)
    }

    span.AddEvent("schema_validated")

    if err := s.publishToKafka(ctx, event); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "kafka publish failed")
        return fmt.Errorf("kafka publish: %w", err)
    }

    span.SetStatus(codes.Ok, "")
    return nil
}
```

---

## 5. Semantic Conventions Standard

All AetherFlow spans, metrics, and log fields must follow these conventions. Consistency is enforced by a CI lint rule (`scripts/otel-conventions-check.go`) that scans for non-conforming attribute names.

### Required Span Attributes

Every span **must** carry:

| Attribute | Type | Example |
|-----------|------|---------|
| `service.name` | string | `ingestion-api` |
| `service.version` | string | `a3f8c21` |
| `deployment.environment` | string | `production` |

Customer-context spans (any span handling a specific tenant's data) **must additionally** carry:

| Attribute | Type | Example |
|-----------|------|---------|
| `aetherflow.tenant_id` | string | `tenant-acme-corp` |
| `aetherflow.request_id` | string | `req_01HQ...` |

### Metric Naming

All custom metrics must follow this pattern:

```
aetherflow.<service_name>.<noun>.<verb>_<unit>_total

Examples:
  aetherflow_ingestion_api_events_processed_total        (counter)
  aetherflow_ingestion_api_event_payload_bytes           (histogram)
  aetherflow_stream_processor_enrichment_duration_seconds (histogram)
  aetherflow_auth_service_token_validations_total         (counter)
```

Required labels on all custom metrics:
- `service` — service name
- `environment` — deployment environment
- `result` — `success` or `error` (for counters that track outcomes)

### Log Field Conventions

All structured log fields must use `snake_case`. Required fields in every log line:

```json
{
  "timestamp": "2024-02-28T14:32:00.123Z",
  "level": "info",
  "service": "ingestion-api",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "message": "event processed successfully",
  "tenant_id": "tenant-acme-corp",
  "request_id": "req_01HQ..."
}
```

The `trace_id` and `span_id` fields are injected automatically by the `pkg/logging` package, which extracts them from the OTel span context. Loki's LogQL can then correlate log lines directly to Tempo traces.

---

## 6. OTel Collector Configuration

The OTel Collector runs as a **DaemonSet** (one pod per node), receiving telemetry from all application pods on the same node via `localhost`.

```yaml
# otel-collector-config.yaml — key pipelines excerpt
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # Application pods send here
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Add K8s metadata to all telemetry (pod name, namespace, node)
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    extract:
      metadata: [k8s.pod.name, k8s.namespace.name, k8s.node.name]

  # Tail-based sampling: always sample errors and slow requests;
  # sample 10% of healthy fast requests
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors-policy
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow-traces-policy
        type: latency
        latency: { threshold_ms: 500 }
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

  batch:
    send_batch_size: 1024
    timeout: 5s

exporters:
  # Traces → Grafana Tempo (100% of sampled traces)
  otlp/tempo:
    endpoint: tempo-distributor.monitoring.svc:4317
    tls: { insecure: true }

  # Traces → Datadog (sampled 5% — for existing Datadog dashboards during migration)
  datadog:
    api:
      key: ${env:DD_API_KEY}
    traces:
      sample_rate: 0.05

  # Metrics → Prometheus (scraped by existing Prometheus setup)
  prometheus:
    endpoint: 0.0.0.0:8889

pipelines:
  traces:
    receivers: [otlp]
    processors: [k8sattributes, tail_sampling, batch]
    exporters: [otlp/tempo, datadog]
  metrics:
    receivers: [otlp]
    processors: [k8sattributes, batch]
    exporters: [prometheus]
```

---

## 7. Migration Schedule

Services migrate to the OTel SDK on a team-by-team basis. The Datadog APM SDK remains active during the migration (both SDKs can coexist via `dd-trace-go`'s OTel bridge).

| Service | Migration Status | Target Complete | Owner |
|---------|-----------------|----------------|-------|
| `auth-service` | ✅ Done | Q4 2023 (pilot) | @sara.lindqvist |
| `ingestion-api` | 🔄 In Progress | Q1 2024 | @priya.nair |
| `stream-processor` | 🔄 In Progress | Q1 2024 | @priya.nair |
| `query-api` | 📅 Planned | Q2 2024 | @elena.vasquez |
| `tenant-service` | 📅 Planned | Q2 2024 | @james.whitfield |
| `billing-service` | 📅 Planned | Q2 2024 | @marcus.chen |
| `notification-service` | 📅 Planned | Q3 2024 | Platform |
| `dashboard-bff` | 📅 Planned | Q3 2024 | @yuki.tanaka |
| `schema-enforcer` | 📅 Planned | Q3 2024 | Platform |
| `metrics-aggregator` | 📅 Planned | Q3 2024 | @fatima.al-rashid |

Datadog APM contract ends **Q4 2024** (renewal avoided). All services must be on OTel before then.

Migration progress tracked in Jira Epic: `SRE-512 — OTel Migration`.

---

## Consequences

### Positive

- **~$320,000/year cost reduction** — Datadog APM replaced by self-hosted Tempo for bulk trace storage; Datadog retained only for sampled high-value traces and existing metrics/logging dashboards
- **Signal correlation in a single pane of glass** — Grafana can link from a Prometheus alert → Tempo trace → Loki log line using the injected `trace_id`
- **Backend flexibility** — changing trace backends in the future requires only a Collector config change, not application code changes
- **Enforced instrumentation standards** — semantic conventions CI check eliminates label inconsistency between teams

### Negative

- **Migration effort** — ~4 engineer-months estimated across all services; significant but one-time
- **Collector fleet is a new operational component** — DaemonSet failure causes telemetry loss on that node; requires monitoring (alert: `OTelCollectorDown`)
- **Tail sampling adds latency to trace export** — the 10-second `decision_wait` in the tail sampling processor means traces are buffered for up to 10 seconds in the Collector before being exported. This does not affect application latency but means incident traces may not appear in Tempo for up to ~15 seconds
- **Datadog and OTel trace IDs are different formats** — during the migration period, some traces start in Datadog format and continue in W3C TraceContext format, making cross-service correlation imperfect for in-flight requests that span migrated and unmigrated services

### Neutral

- Prometheus and Grafana (for metrics and dashboards) are unchanged
- Loki and Promtail (for log aggregation) are unchanged
- Datadog is retained for the custom metrics dashboards used by the Business Intelligence team (these dashboards are out of scope for this ADR)

---

*Document Owner: SRE Team | Review Cycle: Bi-annual or post-migration*
*Related: `reference-service-catalog.md`, `runbook-incident-response.md`, `adr-007-messaging-kafka.md`*
*Jira Epic: SRE-512 | Tempo UI: https://grafana.aetherflow.internal/explore (datasource: Tempo)*
