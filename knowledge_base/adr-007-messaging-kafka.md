---
title: "ADR-007: Migrate Messaging Layer to Apache Kafka"
author: "Platform Team"
tags: ["adr", "messaging", "kafka", "architecture", "streaming", "migration", "current"]
last_updated: "2024-01-08"
version: "2.0.0"
---

# ADR-007: Migrate Messaging Layer to Apache Kafka

> ✅ **CURRENT DECISION** — This ADR supersedes `adr-001-messaging-rabbitmq.md` (v1.0.0, 2022-03-14).
> Apache Kafka is the **mandated** messaging standard for all new AetherFlow services as of 2024-01-08.
> RabbitMQ is in active decommission. See the migration timeline in Section 7.

---

## Status

**Accepted**

| Field | Value |
|-------|-------|
| Date Proposed | 2023-11-15 |
| Date Accepted | 2024-01-08 |
| Proposers | @priya.nair, @daniel.osei, @sre-team |
| Stakeholders | Platform, Backend, SRE, Data Engineering |
| Supersedes | ADR-001 (`adr-001-messaging-rabbitmq.md`) |

---

## Context

AetherFlow's data volume has grown substantially since ADR-001 was written in March 2022. We now process a sustained **85,000 events/second** with peaks reaching **140,000 events/second** during customer batch-upload windows. Three conditions that ADR-001 explicitly flagged as triggers for revisiting the RabbitMQ decision have all been met:

1. ✅ **Peak throughput exceeded 50k events/s** — reached in Q3 2023
2. ✅ **Message replay required** — the new Stream Replay feature (GA in Q4 2023) requires consumers to reprocess historical event windows
3. ✅ **Team Kafka expertise acquired** — two Platform engineers completed CKA certification; we hired @daniel.osei with 4 years of Kafka operations experience

### Pain Points with RabbitMQ at Scale

- **No replay capability:** The Stream Replay feature required a separate, parallel Kafka cluster we provisioned as a workaround in Q3 2023 — we are already running both systems
- **Memory pressure:** At high queue depths (>500k messages), RabbitMQ nodes routinely hit memory alarms and apply flow control, causing cascading producer backpressure
- **Operational complexity of HA mirroring:** `ha-mode: all` with 3 nodes means every message is written 3 times; at 85k events/s this becomes a real I/O bottleneck
- **Limited consumer group semantics:** Building competing-consumer patterns in RabbitMQ requires manual queue-per-consumer setups that are fragile

### Why Kafka

Apache Kafka's log-based architecture directly addresses each of these pain points:

| Requirement | RabbitMQ | Kafka |
|-------------|----------|-------|
| Message replay | ❌ Not supported | ✅ Consumer offset management |
| >100k events/s throughput | ⚠️ With tuning | ✅ Native |
| Consumer group coordination | ⚠️ Manual | ✅ Built-in |
| Exactly-once semantics | ⚠️ Limited | ✅ With transactions |
| Schema evolution tooling | ❌ DIY | ✅ Confluent Schema Registry |

---

## Decision

**AetherFlow will migrate all asynchronous inter-service messaging from RabbitMQ to Apache Kafka. Kafka is the mandated standard for all new async messaging as of 2024-01-08.**

### Cluster Configuration

We operate a **3-broker Kafka cluster** deployed on dedicated `m6i.2xlarge` EC2 instances in our production VPC, managed via the Strimzi Kafka Operator on Kubernetes.

```yaml
# k8s/kafka/cluster.yaml — Strimzi KafkaCluster resource
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: aetherflow-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      # Retention: 7 days default, overridden per-topic for replay topics
      log.retention.hours: 168
      log.segment.bytes: 1073741824
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 2Ti
          class: gp3-encrypted
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
      class: gp3-encrypted
```

### Topic Naming Convention

```
aetherflow.<domain>.<event-type>.<version>
```

Examples:
- `aetherflow.ingestion.raw-event.v1`
- `aetherflow.analytics.aggregated-metric.v2`
- `aetherflow.billing.usage-record.v1`

Compacted topics (for state stores) use the suffix `.changelog`:
- `aetherflow.tenants.config.changelog`

### Standard Producer Configuration (Go)

```go
// go/pkg/messaging/kafka_producer.go
package messaging

import (
    "github.com/IBM/sarama"
    "go.uber.org/zap"
)

// KafkaProducerConfig returns the standard AetherFlow Kafka producer config.
// All services MUST use this config; do not override without Platform Team approval.
func KafkaProducerConfig() *sarama.Config {
    cfg := sarama.NewConfig()

    // Reliability settings
    cfg.Producer.RequiredAcks = sarama.WaitForAll  // acks=all for durability
    cfg.Producer.Retry.Max = 5
    cfg.Producer.Idempotent = true  // Exactly-once producer semantics

    // Performance settings
    cfg.Producer.Compression = sarama.CompressionSnappy
    cfg.Producer.Flush.Frequency = 10 * time.Millisecond
    cfg.Producer.Flush.MaxMessages = 1000

    // Schema Registry integration (Confluent-compatible)
    cfg.Version = sarama.V3_6_0_0

    return cfg
}

// BrokerAddresses returns the current production broker list.
// These are injected via environment variable KAFKA_BROKERS in production.
func BrokerAddresses() []string {
    brokers := os.Getenv("KAFKA_BROKERS")
    if brokers == "" {
        // Fallback for local development only
        return []string{
            "kafka-broker-1.aetherflow.internal:9092",
            "kafka-broker-2.aetherflow.internal:9092",
            "kafka-broker-3.aetherflow.internal:9092",
        }
    }
    return strings.Split(brokers, ",")
}
```

### Standard Consumer Configuration (Go)

```go
// go/pkg/messaging/kafka_consumer.go
package messaging

// ConsumerGroupConfig returns the standard AetherFlow consumer group config.
func ConsumerGroupConfig(groupID string) *sarama.Config {
    cfg := sarama.NewConfig()
    cfg.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{
        sarama.NewBalanceStrategyRoundRobin(),
    }
    cfg.Consumer.Offsets.Initial = sarama.OffsetNewest
    cfg.Consumer.Offsets.AutoCommit.Enable = false  // Manual commits only
    cfg.Consumer.Offsets.AutoCommit.Interval = 1 * time.Second

    // Heartbeat and session timeouts
    cfg.Consumer.Group.Heartbeat.Interval = 3 * time.Second
    cfg.Consumer.Group.Session.Timeout = 30 * time.Second
    cfg.Consumer.Group.Rebalance.Timeout = 60 * time.Second

    return cfg
}
```

### Schema Registry

All Kafka messages at AetherFlow use **Avro schemas** registered in the **Confluent Schema Registry** (`https://schema-registry.aetherflow.internal`). Schema evolution rules:

- **Backward compatibility** is the default (new consumers can read old messages)
- Breaking schema changes require a new topic version (e.g., `.v2`)
- Schema registration is automated in CI — see `onboarding-cicd-overview.md`

---

## Consequences

### Positive

- Message replay fully unlocks the Stream Replay product feature
- Kafka's partition model provides horizontal scalability without operational tricks
- Confluent Schema Registry enforces contract between producers and consumers
- Consumer offset management gives us exactly-once processing semantics
- We consolidate the Kafka cluster we already provisioned for Stream Replay — net reduction in infrastructure surface

### Negative

- Higher operational complexity than RabbitMQ — requires Kafka-specific operational knowledge
- Consumer rebalances during deployments can cause brief processing pauses (see `runbook-kafka-operations.md` for mitigation)
- Schema Registry adds a network hop on every message serialize/deserialize; must be co-located with brokers

### Neutral

- RabbitMQ decommission will take ~2 quarters; during migration, some services will produce to both systems

---

## Migration Timeline

| Quarter | Milestone |
|---------|-----------|
| **Q4 2023** *(done)* | Kafka cluster provisioned; Stream Replay topics migrated |
| **Q1 2024** *(in progress)* | ingestion-api, stream-processor migrated to Kafka |
| **Q2 2024** | analytics-svc, billing-svc migrated |
| **Q3 2024** | RabbitMQ cluster decommissioned; ADR-001 archived |

Migration status is tracked in Jira Epic: `PLAT-3100 — Kafka Migration`.

---

## Runbook Reference

For day-to-day Kafka operations including consumer lag management, partition rebalancing, and disaster recovery, see `runbook-kafka-operations.md`.

---

*Document Owner: Platform Team | Review Cycle: As-needed post-migration*
*Supersedes: `adr-001-messaging-rabbitmq.md`*
*Related: `runbook-kafka-operations.md`, `adr-010-observability-otel.md`*
