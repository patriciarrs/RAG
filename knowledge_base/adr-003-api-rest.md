---
title: "ADR-003: Adopt REST/JSON for Internal Service APIs"
author: "Backend Team"
tags: ["adr", "rest", "api", "architecture", "http", "json", "legacy", "internal-services"]
last_updated: "2022-07-20"
version: "1.0.1"
---

# ADR-003: Adopt REST/JSON for Internal Service APIs

> ⚠️ **LEGACY DOCUMENT** — This ADR has been superseded by `adr-009-api-grpc.md` (v1.0.0, 2023-10-05).
> REST/JSON for **internal** service-to-service communication is **no longer the standard**.
> This document is preserved for historical context only.
> **External-facing (customer-facing) APIs remain REST/JSON** — this supersession applies only to internal inter-service communication.

---

## Status

~~**Accepted**~~ → **SUPERSEDED** by ADR-009 (internal APIs only)

| Field | Value |
|-------|-------|
| Date Proposed | 2022-06-30 |
| Date Accepted | 2022-07-20 |
| Date Superseded | 2023-10-05 |
| Proposers | @james.whitfield, @sara.lindqvist |
| Stakeholders | Backend Team, Frontend Team, Platform Team |
| Superseded By | ADR-009 (`adr-009-api-grpc.md`) — internal APIs only |

---

## Context

In mid-2022, AetherFlow had grown from a monolith to approximately 9 distinct services. These services communicated over a mix of direct database access, Redis-based side channels, and informal HTTP calls with no consistent contract. This created several problems:

- No clear service ownership boundaries
- Schema drift between what one service produced and what another consumed
- Debugging cross-service failures required reading multiple services' source code simultaneously
- No standard for authentication between internal services

We needed a standard for internal service-to-service communication that would:

1. Establish clear API contracts
2. Be familiar to the entire engineering team
3. Have excellent tooling in Go and Python (our two primary languages)
4. Support reasonable performance at our then-current scale (~5,000 req/min peak internal traffic)

### Options Evaluated

| Option | Pros | Cons |
|--------|------|------|
| **REST/JSON over HTTP/1.1** | Universal familiarity; excellent tooling; curl-debuggable; OpenAPI spec tooling | Verbose payloads; no native streaming; no code generation; HTTP/1.1 lacks multiplexing |
| **gRPC (Protobuf)** | Strongly typed contracts; code gen; efficient binary encoding; bidirectional streaming | Learning curve; not curl-debuggable; required Protobuf toolchain setup; team had no experience |
| **GraphQL** | Flexible queries; self-documenting | Overkill for internal RPC; N+1 problem risk; no meaningful benefit over REST for backend-to-backend |
| **Thrift (Apache)** | Efficient; code gen | Poor Go ecosystem; essentially legacy at this point |

---

## Decision

**AetherFlow will use REST/JSON over HTTP/1.1 as the standard protocol for all internal service-to-service communication.**

All new internal service endpoints must:

- Follow RESTful resource naming conventions (`/v1/resources/{id}`)
- Use JSON with `Content-Type: application/json`
- Be documented with an OpenAPI 3.0 spec in the service's `api/openapi.yaml`
- Authenticate using the internal mTLS certificate pair provisioned by Vault (see `policy-secret-management.md`)

### URL Convention

```
http://<service-name>.internal.aetherflow.com/v1/<resource>
```

Examples:
- `http://auth-service.internal.aetherflow.com/v1/tokens/validate`
- `http://tenant-service.internal.aetherflow.com/v1/tenants/{tenant_id}`
- `http://billing-service.internal.aetherflow.com/v1/usage-records`

### Standard HTTP Client (Go)

```go
// go/pkg/httpclient/internal.go
// ⚠️ LEGACY: This client is deprecated for internal service communication.
// New services must use the gRPC client. See adr-009-api-grpc.md.
package httpclient

import (
    "net/http"
    "time"
)

// InternalClient is the standard HTTP client for internal service calls.
// Pre-configured with timeouts, mTLS, and retry logic.
func NewInternalClient(serviceName string) *http.Client {
    transport := &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
        TLSClientConfig:     internalMTLSConfig(serviceName), // From Vault-provisioned cert
    }

    return &http.Client{
        Transport: transport,
        Timeout:   10 * time.Second,
    }
}

// Standard response error type
type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    TraceID string `json:"trace_id"`
}
```

### Standard Response Envelope

All internal API responses MUST use this envelope:

```json
{
  "data": { ... },
  "meta": {
    "request_id": "req_01HQ...",
    "api_version": "v1",
    "timestamp": "2022-07-20T14:32:00Z"
  },
  "error": null
}
```

On error:
```json
{
  "data": null,
  "meta": { "request_id": "req_01HQ...", "api_version": "v1" },
  "error": {
    "code": "TENANT_NOT_FOUND",
    "message": "No tenant found with id 'acme-corp'",
    "status": 404
  }
}
```

---

## Consequences

### Positive

- Zero learning curve — every engineer on the team knows REST
- OpenAPI specs generated automatically via `swaggo` annotations
- Easy to inspect and debug with curl, Postman, or browser DevTools
- Excellent Go standard library support; no external dependencies required for the server

### Negative

- JSON verbosity adds overhead at high call frequencies
- No compile-time contract enforcement between caller and server — schema drift is possible if OpenAPI specs lag the implementation
- HTTP/1.1 lacks multiplexing; under high concurrency, connection pool management becomes a concern
- No native support for server-side streaming patterns

### Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| OpenAPI spec drift | CI check via `oapi-codegen` compares spec against handler signatures |
| Missing timeouts causing cascading failures | `NewInternalClient` enforces a hard 10s timeout |
| Authentication bypass | All internal traffic routed through service mesh (Linkerd); mTLS enforced at the mesh layer |

---

## Notes on Supersession

When this ADR was written, the choice of REST/JSON was correct and pragmatic. The AetherFlow backend team had zero gRPC experience, our internal call volume was moderate, and the primary value we needed was consistency and contract documentation — not performance.

By mid-2023, two things changed:

1. Internal service call volume grew 8× due to the launch of the real-time query feature, and REST/JSON overhead was measurable in our P99 latency budgets
2. The team had grown to include engineers with gRPC experience, and the operational overhead of the Protobuf toolchain became acceptable

See `adr-009-api-grpc.md` for the full rationale.

---

*Document Owner: Backend Team | Status: Superseded (internal APIs only)*
*Superseded by: `adr-009-api-grpc.md`*
*Note: External/customer-facing APIs remain REST/JSON. This supersession is for inter-service communication only.*
