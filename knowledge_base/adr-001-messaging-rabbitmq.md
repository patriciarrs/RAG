---
title: "ADR-001: Adopt RabbitMQ for Asynchronous Messaging"
author: "Platform Team"
tags: ["adr", "messaging", "rabbitmq", "architecture", "async", "legacy"]
last_updated: "2022-03-14"
version: "1.0.0"
---

# ADR-001: Adopt RabbitMQ for Asynchronous Messaging

> ⚠️ **LEGACY DOCUMENT** — This ADR has been superseded by `adr-007-messaging-kafka.md` (v2.0.0, 2024-01-08).
> The decision recorded here is **no longer active**. It is preserved for historical context and to understand the migration rationale.
> Do NOT use this document to make new architectural decisions about messaging.

---

## Status

~~**Accepted**~~ → **SUPERSEDED** by ADR-007

| Field | Value |
|-------|-------|
| Date Proposed | 2022-02-28 |
| Date Accepted | 2022-03-14 |
| Date Superseded | 2024-01-08 |
| Proposers | @marcus.chen, @priya.nair |
| Stakeholders | Platform Team, Backend Team |

---

## Context

As of Q1 2022, AetherFlow's data ingestion pipeline processed approximately **8,000 events/second** at peak. Inter-service communication relied on a mix of synchronous HTTP calls and ad-hoc Redis pub/sub channels — a pattern that had grown organically and caused significant coupling between services.

We needed a dedicated, reliable async messaging layer that would:

1. Decouple producers from consumers
2. Provide durable message storage for retry logic
3. Support flexible routing patterns (topic exchanges, fanout)
4. Be operationally manageable with our existing team (3 engineers in Platform)

### Options Evaluated

| Option | Pros | Cons |
|--------|------|------|
| **RabbitMQ** | Mature, well-understood, AMQP standard, excellent Go/Python clients, flexible routing via exchanges | Not designed for log-style replay; limited horizontal scalability |
| **Apache Kafka** | High throughput, log replay, strong ordering guarantees | High operational complexity; felt like over-engineering for 8k events/s; team had no Kafka expertise |
| **AWS SQS/SNS** | Fully managed, no ops burden | Vendor lock-in; limited routing expressiveness; per-message pricing would be costly at scale |
| **NATS** | Extremely fast, lightweight | Immature ecosystem; limited durability guarantees at the time |

---

## Decision

**We will adopt RabbitMQ 3.9 as the standard asynchronous messaging layer for AetherFlow inter-service communication.**

All new async messaging MUST use RabbitMQ. Existing Redis pub/sub usage should be migrated over the following two quarters.

### Topology

```
                    ┌─────────────────────────────────────┐
                    │         RabbitMQ Cluster (3 nodes)   │
                    │                                      │
Producer ──AMQP──▶  │  [aetherflow.events] topic exchange │
                    │           │          │               │
                    │    [ingestion]   [analytics]         │
                    │      queue           queue           │
                    └──────────┼───────────┼───────────────┘
                               │           │
                          Consumer A   Consumer B
                      (stream-processor) (analytics-svc)
```

### Configuration Standards

```go
// go/pkg/messaging/rabbitmq.go
// Standard RabbitMQ connection pattern — AetherFlow internal library
package messaging

import (
    amqp "github.com/rabbitmq/amqp091-go"
    "log"
    "time"
)

const (
    // ⚠️ LEGACY: This connection string pattern is no longer used.
    // See adr-007-messaging-kafka.md for current Kafka configuration.
    DefaultRabbitMQURL = "amqp://aetherflow:%s@rabbitmq.internal.aetherflow.com:5672/"
    ExchangeName       = "aetherflow.events"
    ExchangeType       = "topic"
    ReconnectDelay     = 5 * time.Second
    MaxRetries         = 5
)

type RabbitMQClient struct {
    conn    *amqp.Connection
    channel *amqp.Channel
    url     string
}

func NewRabbitMQClient(url string) (*RabbitMQClient, error) {
    conn, err := amqp.Dial(url)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to RabbitMQ: %w", err)
    }

    ch, err := conn.Channel()
    if err != nil {
        return nil, fmt.Errorf("failed to open channel: %w", err)
    }

    // Declare the canonical topic exchange
    err = ch.ExchangeDeclare(
        ExchangeName,
        ExchangeType,
        true,  // durable
        false, // auto-deleted
        false, // internal
        false, // no-wait
        nil,
    )
    if err != nil {
        return nil, fmt.Errorf("failed to declare exchange: %w", err)
    }

    return &RabbitMQClient{conn: conn, channel: ch, url: url}, nil
}
```

### Queue Naming Convention

```
aetherflow.<domain>.<event-type>
```

Examples:
- `aetherflow.ingestion.raw-event`
- `aetherflow.analytics.aggregated-metric`
- `aetherflow.billing.usage-record`

Dead-letter queues follow the pattern: `<original-queue>.dlq`

---

## Consequences

### Positive

- Clear separation of producers and consumers — services no longer need to know each other's addresses
- Built-in retry and dead-letter queue support for fault tolerance
- The AMQP Go client (`github.com/rabbitmq/amqp091-go`) is excellent and well-maintained
- RabbitMQ Management UI provides visibility into queue depths and consumer counts

### Negative

- RabbitMQ does **not** support message replay — consumed messages are gone. This is acceptable for our current use cases but will become a limitation if we need event sourcing.
- Horizontal scaling requires careful attention to queue mirroring policies. Our initial setup uses `ha-mode: all` mirroring across 3 nodes, which increases write amplification.
- No native support for stream processing semantics (consumer groups with offset tracking).

### Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| RabbitMQ node failure | 3-node cluster with quorum queues enabled for critical queues |
| Queue depth explosion (consumer crashes) | Alertmanager alert fires at >10,000 messages in any queue |
| Schema evolution breaking consumers | Use JSON with optional fields; producers must not remove fields without a deprecation window |

---

## Notes

*This was the right call for AetherFlow in Q1 2022.* As of the time of writing, our throughput is manageable, our team has solid RabbitMQ operational knowledge, and the flexibility of topic exchanges covers our routing needs well.

However, we explicitly acknowledged at the time that this decision should be revisited if:
- Peak throughput exceeds **50,000 events/second**
- We require **message replay** or **event sourcing** capabilities
- The team grows the expertise needed to operate Kafka safely

All three conditions were eventually met. See `adr-007-messaging-kafka.md` for the migration decision.

---

*Document Owner: Platform Team | Status: Superseded*
*Superseded by: `adr-007-messaging-kafka.md`*
*Historical context only — do not implement new RabbitMQ consumers or producers.*
