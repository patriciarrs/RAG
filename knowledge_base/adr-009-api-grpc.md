---
title: "ADR-009: Migrate Internal APIs from REST to gRPC"
author: "Backend Team"
tags: ["adr", "grpc", "protobuf", "api", "architecture", "internal-services", "performance", "current"]
last_updated: "2023-10-05"
version: "1.0.0"
---

# ADR-009: Migrate Internal APIs from REST to gRPC

> ✅ **CURRENT DECISION** — This ADR supersedes `adr-003-api-rest.md` (v1.0.1, 2022-07-20) for **internal service-to-service communication**.
> All new internal services MUST expose gRPC endpoints. REST/JSON internal endpoints are in active deprecation.
> **External-facing (customer-facing) APIs remain REST/JSON** — this decision covers internal communication only.

---

## Status

**Accepted**

| Field | Value |
|-------|-------|
| Date Proposed | 2023-09-01 |
| Date Accepted | 2023-10-05 |
| Proposers | @sara.lindqvist, @daniel.osei, @elena.vasquez |
| Stakeholders | Backend, Platform, SRE, Data Engineering |
| Supersedes | ADR-003 (`adr-003-api-rest.md`) — internal APIs only |

---

## Context

ADR-003 established REST/JSON as the internal API standard in July 2022. At the time, this was the right call: the team had no gRPC experience and our call volume was modest. ADR-003 explicitly listed the conditions under which this decision should be revisited.

By Q3 2023, **two of those conditions had been met**:

### Condition 1: Performance at Scale

The launch of our real-time query feature in Q1 2023 dramatically increased internal service call volume. Key observations from our APM traces:

- `query-api` → `stream-processor`: **180,000 internal RPC calls/minute** at peak
- P99 latency for `auth-service` token validation: **42ms** (budget: 20ms)
- JSON serialization/deserialization accounted for **~18% of CPU time** in `query-api` profiling

Switching to gRPC with Protobuf binary encoding is projected to reduce serialization overhead by ~70% based on benchmarks run against our actual message shapes (see `benchmarks/grpc-vs-rest-2023-08.md`).

### Condition 2: Team Expertise

- @daniel.osei joined in Q1 2023 with 4 years of gRPC/Protobuf production experience
- @sara.lindqvist and @elena.vasquez completed internal gRPC training in Q2 2023
- The Backend Team ran a 6-week pilot migrating `auth-service` to gRPC; the result was positive

### Additional Driver: Bidirectional Streaming

The planned Stream Subscription feature (roadmap Q1 2024) requires server-push streaming semantics that are awkward with REST (SSE is limited; WebSockets don't compose well with our service mesh). gRPC bidirectional streaming is a natural fit.

---

## Decision

**AetherFlow will migrate all internal service-to-service APIs from REST/JSON to gRPC with Protocol Buffers (proto3). All new internal services must expose gRPC interfaces. REST internal endpoints must be deprecated on a service-by-service migration schedule.**

### Protobuf Repository Structure

All `.proto` files live in `aetherflow/proto` (a dedicated repository, not embedded in service repos):

```
proto/
├── aetherflow/
│   ├── auth/
│   │   └── v1/
│   │       └── auth.proto
│   ├── ingestion/
│   │   └── v1/
│   │       └── ingestion.proto
│   ├── streaming/
│   │   └── v1/
│   │       └── streaming.proto
│   └── common/
│       └── v1/
│           ├── errors.proto
│           └── pagination.proto
└── buf.yaml
```

### Toolchain

We use [Buf](https://buf.build) for Protobuf linting, breaking-change detection, and code generation. **Do not use raw `protoc`** — it bypasses our linting and breaking-change checks.

```bash
# Generate Go and Python stubs from .proto files
# Run from the root of the aetherflow/proto repository
buf generate

# Check for breaking changes against the BSR (Buf Schema Registry)
buf breaking --against "https://buf.build/aetherflow/proto"

# Lint proto files
buf lint
```

### Example Service Definition

```protobuf
// proto/aetherflow/auth/v1/auth.proto
syntax = "proto3";

package aetherflow.auth.v1;

option go_package = "github.com/aetherflow/proto/gen/go/aetherflow/auth/v1;authv1";

import "aetherflow/common/v1/errors.proto";

service AuthService {
  // ValidateToken validates a bearer token and returns the associated claims.
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);

  // StreamPermissionUpdates streams real-time permission changes for a tenant.
  rpc StreamPermissionUpdates(PermissionUpdateRequest)
      returns (stream PermissionUpdateEvent);
}

message ValidateTokenRequest {
  string token = 1;
  string service_name = 2;  // The calling service, for audit logging
}

message ValidateTokenResponse {
  string tenant_id = 1;
  string user_id = 2;
  repeated string scopes = 3;
  int64 expires_at_unix = 4;
}

message PermissionUpdateRequest {
  string tenant_id = 1;
}

message PermissionUpdateEvent {
  string user_id = 1;
  string resource_type = 2;
  string action = 3;  // "grant" | "revoke"
  int64 timestamp_unix = 4;
}
```

### Standard gRPC Client (Go)

```go
// go/pkg/grpcclient/client.go
package grpcclient

import (
    "context"
    "crypto/tls"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "google.golang.org/grpc/keepalive"
    "time"
)

// NewInternalClient creates a gRPC client connection to an internal AetherFlow service.
// This replaces the deprecated httpclient.NewInternalClient from ADR-003.
func NewInternalClient(serviceAddress string) (*grpc.ClientConn, error) {
    tlsConfig := &tls.Config{
        // mTLS certificates from Vault — injected via init container
        // See policy-secret-management.md for cert rotation details
    }

    return grpc.Dial(
        serviceAddress,
        grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)),
        grpc.WithKeepaliveParams(keepalive.ClientParameters{
            Time:                10 * time.Second,
            Timeout:             5 * time.Second,
            PermitWithoutStream: true,
        }),
        grpc.WithDefaultServiceConfig(`{
            "methodConfig": [{
                "name": [{"service": ""}],
                "retryPolicy": {
                    "maxAttempts": 4,
                    "initialBackoff": "0.1s",
                    "maxBackoff": "2s",
                    "backoffMultiplier": 2,
                    "retryableStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"]
                }
            }]
        }`),
    )
}
```

### Service Address Convention

Internal gRPC service addresses follow the pattern:

```
<service-name>-grpc.internal.aetherflow.com:50051
```

Examples:
- `auth-service-grpc.internal.aetherflow.com:50051`
- `tenant-service-grpc.internal.aetherflow.com:50051`
- `billing-service-grpc.internal.aetherflow.com:50051`

The REST endpoints on `.internal.aetherflow.com:443` will remain active during the migration period but will be removed once all callers have migrated.

### Error Handling

Use gRPC status codes. Map application errors to the appropriate canonical codes:

| Situation | gRPC Status Code |
|-----------|-----------------|
| Resource not found | `NOT_FOUND` |
| Invalid input | `INVALID_ARGUMENT` |
| Caller not authorized | `PERMISSION_DENIED` |
| Caller not authenticated | `UNAUTHENTICATED` |
| Rate limit exceeded | `RESOURCE_EXHAUSTED` |
| Transient internal error | `UNAVAILABLE` (retriable) |
| Non-transient internal error | `INTERNAL` (not retriable) |

Do NOT use `UNKNOWN` — it is never acceptable to return an undifferentiated error. If nothing else fits, use `INTERNAL`.

---

## Consequences

### Positive

- **~70% serialization overhead reduction** (benchmarked on our message shapes)
- **Compile-time contract enforcement** — generated stubs mean schema drift causes a build failure, not a runtime 400
- **Bidirectional streaming** unlocks the Stream Subscription feature
- **Breaking-change detection** via `buf breaking` in CI prevents accidental API regressions
- HTTP/2 multiplexing eliminates connection pool management problems we had under REST

### Negative

- Loss of curl-debuggability for internal APIs. Engineers must use `grpcurl` or `evans` instead:
  ```bash
  grpcurl -plaintext auth-service-grpc.internal.aetherflow.com:50051 list
  grpcurl -plaintext -d '{"token": "eyJ..."}' \
    auth-service-grpc.internal.aetherflow.com:50051 \
    aetherflow.auth.v1.AuthService/ValidateToken
  ```
- Protobuf toolchain (`buf`, generated stubs) adds build complexity; addressed by the `proto` Makefile targets
- Migration is a multi-quarter effort; both systems will coexist, increasing cognitive overhead temporarily

### Neutral

- REST/JSON remains the standard for **external customer-facing APIs**. OpenAPI specs for external APIs are still required and enforced via CI.

---

## Migration Schedule

| Service | REST Deprecation Target | gRPC GA Target | Owner |
|---------|------------------------|---------------|-------|
| `auth-service` | ✅ Done (Q3 2023) | ✅ Done (Q3 2023) | @sara.lindqvist |
| `tenant-service` | Q4 2023 | Q4 2023 | @james.whitfield |
| `ingestion-api` | Q1 2024 | Q1 2024 | @priya.nair |
| `query-api` | Q1 2024 | Q1 2024 | @elena.vasquez |
| `billing-service` | Q2 2024 | Q2 2024 | @marcus.chen |
| `stream-processor` | Q2 2024 | Q2 2024 | @daniel.osei |

Migration status is tracked in Jira Epic: `BACK-1450 — gRPC Migration`.

---

## Developer Setup

For onboarding, proto codegen setup, and local gRPC testing tools, see `onboarding-env-setup.md` (Section 5: "Proto Toolchain Setup").

---

*Document Owner: Backend Team | Review Cycle: As-needed*
*Supersedes: `adr-003-api-rest.md` (internal APIs)*
*Related: `onboarding-env-setup.md`, `policy-secret-management.md`, `reference-service-catalog.md`*
