---
title: "ADR-005: PostgreSQL as Primary Relational Store"
author: "Platform Team"
tags: ["adr", "postgresql", "database", "architecture", "rds", "timescaledb", "current", "storage"]
last_updated: "2023-04-02"
version: "1.3.0"
---

# ADR-005: PostgreSQL as Primary Relational Store

> ✅ **CURRENT DECISION** — Active and uncontested. PostgreSQL 15 is the mandated relational database for all AetherFlow services that require persistent relational storage.
> This ADR has been updated three times since initial acceptance (v1.0.0, 2022-05-10) to reflect schema governance additions (v1.1.0), TimescaleDB adoption for time-series data (v1.2.0), and the PgBouncer connection pooling standard (v1.3.0).

---

## Status

**Accepted — Active**

| Field | Value |
|-------|-------|
| Date Proposed | 2022-04-20 |
| Date Accepted | 2022-05-10 |
| Last Amended | 2023-04-02 (v1.3.0 — PgBouncer standard) |
| Proposers | @marcus.chen, @priya.nair |
| Stakeholders | Platform, Backend, SRE, Data Engineering |
| Supersedes | Nothing (first formal DB decision) |

---

## Context

When AetherFlow was founded, data persistence was handled by a single hosted MongoDB Atlas cluster — a fast choice made in the early startup phase to avoid operational overhead. By Q1 2022, several problems had accumulated:

**Problem 1: Schema chaos.** MongoDB's schemaless model meant different services stored subtly incompatible representations of the same entities. The `tenant` document had 11 different shapes in production, depending on when it was created and which service last wrote it. Querying across these shapes required defensive application code and produced subtle bugs.

**Problem 2: No transactional integrity for billing.** Usage metering and invoice generation required atomic multi-document writes. MongoDB multi-document transactions exist but are significantly more complex to use correctly than SQL transactions and carry performance overhead.

**Problem 3: Query expressiveness.** Several analytical queries needed for internal reporting required multi-collection joins, window functions, and aggregations that were clumsy in MongoDB's aggregation pipeline. The same queries in SQL were 5–10 lines of readable code.

**Problem 4: Operational familiarity.** Our engineering team had strong collective SQL and PostgreSQL expertise. MongoDB expertise was shallow — one engineer. Operational runbooks, community resources, and tooling are significantly more mature in the PostgreSQL ecosystem.

We evaluated three alternatives for a migration target:

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL (RDS)** | Team expertise; rich SQL; ACID transactions; JSONB for semi-structured data; AWS-managed | Requires schema migration from MongoDB; vertical scaling limits |
| **MySQL / Aurora MySQL** | AWS-managed; large ecosystem | Team has PostgreSQL preference; Aurora MySQL replication complexity; weaker JSON support vs. JSONB |
| **CockroachDB** | Horizontal scalability; distributed ACID | High operational complexity; significant deviation from standard Postgres behavior; expensive; premature optimization for our scale |
| **Stay on MongoDB** | No migration cost | All problems above remain; harder to hire for in our market |

---

## Decision

**AetherFlow will use PostgreSQL 15 (managed via AWS RDS) as the primary relational data store for all services requiring structured persistent storage.**

All new data models must be defined as PostgreSQL schemas using `golang-migrate` for migrations. Services must not introduce new MongoDB, MySQL, or other relational database dependencies without a new ADR.

### Version Standard

| Component | Version | Notes |
|-----------|---------|-------|
| PostgreSQL engine | **15.x** (currently 15.5) | Upgrade to new minor versions within 30 days of release |
| RDS instance class | `db.r6g.2xlarge` (primary) | Graviton2; memory-optimized for PostgreSQL's buffer pool |
| PgBouncer | **1.21.x** | Transaction-mode pooling; see Section 5 |
| `golang-migrate` | **4.17.x** | Schema migration tooling |
| TimescaleDB extension | **2.13.x** | Time-series data only; see Section 4 |

---

## Schema Design Standards

All PostgreSQL schemas at AetherFlow must follow these conventions. They are enforced via `sqlfluff` lint in CI (configured in `.sqlfluff`) and reviewed by the Platform Team for any migration touching the `events`, `metrics`, or `tenants` tables.

### Naming Conventions

```sql
-- Tables: plural snake_case nouns
CREATE TABLE tenant_configurations (...);
CREATE TABLE ingestion_events (...);

-- Columns: singular snake_case
tenant_id       UUID NOT NULL
created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
is_deleted      BOOLEAN NOT NULL DEFAULT FALSE

-- Indexes: idx_<table>_<columns>
CREATE INDEX idx_ingestion_events_tenant_id_created_at
  ON ingestion_events (tenant_id, created_at DESC);

-- Foreign keys: fk_<table>_<referenced_table>
ALTER TABLE ingestion_events
  ADD CONSTRAINT fk_ingestion_events_tenants
  FOREIGN KEY (tenant_id) REFERENCES tenants (id);

-- Primary keys: always named 'id', always UUID
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

### Mandatory Columns

Every table (except pure join tables) must include:

```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
-- For soft-delete tables (most tables — see deletion policy):
deleted_at  TIMESTAMPTZ NULL DEFAULT NULL
```

The `updated_at` column must be maintained via a trigger (not application logic — application code forgets):

```sql
-- Reusable trigger function (defined once in db/migrations/0001_init.sql)
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to every table that has updated_at
CREATE TRIGGER set_tenants_updated_at
  BEFORE UPDATE ON tenants
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### JSONB for Semi-Structured Data

PostgreSQL's `JSONB` type is the approved escape hatch for data with variable or evolving structure. Use it when:
- The structure varies significantly per row (e.g., event payloads, webhook configurations)
- The data does not need to be queried by its internal fields in the hot path

```sql
-- Good: event payload stored as JSONB (variable structure per event type)
CREATE TABLE ingestion_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    event_type  TEXT NOT NULL,
    payload     JSONB NOT NULL,           -- variable structure
    metadata    JSONB NOT NULL DEFAULT '{}', -- optional rider fields
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index JSONB fields that appear in WHERE clauses
-- Use GIN for containment queries (@>, ?)
CREATE INDEX idx_ingestion_events_payload_gin
  ON ingestion_events USING GIN (payload);

-- Use expression index for specific key extractions
CREATE INDEX idx_ingestion_events_payload_user_id
  ON ingestion_events ((payload->>'user_id'))
  WHERE payload ? 'user_id';
```

**Do NOT use JSONB for:**
- Fields that will always be present and queried directly (use typed columns)
- Fields used in JOINs (use foreign keys)
- The entirety of a row's data when the structure is known (use typed columns)

---

## Migration Standards

All schema changes use `golang-migrate` with numbered sequential migration files.

```
db/
└── migrations/
    ├── 0001_init.sql
    ├── 0002_add_tenants_table.sql
    ├── 0003_add_ingestion_events_table.sql
    ├── 0004_add_events_tenant_index.sql
    └── ...
```

### Migration Rules (Non-Negotiable)

```sql
-- ✅ SAFE: Adding a nullable column (backward compatible)
ALTER TABLE tenants ADD COLUMN billing_tier TEXT NULL;

-- ✅ SAFE: Adding a column with a default (backward compatible if default is cheap)
ALTER TABLE tenants ADD COLUMN is_enterprise BOOLEAN NOT NULL DEFAULT FALSE;

-- ✅ SAFE: Adding an index CONCURRENTLY (does not lock table)
CREATE INDEX CONCURRENTLY idx_tenants_billing_tier ON tenants (billing_tier);

-- ✅ SAFE: Adding a new table (no impact on existing code)
CREATE TABLE tenant_feature_flags (...);

-- ⚠️  UNSAFE without migration window: Adding NOT NULL column to existing table
-- Must use: ADD COLUMN nullable → backfill → SET NOT NULL in separate migrations
ALTER TABLE tenants ADD COLUMN timezone TEXT NULL;         -- Migration N
UPDATE tenants SET timezone = 'UTC' WHERE timezone IS NULL; -- Migration N+1 (backfill)
ALTER TABLE tenants ALTER COLUMN timezone SET NOT NULL;     -- Migration N+2

-- ❌ NEVER in a single migration on large tables: DROP COLUMN
-- Drop the column in a separate PR at least 1 week after code stops using it.
-- Two-step: code stops reading/writing the column → deploy → then DROP in a new migration.

-- ❌ NEVER: Rename a column in place (breaks existing code reading the old name)
-- Instead: add new column → backfill → migrate code to new column → drop old column
```

### Large Table Migration Protocol

For migrations on tables with > 5 million rows (`events`, `metrics`, `audit_log`), standard `ALTER TABLE` acquires an `ACCESS EXCLUSIVE` lock and blocks all reads and writes. Use these patterns instead:

```sql
-- Pattern: Adding an index on a large table
-- Use CONCURRENTLY — builds the index without locking
-- NOTE: Cannot be run inside a transaction block; golang-migrate wraps in a
-- transaction by default. Use the -- migrate:noTransaction annotation.

-- migrate:noTransaction
CREATE INDEX CONCURRENTLY idx_ingestion_events_event_type
  ON ingestion_events (event_type);

-- Pattern: Adding a NOT NULL constraint to a large table
-- Step 1: Add CHECK constraint as NOT VALID (fast, no scan)
ALTER TABLE ingestion_events
  ADD CONSTRAINT chk_event_type_not_null
  CHECK (event_type IS NOT NULL) NOT VALID;

-- Step 2 (separate migration, can run in off-peak): Validate the constraint
-- This acquires only a SHARE UPDATE EXCLUSIVE lock (allows reads and writes)
ALTER TABLE ingestion_events
  VALIDATE CONSTRAINT chk_event_type_not_null;

-- Step 3 (separate migration, after Step 2): Replace with proper NOT NULL
ALTER TABLE ingestion_events ALTER COLUMN event_type SET NOT NULL;
ALTER TABLE ingestion_events DROP CONSTRAINT chk_event_type_not_null;
```

---

## TimescaleDB Extension (v1.2.0 Amendment)

Added in ADR-005 v1.2.0 (2022-11-15) after benchmarking demonstrated that time-series metric aggregation queries against the `metrics` table (which grows at ~8M rows/day) were consuming 30–40% of primary CPU during peak analytics windows.

### What TimescaleDB Adds

TimescaleDB is a PostgreSQL extension that transparently partitions time-series tables into **hypertable chunks** by time interval. Queries that filter by time range skip irrelevant chunks entirely (chunk exclusion), reducing scan size dramatically.

```sql
-- Convert the metrics table to a hypertable (done in migration 0031)
-- This is a one-time operation; all subsequent INSERTs are transparently partitioned
SELECT create_hypertable(
    'metrics',
    'bucket_time',          -- The time dimension column
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Add compression policy: compress chunks older than 7 days
-- Compressed chunks use ~10x less disk space for time-series data
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Continuous aggregate: pre-computed hourly rollups for the dashboard
-- Refreshes automatically every 30 minutes
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    tenant_id,
    metric_name,
    time_bucket('1 hour', bucket_time) AS hour,
    avg(value)    AS avg_value,
    max(value)    AS max_value,
    min(value)    AS min_value,
    count(*)      AS sample_count
FROM metrics
GROUP BY tenant_id, metric_name, time_bucket('1 hour', bucket_time)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset  => INTERVAL '3 hours',
    end_offset    => INTERVAL '1 minute',
    schedule_interval => INTERVAL '30 minutes'
);
```

**TimescaleDB tables:** `metrics`, `metric_rollups_hourly`, `metric_rollups_daily`
**Standard PostgreSQL tables:** everything else (tenants, ingestion_events, audit_log, billing_records, etc.)

Do NOT use TimescaleDB hypertables for non-time-series data. The operational complexity is only justified when query patterns are predominantly time-range filtered.

---

## PgBouncer Connection Pooling Standard (v1.3.0 Amendment)

Added in ADR-005 v1.3.0 (2023-04-02) after a connection storm during a traffic spike in Q1 2023 exhausted RDS `max_connections`.

All application services MUST connect to PostgreSQL through PgBouncer, never directly to the RDS endpoint. The RDS endpoint is reserved for SRE/DBA direct access and `golang-migrate` during CI.

### Pooling Mode

PgBouncer is configured in **transaction pooling** mode (`pool_mode = transaction`). In this mode, a server connection is held only for the duration of a transaction — not the entire client session lifetime. This allows 500+ client connections to share a pool of 180 server connections.

```ini
# pgbouncer.ini — key settings
[databases]
aetherflow = host=postgres-primary.aetherflow.internal
             port=5432
             dbname=aetherflow

[pgbouncer]
pool_mode            = transaction
max_client_conn      = 500
default_pool_size    = 30    ; server connections per (db, user) pair
reserve_pool_size    = 10    ; emergency overflow connections
reserve_pool_timeout = 3     ; seconds before using reserve pool
server_idle_timeout  = 600   ; recycle idle server connections after 10 min
client_idle_timeout  = 300   ; drop idle client connections after 5 min
```

### Transaction Pooling Constraints

Transaction mode is powerful but has one hard constraint: **session-level features do not persist across transactions.** Code that relies on the following will break silently in transaction pooling mode:

```sql
-- ❌ BROKEN in transaction pooling: SET is session-scoped
SET search_path TO tenant_acme_corp, public;
-- The next query may run on a different server connection with the default search_path

-- ❌ BROKEN in transaction pooling: advisory locks are session-scoped
SELECT pg_advisory_lock(12345);
-- Lock is released when the server connection is returned to the pool

-- ❌ BROKEN in transaction pooling: prepared statements are session-scoped
PREPARE my_plan AS SELECT * FROM tenants WHERE id = $1;
-- Plan is not available on the next server connection

-- ✅ SAFE: Everything within a single explicit transaction block
BEGIN;
  SET LOCAL search_path TO tenant_acme_corp, public; -- LOCAL = transaction-scoped
  SELECT * FROM events WHERE tenant_id = $1;
COMMIT;
```

The `sarama` Kafka client and `database/sql` standard library are both compatible with transaction pooling as long as engineers avoid session-scoped features. If you encounter an inexplicable query behavior that "works without PgBouncer," session-scoped state is almost always the cause.

---

## Operational Runbooks

Day-to-day PostgreSQL operations, failover procedures, and disaster recovery are covered in `runbook-database-failover.md`. Key quick-references:

```bash
# Connect to production primary (via PgBouncer)
psql -h pgbouncer.aetherflow.internal -U aetherflow_admin -d aetherflow

# Check connection counts by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY count DESC;

# Find long-running queries (> 5 minutes)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > INTERVAL '5 minutes'
  AND state != 'idle'
ORDER BY duration DESC;

# Check table bloat on large tables
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

---

## Consequences

### Positive

- Strong ACID guarantees for billing and tenant provisioning — zero data integrity incidents since migration from MongoDB
- SQL expressiveness enables ad-hoc internal analytics without a dedicated analytics platform
- AWS RDS Multi-AZ provides 99.95% uptime SLA with automatic failover
- TimescaleDB continuous aggregates reduced metrics query P99 from 4.2s → 180ms for 30-day window queries
- `golang-migrate` sequential migration files give a complete, auditable history of every schema change

### Negative

- Vertical scaling ceiling — at some future write throughput, we will need to consider sharding or a write-optimized alternative for the `events` table. Current estimate: this becomes necessary at ~500k events/sec (current peak: 140k). Headroom is sufficient for 18–24 months at current growth.
- TimescaleDB adds operational complexity to the tables it manages — chunk management, compression policy monitoring, and continuous aggregate refresh require SRE attention
- PgBouncer transaction pooling mode is a source of subtle bugs for engineers unfamiliar with its constraints (see above)

### Future Considerations

At AetherFlow's current growth rate, two signals should trigger revisiting this ADR:

1. **Write throughput approaching 400k events/sec** → evaluate CockroachDB or Citus for the `events` table specifically (not all tables)
2. **`events` table exceeding 5TB per shard** → evaluate archival to S3 + Athena for cold query access, retaining only a rolling 90-day window in PostgreSQL

Neither condition is close at present. The ADR review trigger is tracked as a Jira ticket: `PLAT-3401`.

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | 2022-05-10 | Initial acceptance — PostgreSQL as primary store |
| 1.1.0 | 2022-08-22 | Added schema governance standards, naming conventions, migration rules |
| 1.2.0 | 2022-11-15 | Added TimescaleDB extension decision for metrics time-series data |
| 1.3.0 | 2023-04-02 | Added PgBouncer connection pooling standard; transaction mode constraints |

---

*Document Owner: Platform Team | Review Cycle: Annual or on trigger conditions (see Future Considerations)*
*Next Scheduled Review: 2024-05-10 | Related: `runbook-database-failover.md`, `reference-infra-terraform.md`*
