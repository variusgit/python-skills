---
name: arch-data-platform
description: Data platform architecture — platform topology layers, storage responsibilities, data governance, CDC patterns, streaming vs batch decisions, data contracts, backfill architecture, and cost management. Use when designing or evolving data platform topology and data integration patterns.
---

# Data Platform Architecture

Architectural patterns for designing data platforms that are correct, replayable, governed, and cost-efficient within the fixed stack (Airflow, PySpark, PostgreSQL, S3, Greenplum/ClickHouse).

## When to use

- Designing or evolving data platform topology (ingestion → processing → serving).
- Defining data ownership, governance, and quality contracts.
- Choosing between CDC, event-based, and batch ingestion patterns.
- Deciding streaming vs batch architecture.
- Planning data lifecycle, retention, and cost management.

## Platform topology

### Canonical layers

```
Sources → Ingestion → Raw → Curated → Serving → Consumers
```

| Layer | Purpose | Storage | Ownership |
|-------|---------|---------|-----------|
| **Sources** | Operational databases, APIs, external feeds, event streams | Source systems (PostgreSQL, third-party) | Source teams |
| **Ingestion** | Extract and transport data from sources | Transient (pipeline state) | Data platform team |
| **Raw / Landing** | Immutable copy of source data, as-is | S3 (Parquet/JSON) | Data platform team |
| **Curated / Processed** | Cleaned, validated, enriched, conformed | S3 (Parquet) | Domain data owners |
| **Serving** | Optimized for consumer access patterns | Greenplum/ClickHouse, PostgreSQL (read replicas), S3 | Domain data owners |
| **Consumers** | Applications, dashboards, ML models, APIs | N/A | Consumer teams |

### Design rules

- **Raw is immutable** — never modify raw data; always reprocess from raw to curated.
- **Curated is the contract** — consumers depend on curated layer schemas, not raw.
- **Serving is optimized** — may denormalize, pre-aggregate, or materialize for query performance.
- **Each layer has an owner** — no orphan datasets.

## Storage responsibilities

| Engine | Role | Use for | Do NOT use for |
|--------|------|---------|---------------|
| **PostgreSQL** | OLTP, metadata, coordination | Transactional workflows, application state, idempotency keys, pipeline metadata | Large analytical scans, append-heavy event storage |
| **S3 (Parquet)** | Immutable replayable storage, OLAP | Raw/curated data lake, batch job outputs, ML training data | Low-latency point lookups, transactional writes |
| **Greenplum / ClickHouse** | Analytical serving | Aggregation-heavy queries, dashboards, reporting, ad-hoc analysis | Source of truth for transactional workflows |

Blurring these responsibilities requires explicit ADR with justification.

## Data governance

### Data ownership

Every dataset must have:
- **Owner** — team or person responsible for correctness, freshness, and evolution.
- **Classification** — sensitivity level (public, internal, confidential, PII).
- **Retention policy** — how long data is kept, when and how it is deleted.
- **SLA** — freshness, completeness, availability guarantees.

### Data contracts

A data contract defines the agreement between producer and consumer:

- **Schema** — field names, types, nullability. Machine-checkable (JSON Schema, Protobuf, Avro, Parquet schema).
- **Semantics** — what each field means, units, timezone conventions.
- **Freshness SLA** — maximum acceptable lag from source event to availability in serving layer.
- **Completeness** — expected row count ranges, partition expectations.
- **Compatibility rules** — additive changes only (new nullable fields) without breaking consumers.

Breaking changes require: versioning, migration period, consumer notification.

### Data quality framework

Monitor per dataset:

| Dimension | What to check | Example signal |
|-----------|---------------|----------------|
| **Completeness** | Expected partitions present? Row count in expected range? | Partition count vs expected, row count anomaly detection |
| **Freshness** | Data available within SLA? | max(event_time) vs wall clock, pipeline duration |
| **Accuracy** | Values pass domain-specific checks? | Null rate, range violations, referential integrity |
| **Uniqueness** | No unexpected duplicates? | Duplicate key count per batch |
| **Consistency** | Cross-dataset invariants hold? | FK references exist in target, sum reconciliation |

### Data lineage

Track how data flows from source to serving:
- Which sources feed which curated datasets.
- Which transformations are applied.
- Which consumers depend on which datasets.

Lineage is essential for impact analysis (what breaks if source X changes) and compliance (which systems handle PII).

## CDC patterns (Change Data Capture)

### Decision framework

| Pattern | How | Latency | Complexity | Best for |
|---------|-----|---------|------------|----------|
| **Log-based CDC** | Read database WAL/binlog (Debezium, pg_logical) | Seconds | High (infra setup, schema evolution) | Near-real-time replication, event-driven pipelines |
| **Query-based CDC** | Poll source with watermark (WHERE updated_at > hwm) | Minutes | Low | Batch pipelines, sources without WAL access |
| **Outbox pattern** | Write domain event to outbox table in same transaction, async publish | Seconds | Medium | Transactional consistency between DB and events |
| **Full snapshot** | Periodically dump and diff | Hours | Low | Small tables, infrequent changes, initial loads |

### Decision rules

- **Default**: start with query-based CDC (watermark polling). Simplest, works with Airflow.
- **Upgrade to log-based**: when latency requirement is seconds, not minutes.
- **Use outbox**: when domain events must be atomically consistent with DB writes.
- **Avoid full snapshot** for large tables — use only for reference/dimension data or initial bootstrap.

## Streaming vs batch

### Decision framework

| Criteria | Batch | Streaming |
|----------|-------|-----------|
| **Latency requirement** | Minutes to hours acceptable | Seconds required |
| **Data volume** | Large volumes, periodic processing | Continuous flow, event-by-event |
| **Complexity** | Lower (bounded datasets, clear start/end) | Higher (unbounded, ordering, late data) |
| **Error recovery** | Replay from source, reprocess partition | Complex (offset management, state recovery) |
| **Cost** | Pay for compute when running | Pay for always-on infrastructure |
| **Stack fit** | Airflow + PySpark (native) | Requires add-on (Kafka, Flink) |

### Recommendations

- **Start with batch** unless seconds-latency is a hard requirement. Batch is simpler, cheaper, and native to the stack (Airflow + PySpark).
- **Add streaming selectively** for specific use cases (real-time alerts, live dashboards, fraud detection). Treat streaming infrastructure as an add-on with ADR.
- **Avoid Lambda architecture** (parallel batch + stream) unless absolutely necessary — operational burden is high. Prefer Kappa (streaming only) or batch-only with reduced interval.

## Backfill and reprocessing architecture

Every pipeline must support reprocessing as a standard operation, not an emergency procedure.

### Design requirements

- **Replayable source** — raw layer in S3 is the canonical replay source. Never depend on source system for historical reprocessing.
- **Idempotent processing** — reprocessing the same input produces the same output. Overwrite-by-partition or staging + commit.
- **Isolation** — backfills must not interfere with production SLAs. Use separate compute resources, throttling, and scheduling.
- **Validation** — backfilled data must pass quality checks before promoting to serving layer.
- **Bounded** — explicit time/ID range. Never "reprocess everything" without scoping.

### Backfill topology

```
Raw (S3, immutable) → Backfill pipeline (scoped, throttled) → Staging → Validation → Promote to curated/serving
```

## Cost management

### Storage

- **Tiering** — hot data in Greenplum/ClickHouse, warm data in S3 (Parquet), cold data in S3 (Glacier/archive).
- **Retention** — define per dataset. Delete or archive data beyond retention period automatically.
- **Compression** — Snappy (default), Zstd (when ratio matters), align with query patterns.

### Compute

- **Right-size PySpark jobs** — tune executor count, memory, and parallelism per job, not globally.
- **Schedule awareness** — run heavy jobs during off-peak hours.
- **Avoid always-on** for batch workloads — use Airflow-triggered ephemeral clusters or K8s jobs.

### Monitoring

- Track cost per pipeline, per dataset, per team.
- Alert on cost anomalies (sudden spike in compute or storage).
- Review cost allocation monthly.

## Checklist

- Platform topology layers are defined with clear ownership per layer.
- Storage engine responsibilities are explicit (OLTP vs OLAP vs object storage).
- Data contracts are defined for curated layer datasets.
- Data quality checks cover completeness, freshness, accuracy, uniqueness.
- CDC pattern is chosen based on latency requirements and source capabilities.
- Batch vs streaming decision is justified with latency requirements.
- Backfill/reprocessing paths are designed from the start.
- Data retention and cost management policies are defined.
- Data lineage is traceable from source to serving.

## Failure modes

- No data ownership — orphan datasets with no maintenance, quality, or evolution.
- Raw data is mutable — reprocessing produces different results each time.
- CDC pattern mismatch — log-based CDC for a use case that only needs daily batch.
- Missing data contracts — consumers break silently on schema changes.
- No backfill path — emergency reprocessing requires ad-hoc scripts and manual intervention.
- Streaming added without justification — always-on infrastructure cost for a use case that tolerates minutes of latency.
- Data quality checks missing — stale or corrupt data propagates to serving layer undetected.
- Cost not monitored — storage grows unbounded, compute costs spike unnoticed.
