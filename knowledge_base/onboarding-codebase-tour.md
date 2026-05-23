---
title: "Codebase Architecture Tour"
author: "Engineering Enablement Team"
tags: ["onboarding", "codebase", "architecture", "go", "repository-structure", "patterns", "new-engineer"]
last_updated: "2023-08-14"
version: "1.4.0"
---

# Codebase Architecture Tour

> **Audience:** Engineers in their first two weeks at AetherFlow.
> **Purpose:** Orient you to the shape of the codebase — where things live, why they're structured that way, and the conventions you'll encounter repeatedly.
> **Time to complete:** ~90 minutes of reading + exploration. Follow along with the actual repo open.
> **Related Docs:** `onboarding-env-setup.md`, `onboarding-first-30-days.md`, `adr-009-api-grpc.md`, `reference-service-catalog.md`

This document is not a reference manual — it's a guided tour. It will give you a mental map that makes the reference material in the rest of the wiki coherent. Read it once, then return to specific sections as you encounter unfamiliar patterns.

---

## Table of Contents

1. [Repository Overview](#1-repository-overview)
2. [The `cmd/` Directory — Service Entry Points](#2-the-cmd-directory--service-entry-points)
3. [The `internal/` Directory — Business Logic](#3-the-internal-directory--business-logic)
4. [The `pkg/` Directory — Shared Libraries](#4-the-pkg-directory--shared-libraries)
5. [The `db/` Directory — Schema and Migrations](#5-the-db-directory--schema-and-migrations)
6. [The `tests/` Directory — Integration and E2E](#6-the-tests-directory--integration-and-e2e)
7. [Key Patterns You'll See Everywhere](#7-key-patterns-youll-see-everywhere)
8. [Data Flow: Event Ingestion End-to-End](#8-data-flow-event-ingestion-end-to-end)
9. [How to Find Things](#9-how-to-find-things)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. Repository Overview

The main application repository (`github.com/aetherflow/aetherflow`) is a **multi-service monorepo**. All Go services live here. We chose this structure over separate per-service repositories to make cross-service refactors tractable and to keep shared library changes atomic with the services that consume them.

```
aetherflow/
│
├── cmd/                        # Service entry points (one subdirectory per service)
│   ├── ingestion-api/
│   ├── auth-service/
│   ├── query-api/
│   ├── tenant-service/
│   ├── billing-service/
│   ├── stream-processor/
│   ├── notification-service/
│   └── schema-enforcer/
│
├── internal/                   # Private business logic (not importable externally)
│   ├── ingestion/              # Domain logic for the ingestion-api service
│   ├── auth/                   # Domain logic for the auth-service
│   ├── query/                  # Domain logic for the query-api
│   ├── tenant/                 # Domain logic for the tenant-service
│   ├── billing/                # Domain logic for the billing-service
│   └── stream/                 # Domain logic for the stream-processor
│
├── pkg/                        # Shared libraries (importable by any service)
│   ├── messaging/              # Kafka producer/consumer client — see adr-007
│   ├── grpcclient/             # Standard gRPC client factory — see adr-009
│   ├── telemetry/              # OTel SDK initialization — see adr-010
│   ├── logging/                # Structured logger (zap wrapper + trace injection)
│   ├── middleware/             # HTTP and gRPC middleware (auth, rate limit, tracing)
│   ├── testutil/               # Test helpers (mock servers, fixture builders)
│   └── errors/                 # Canonical error types and gRPC status mapping
│
├── db/
│   ├── migrations/             # golang-migrate SQL migration files
│   └── seeds/                  # Development fixture data
│
├── tests/
│   ├── integration/            # Integration tests (tagged: //go:build integration)
│   └── e2e/                    # End-to-end tests against staging
│
├── proto/                      # Git submodule: github.com/aetherflow/proto
│   └── gen/go/                 # Generated Go stubs (do not edit manually)
│
├── scripts/                    # Developer and CI utility scripts
├── docker/                     # Docker Compose files for local development
├── k8s/                        # Kubernetes manifests and Helm chart values
├── .github/                    # GitHub Actions workflow definitions
│
├── go.work                     # Go workspace file (links aetherflow + proto/gen/go)
├── go.mod                      # Module: github.com/aetherflow/aetherflow
├── .go-version                 # Pinned Go version (read by setup-go and goenv)
└── Makefile                    # Developer convenience targets
```

The `proto/` directory is a **git submodule** pointing to `github.com/aetherflow/proto`. When you clone the main repo, run `git submodule update --init --recursive` to pull it. The generated Go stubs in `proto/gen/go/` are referenced by the `go.work` workspace file, making them importable in service code as `github.com/aetherflow/proto/gen/go/...`.

---

## 2. The `cmd/` Directory — Service Entry Points

Each subdirectory under `cmd/` is a standalone Go binary — a service. The pattern is consistent across all services:

```
cmd/ingestion-api/
├── main.go         # Entry point: wires everything together, starts servers
├── .air.toml       # Hot-reload config for local development (air)
└── Dockerfile      # Multi-stage build (builder + distroless runtime image)
```

### Anatomy of a `main.go`

Open `cmd/ingestion-api/main.go`. You'll see the same structure in every service:

```go
// cmd/ingestion-api/main.go
package main

import (
    // Standard library
    "context"
    "os"
    "os/signal"
    "syscall"

    // Internal packages
    "github.com/aetherflow/aetherflow/internal/ingestion"
    "github.com/aetherflow/aetherflow/pkg/logging"
    "github.com/aetherflow/aetherflow/pkg/telemetry"

    // Generated proto stubs
    ingestionv1 "github.com/aetherflow/proto/gen/go/aetherflow/ingestion/v1"
)

// buildVersion is injected at compile time via -ldflags
// e.g.: go build -ldflags "-X main.buildVersion=$(git rev-parse --short HEAD)"
var buildVersion = "dev"

func main() {
    // 1. Initialize structured logger (must be first — everything else logs)
    logger := logging.New(os.Getenv("LOG_LEVEL"))
    defer logger.Sync()

    ctx, cancel := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    // 2. Initialize OTel telemetry (traces, metrics)
    //    See pkg/telemetry/telemetry.go and adr-010-observability-otel.md
    shutdownTelemetry, err := telemetry.Init(ctx, telemetry.Config{
        ServiceName:    "ingestion-api",
        ServiceVersion: buildVersion,
        Environment:    os.Getenv("AETHERFLOW_ENV"),
        OTLPEndpoint:   os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT"),
    })
    if err != nil {
        logger.Fatal("failed to initialize telemetry", zap.Error(err))
    }
    defer shutdownTelemetry(ctx)

    // 3. Initialize the service (dependency injection — no global state)
    //    All dependencies are passed explicitly; no init() functions with side effects
    svc, err := ingestion.NewService(ctx, ingestion.Config{
        DatabaseURL:    os.Getenv("DATABASE_URL"),
        KafkaBrokers:   strings.Split(os.Getenv("KAFKA_BROKERS"), ","),
        SchemaRegistry: os.Getenv("SCHEMA_REGISTRY_URL"),
        Logger:         logger,
    })
    if err != nil {
        logger.Fatal("failed to initialize service", zap.Error(err))
    }

    // 4. Register gRPC handlers
    grpcServer := grpc.NewServer(
        grpc.StatsHandler(otelgrpc.NewServerHandler()),
        grpc.UnaryInterceptor(middleware.ChainUnaryInterceptors(
            middleware.AuthInterceptor(svc.AuthClient),
            middleware.RateLimitInterceptor(svc.RateLimiter),
            middleware.LoggingInterceptor(logger),
        )),
    )
    ingestionv1.RegisterIngestionServiceServer(grpcServer, svc)

    // 5. Register HTTP handlers (health + external REST endpoint)
    httpMux := http.NewServeMux()
    httpMux.HandleFunc("/healthz", svc.HandleHealthz)
    httpMux.Handle("/v1/events", otelhttp.NewHandler(
        http.HandlerFunc(svc.HandleIngestEvent),
        "ingestion.handle_events",
    ))

    // 6. Start servers and block until shutdown signal
    eg, ctx := errgroup.WithContext(ctx)
    eg.Go(func() error { return runGRPCServer(ctx, grpcServer, ":50051") })
    eg.Go(func() error { return runHTTPServer(ctx, httpMux, ":8080") })

    if err := eg.Wait(); err != nil {
        logger.Error("server exited with error", zap.Error(err))
    }
    logger.Info("shutdown complete")
}
```

**Key conventions visible in `main.go`:**
- No `init()` functions with side effects — all initialization is explicit and ordered
- Context-driven shutdown — the `signal.NotifyContext` pattern propagates cancellation cleanly
- `errgroup` for concurrent server management — if either server exits unexpectedly, the other is cancelled
- All configuration via environment variables — no config files, no flags (except in local dev tooling)

---

## 3. The `internal/` Directory — Business Logic

`internal/` is Go's built-in access control mechanism: packages under `internal/` cannot be imported by code outside the module. This ensures that service domain logic stays private to its service and is only accessible through the service's gRPC interface.

Each domain package mirrors its service's structure:

```
internal/ingestion/
├── service.go          # Service struct and constructor (NewService)
├── handler.go          # gRPC and HTTP handler implementations
├── handler_test.go     # Unit tests for handlers
├── event.go            # Core domain types (Event, EventBatch, etc.)
├── validator.go        # Schema validation logic
├── publisher.go        # Kafka publish logic
├── repository.go       # Database read/write operations
├── repository_test.go  # Unit tests with mock DB
└── config.go           # Config struct (what NewService expects)
```

### The `Service` Struct Pattern

Every domain package has a central `Service` struct that owns all dependencies. This is the primary unit of dependency injection:

```go
// internal/ingestion/service.go
package ingestion

import (
    "github.com/aetherflow/aetherflow/pkg/messaging"
    authv1 "github.com/aetherflow/proto/gen/go/aetherflow/auth/v1"
    "go.uber.org/zap"
)

// Service is the ingestion domain service.
// It holds all dependencies and implements the gRPC IngestionServiceServer interface.
// It is safe for concurrent use.
type Service struct {
    // Generated gRPC interface (ensures compile-time compliance)
    ingestionv1.UnimplementedIngestionServiceServer

    // Dependencies — all interfaces for testability
    repo        Repository         // PostgreSQL operations
    publisher   messaging.Publisher // Kafka producer
    validator   SchemaValidator    // Schema Registry client
    authClient  authv1.AuthServiceClient
    rateLimiter RateLimiter

    logger *zap.Logger
    cfg    Config
}

// NewService constructs a Service with all dependencies initialized.
// Returns an error if any dependency fails to connect.
func NewService(ctx context.Context, cfg Config) (*Service, error) {
    // ... dependency initialization
}
```

**Why this pattern?**
- `Service` owns concrete implementations in production
- Tests substitute `Repository`, `Publisher`, etc. with mock implementations
- No global state means multiple `Service` instances can coexist (useful in tests)
- `UnimplementedIngestionServiceServer` embedding provides forward compatibility: new RPC methods added to the proto won't break compilation until explicitly implemented

### The `Repository` Interface Pattern

Database access is always abstracted behind an interface. This is not premature abstraction — it is what allows handler unit tests to run without a real database:

```go
// internal/ingestion/repository.go

// Repository defines the storage interface for the ingestion domain.
// The concrete implementation (postgresRepository) lives in the same file.
// A mock implementation (MockRepository) is in pkg/testutil/mocks.go.
type Repository interface {
    InsertEvent(ctx context.Context, event *Event) error
    GetEventByID(ctx context.Context, tenantID, eventID string) (*Event, error)
    ListEventsByTenant(ctx context.Context, tenantID string, opts ListOptions) ([]*Event, error)
}

// postgresRepository is the production PostgreSQL implementation.
// It is unexported — callers only ever see the Repository interface.
type postgresRepository struct {
    db *pgxpool.Pool
}

func newPostgresRepository(ctx context.Context, databaseURL string) (Repository, error) {
    pool, err := pgxpool.New(ctx, databaseURL)
    if err != nil {
        return nil, fmt.Errorf("connect to postgres: %w", err)
    }
    return &postgresRepository{db: pool}, nil
}
```

---

## 4. The `pkg/` Directory — Shared Libraries

`pkg/` contains code that is shared across multiple services. Unlike `internal/`, packages in `pkg/` are importable by any service in the monorepo. Think of it as AetherFlow's internal standard library.

### `pkg/messaging` — Kafka Client

The canonical Kafka producer and consumer implementations. Any service that produces to or consumes from Kafka imports from here. **Do not use `github.com/IBM/sarama` directly in service code** — always go through `pkg/messaging`, which encodes our standard configuration (see `adr-007-messaging-kafka.md`).

```go
// Usage in a service
import "github.com/aetherflow/aetherflow/pkg/messaging"

producer, err := messaging.NewProducer(messaging.ProducerConfig{
    Brokers:        cfg.KafkaBrokers,
    SchemaRegistry: cfg.SchemaRegistryURL,
})

// Publish an event (Avro-serialized, schema-validated)
err = producer.Publish(ctx, messaging.Message{
    Topic:   "aetherflow.ingestion.raw-event.v1",
    Key:     []byte(event.TenantID),    // partition key
    Payload: event,                      // struct — serialized to Avro automatically
})
```

### `pkg/grpcclient` — gRPC Client Factory

Standardized gRPC dial logic with mTLS, keepalive, and retry policy pre-configured (see `adr-009-api-grpc.md`). All service-to-service gRPC calls go through this:

```go
import "github.com/aetherflow/aetherflow/pkg/grpcclient"
import authv1 "github.com/aetherflow/proto/gen/go/aetherflow/auth/v1"

conn, err := grpcclient.NewInternalClient("auth-service-grpc.internal.aetherflow.com:50051")
if err != nil {
    return nil, fmt.Errorf("dial auth-service: %w", err)
}
authClient := authv1.NewAuthServiceClient(conn)
```

### `pkg/logging` — Structured Logger

A thin wrapper around `go.uber.org/zap` that adds OTel trace context injection (so every log line carries `trace_id` and `span_id`), standardizes field names, and handles log level parsing from the `LOG_LEVEL` environment variable.

```go
import "github.com/aetherflow/aetherflow/pkg/logging"

logger := logging.New("info")  // or os.Getenv("LOG_LEVEL")

// Context-aware logging — automatically injects trace_id from the span in ctx
logger.InfoContext(ctx, "event processed",
    zap.String("tenant_id", event.TenantID),
    zap.String("event_type", event.EventType),
    zap.Int("payload_bytes", len(event.Payload)),
)

// Never use fmt.Println or the standard library log package in service code.
// They bypass OTel context injection and produce unstructured output.
```

### `pkg/errors` — Error Types

Canonical error types that map cleanly to gRPC status codes (per `adr-009-api-grpc.md`). Use these instead of `fmt.Errorf` for errors that cross service boundaries:

```go
import "github.com/aetherflow/aetherflow/pkg/errors"

// Domain error types
return errors.NotFound("tenant %q not found", tenantID)
    // → gRPC status: NOT_FOUND

return errors.InvalidArgument("event payload exceeds maximum size of %d bytes", maxSize)
    // → gRPC status: INVALID_ARGUMENT

return errors.Internal("failed to publish to kafka: %w", err)
    // → gRPC status: INTERNAL

// Error wrapping that preserves gRPC status across service boundaries
if err := authClient.ValidateToken(ctx, req); err != nil {
    return errors.Wrap(err, "token validation failed")
    // Preserves the UNAUTHENTICATED status from auth-service
}
```

### `pkg/testutil` — Test Helpers

Commonly needed test infrastructure to avoid boilerplate in every test file:

```go
import "github.com/aetherflow/aetherflow/pkg/testutil"

// Build a test PostgreSQL database (uses testcontainers-go — real Postgres in Docker)
db := testutil.NewTestDB(t)  // auto-cleaned up via t.Cleanup()

// Mock implementations of shared interfaces
mockPublisher := testutil.NewMockPublisher(t)
mockPublisher.ExpectPublish("aetherflow.ingestion.raw-event.v1", anyMessage)

// Build fixture objects with sensible defaults (override only what matters)
event := testutil.EventBuilder().
    WithTenantID("test-tenant").
    WithEventType("user.click").
    Build()
```

---

## 5. The `db/` Directory — Schema and Migrations

```
db/
├── migrations/
│   ├── 0001_init.sql                    # Initial schema setup: extensions, functions, triggers
│   ├── 0002_create_tenants.sql
│   ├── 0003_create_ingestion_events.sql
│   ├── 0004_create_metrics_hypertable.sql  # TimescaleDB hypertable creation
│   ├── 0005_add_events_tenant_index.sql
│   └── ...  (currently 47 migrations)
└── seeds/
    ├── dev-tenant.sql                   # The dev-tenant-001 fixture (see onboarding-env-setup.md)
    └── sample-events.sql               # 7 days of synthetic events for local development
```

Migration files are named `NNNN_description.sql` where `NNNN` is a zero-padded sequential integer. `golang-migrate` applies them in order and tracks the last-applied migration in a `schema_migrations` table.

**Rule:** Never edit an existing migration file. Once a migration is applied to any environment, it is immutable. If you need to undo something, write a new migration.

The current schema diagram lives in Confluence: `Engineering → Database → Schema Diagram`. It is auto-generated from the production schema weekly.

---

## 6. The `tests/` Directory — Integration and E2E

```
tests/
├── integration/
│   ├── ingestion_test.go        # Tests against a real Postgres + Redis (testcontainers)
│   ├── auth_test.go
│   └── billing_test.go
└── e2e/
    ├── ingestion_e2e_test.go    # Full flow tests against staging environment
    ├── query_e2e_test.go
    └── helpers/
        └── staging_client.go   # Authenticated client pointed at staging endpoints
```

The integration tests in `tests/integration/` use `testcontainers-go` to spin up real PostgreSQL and Redis instances. They are tagged with `//go:build integration` and run in CI with the `-tags=integration` flag. They do NOT run against live Kafka — Kafka interactions in integration tests are faked via `pkg/testutil.MockPublisher`.

The E2E tests in `tests/e2e/` run against the live staging environment after every staging deployment. They require `STAGING_API_KEY` and `STAGING_BASE_URL` environment variables set to a real staging tenant. These are injected by GitHub Actions from organization secrets.

---

## 7. Key Patterns You'll See Everywhere

These four patterns appear so frequently across the codebase that you should recognize them on sight.

### Pattern 1: Context Propagation

Every function that does I/O takes `context.Context` as its first argument. This is Go convention, and at AetherFlow it carries:
- Cancellation signals (request timeout, shutdown)
- OTel trace spans (for distributed tracing)
- Request-scoped values (tenant ID, request ID — injected by middleware)

```go
// Correct: context threads through the full call chain
func (s *Service) HandleIngestEvent(ctx context.Context, req *ingestionv1.IngestEventRequest) (*ingestionv1.IngestEventResponse, error) {
    // ctx carries the trace span from the gRPC server interceptor
    event, err := s.validator.Validate(ctx, req)   // Validate uses ctx for tracing
    if err != nil { return nil, err }

    if err := s.repo.InsertEvent(ctx, event); err != nil { // Insert uses ctx for cancellation
        return nil, errors.Internal("insert event: %w", err)
    }
    // ...
}

// Wrong: context dropped, trace broken, cancellation ignored
func (s *Service) HandleIngestEvent(ctx context.Context, req *ingestionv1.IngestEventRequest) (...) {
    s.validator.Validate(context.Background(), req)  // ❌ new context loses trace
}
```

### Pattern 2: Error Wrapping with `%w`

Use `fmt.Errorf("description: %w", err)` to wrap errors with context while preserving `errors.Is` and `errors.As` compatibility. At service boundaries, wrap with `pkg/errors` types to ensure the correct gRPC status code propagates:

```go
// Wrapping at the repository layer (adds context, preserves unwrapping)
func (r *postgresRepository) InsertEvent(ctx context.Context, event *Event) error {
    _, err := r.db.Exec(ctx, insertEventSQL, event.ID, event.TenantID, ...)
    if err != nil {
        // Check for specific Postgres error codes
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23505" { // unique_violation
            return errors.AlreadyExists("event %q already exists", event.ID)
        }
        return fmt.Errorf("insert event: %w", err)  // generic wrap — becomes INTERNAL at gRPC layer
    }
    return nil
}
```

### Pattern 3: Table-Driven Tests

Unit tests use Go's table-driven test pattern. You'll see this everywhere:

```go
// internal/ingestion/validator_test.go
func TestValidateEvent(t *testing.T) {
    tests := []struct {
        name    string
        input   *ingestionv1.IngestEventRequest
        wantErr bool
        errCode codes.Code
    }{
        {
            name:  "valid event",
            input: &ingestionv1.IngestEventRequest{EventType: "user.click", Payload: []byte(`{}`)},
        },
        {
            name:    "missing event type",
            input:   &ingestionv1.IngestEventRequest{Payload: []byte(`{}`)},
            wantErr: true,
            errCode: codes.InvalidArgument,
        },
        {
            name:    "payload too large",
            input:   &ingestionv1.IngestEventRequest{
                EventType: "user.click",
                Payload:   make([]byte, 2*1024*1024), // 2MB > 1MB limit
            },
            wantErr: true,
            errCode: codes.InvalidArgument,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            v := newValidator(testSchemaRegistryURL)
            _, err := v.Validate(context.Background(), tt.input)
            if tt.wantErr {
                require.Error(t, err)
                assert.Equal(t, tt.errCode, status.Code(err))
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

### Pattern 4: Graceful Shutdown

Every server listens for `context.Done()` and shuts down cleanly. The pattern in `main.go` (signal-notified context) cascades cancellation through the entire service:

```go
func runGRPCServer(ctx context.Context, srv *grpc.Server, addr string) error {
    lis, err := net.Listen("tcp", addr)
    if err != nil {
        return fmt.Errorf("listen on %s: %w", addr, err)
    }

    // Shut down when context is cancelled (SIGTERM received)
    go func() {
        <-ctx.Done()
        // GracefulStop allows in-flight RPCs to complete before closing
        srv.GracefulStop()
    }()

    return srv.Serve(lis)
}
```

---

## 8. Data Flow: Event Ingestion End-to-End

The best way to understand the codebase is to trace a single customer event through the entire system. Follow along with the code open.

```
Customer HTTP Request → ingestion-api
        │
        │  1. HTTP handler: cmd/ingestion-api/main.go → HandleIngestEvent
        │  2. Auth middleware: pkg/middleware/auth.go → validates bearer token via auth-service gRPC
        │  3. Rate limit middleware: pkg/middleware/ratelimit.go → checks Redis per-tenant quota
        │
        ▼
internal/ingestion/handler.go: HandleIngestEvent()
        │
        │  4. Parse and validate request (proto → domain type)
        │  5. Call internal/ingestion/validator.go: ValidateSchema()
        │     └── gRPC call to schema-enforcer-grpc:50056 (schema-enforcer/internal/schema)
        │  6. Generate event ID (UUID v7 for time-sortability)
        │
        ▼
internal/ingestion/repository.go: InsertEvent()
        │
        │  7. Write event metadata to PostgreSQL (via PgBouncer)
        │     INSERT INTO ingestion_events (id, tenant_id, event_type, payload, ...)
        │
        ▼
internal/ingestion/publisher.go: PublishEvent()
        │
        │  8. Serialize event to Avro (via Schema Registry)
        │  9. Publish to Kafka topic: aetherflow.ingestion.raw-event.v1
        │     Key = tenant_id (ensures ordering within tenant)
        │
        ▼ (async from here — HTTP response returned to customer at step 9)
        │
Kafka topic: aetherflow.ingestion.raw-event.v1
        │
        ▼
stream-processor: cmd/stream-processor/main.go
        │
        │  10. Consume from raw-event.v1 (consumer group: stream-processor)
        │  11. internal/stream/enricher.go: EnrichEvent()
        │      └── gRPC call to tenant-service: GetTenantConfig (for enrichment rules)
        │  12. internal/stream/transformer.go: TransformEvent()
        │      (applies tenant-specific field mappings and normalizations)
        │
        ▼
        ├── Publish to aetherflow.ingestion.enriched-event.v1
        │       └──▶ metrics-aggregator (writes to TimescaleDB)
        │       └──▶ billing-service (usage metering)
        │
        └── Write enriched event metadata to PostgreSQL
                └── query-api reads from this table to serve customer queries
```

This flow is why auth-service and schema-enforcer are in the synchronous hot path — any latency they add is visible to the customer. Everything after the Kafka publish is asynchronous from the customer's perspective.

---

## 9. How to Find Things

Quick cheat-sheet for common "where is X?" questions:

| I'm looking for... | Look here |
|-------------------|-----------|
| How a gRPC method is implemented | `internal/<domain>/handler.go` |
| The database schema for table X | `db/migrations/` — search for `CREATE TABLE X` |
| Where a Kafka message is published | `internal/<domain>/publisher.go` or grep `producer.Publish` |
| Where a Kafka message is consumed | `cmd/<service>/main.go` or `internal/<domain>/consumer.go` |
| The Protobuf definition for a service | `proto/aetherflow/<domain>/v1/*.proto` |
| Configuration a service reads | `cmd/<service>/main.go` — look for `os.Getenv` calls |
| Shared middleware (auth, logging, rate limit) | `pkg/middleware/` |
| Kafka client configuration | `pkg/messaging/kafka_producer.go`, `pkg/messaging/kafka_consumer.go` |
| gRPC client setup | `pkg/grpcclient/client.go` |
| Error types and gRPC status mapping | `pkg/errors/errors.go` |
| How a service talks to another service | grep for the service name in `pkg/grpcclient/` usages |
| Test fixtures and mock builders | `pkg/testutil/` |
| CI pipeline steps | `.github/workflows/ci.yml` |

### Useful `grep` / `rg` Commands

```bash
# Find all places a specific Kafka topic is produced to
rg "aetherflow.ingestion.raw-event.v1" --type go

# Find all gRPC client usages of a specific service
rg "auth-service-grpc" --type go

# Find all places a specific database table is queried
rg "FROM ingestion_events" --type sql --type go

# Find all usages of a specific pkg package
rg '"github.com/aetherflow/aetherflow/pkg/messaging"' --type go

# Find all error wrapping patterns for a specific error type
rg 'errors.NotFound' --type go
```

---

## 10. Tribal Knowledge & Gotchas

- **The `internal/` boundary is enforced by the Go compiler, not convention.** If you try to import `github.com/aetherflow/aetherflow/internal/ingestion` from a package outside the `aetherflow` module, you'll get a compilation error: `use of internal package ... not allowed`. This is intentional. If you find yourself wanting to share something from `internal/`, it probably belongs in `pkg/`.

- **`context.Background()` in non-test code is almost always wrong.** If you see it in a handler, middleware, or repository method, it's a bug — the existing span and cancellation are being discarded. The only legitimate uses are in `main()` (root context) and in test setup code. Reviewers will flag this.

- **The `proto/gen/go/` directory is committed but stale between CI runs.** The generated stubs in `proto/gen/go/` are re-generated by `buf generate` in CI but are committed to the submodule for local development convenience. If your local stubs differ from what CI generates (e.g., after pulling a proto submodule update without regenerating), you'll get confusing compilation errors. Run `cd proto && buf generate` to fix.

- **`go.work` is your friend and your enemy.** The workspace file makes the `proto/gen/go/` module importable without a `replace` directive. But `go mod tidy` inside a service subdirectory doesn't work correctly in workspace mode — always run `go work sync` from the repo root instead.

- **The `billing-service` has a distributed saga pattern.** If you look at `internal/billing/provisioning.go`, you'll see a more complex pattern than the other services — it orchestrates multiple external calls (Stripe, tenant-service, notification-service) with compensating transactions on failure. Don't be alarmed; read the inline comments which explain each compensation step. This is the only place in the codebase where a saga is used.

- **`stream-processor` has no gRPC server.** It's a pure Kafka consumer — `cmd/stream-processor/main.go` starts a Kafka consumer loop, not a gRPC server. It exposes only a health HTTP endpoint on `:8084`. When you look for handler files in `internal/stream/`, you'll find `consumer.go` instead of `handler.go`. This is the only service with this structure.

- **Feature flags are not in this repo.** LaunchDarkly flag definitions (what flags exist, their default values, their targeting rules) are managed in the LaunchDarkly dashboard, not in code. What you'll see in the codebase is flag evaluation calls: `ldClient.BoolVariation("flag-name", context, false)`. If you need to create a new flag, do it in the LaunchDarkly dashboard first, then add the evaluation call in code.

---

*Document Owner: Engineering Enablement Team | Review Cycle: Quarterly | Next Review: 2023-11-14*
*Feedback: `#eng-onboarding` or open a PR — accuracy feedback from new joiners is especially valuable within 30 days of starting*
*See also: `onboarding-env-setup.md`, `onboarding-first-30-days.md`, `reference-service-catalog.md`*
