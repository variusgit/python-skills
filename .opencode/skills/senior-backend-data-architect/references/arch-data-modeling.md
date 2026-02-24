---
name: arch-data-modeling
description: Data modeling patterns for the stack — OLTP normalization, dimensional modeling, data vault, slowly changing dimensions, time-series models, and schema design per storage engine (PostgreSQL, ClickHouse, Greenplum, S3/Parquet). Use when designing data models for transactional, analytical, or ML workloads.
---

# Data Modeling

Data modeling patterns and decision frameworks for designing schemas that serve their intended access patterns correctly and efficiently.

## When to use

- Designing data models for new entities, domains, or systems.
- Choosing between normalized, dimensional, or vault modeling approaches.
- Designing schemas for specific storage engines (PostgreSQL, ClickHouse, Greenplum, Parquet).
- Planning schema evolution and migration strategies.
- Modeling time-series or slowly changing data.

## Modeling principles

- **Model for the access pattern** — OLTP models optimize for writes and point reads; OLAP models optimize for scans and aggregations. Never force one model to serve both.
- **Invariants in the model** — constraints, keys, and relationships enforce business rules. A model without enforced invariants is a spreadsheet.
- **Explicit ownership** — every table/dataset has a bounded context that owns it. No shared writable tables across contexts.
- **Evolution strategy** — design models expecting change. Additive changes (new nullable columns) are safe; breaking changes require expand/contract migration.

## OLTP modeling (PostgreSQL)

### Normalization

Normalize to Third Normal Form (3NF) by default for transactional models:

- **1NF**: no repeating groups, atomic values.
- **2NF**: no partial dependencies on composite keys.
- **3NF**: no transitive dependencies (non-key columns depend only on the primary key).

Denormalize selectively when:
- Read performance on a specific query is proven insufficient with proper indexing.
- The denormalized data is a pre-computed materialized view, not the source of truth.
- The denormalization is documented with ADR.

### Primary key strategy

| Type | When | Trade-offs |
|------|------|------------|
| `bigint GENERATED ALWAYS AS IDENTITY` | Internal tables, sequences, FK targets | Compact, fast joins. Exposes ordering and count. |
| `uuid` (v4 or v7) | Public-facing IDs, cross-service references | No ordering leaked. Larger index, random I/O with v4. Prefer v7 (time-ordered). |
| Natural key `(tenant_id, email)` | Stable business-meaningful uniqueness | No surrogate. Harder to migrate if key changes. |

Default: `bigint` as internal PK + separate `uuid` column for external exposure.

### Constraint design

```sql
CREATE TABLE orders (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    idempotency_key uuid NOT NULL,
    user_id         bigint NOT NULL REFERENCES users(id),
    status          text NOT NULL CHECK (status IN ('pending', 'confirmed', 'cancelled')),
    amount_cents    integer NOT NULL CHECK (amount_cents > 0),
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT uq_orders_idempotency_key UNIQUE (idempotency_key)
);
```

Rules:
- Default to `NOT NULL` — nullable only when null has explicit business meaning.
- `CHECK` for value-domain invariants (status enums, positive amounts).
- `UNIQUE` for business-key uniqueness and idempotency.
- `FOREIGN KEY` for referential integrity.
- Name constraints explicitly for readable error messages.
- Use `text + CHECK` for enums, not `CREATE TYPE ... AS ENUM` (DDL fragility in production).

### Audit columns

Every mutable table:
```sql
created_at  timestamptz NOT NULL DEFAULT now(),
updated_at  timestamptz NOT NULL DEFAULT now()
```

Essential for incremental loads, debugging, and data lineage.

### Index design

- Index columns that appear in `WHERE`, `JOIN`, `ORDER BY` for hot-path queries.
- Composite index column order: most selective first. `(a, b)` supports `a` and `(a, b)` queries, NOT `b` alone.
- Partial indexes for queries filtering on a condition excluding most rows (e.g., `WHERE status = 'pending'`).
- Covering indexes (`INCLUDE`) for index-only scans on latency-sensitive queries.
- Always `CREATE INDEX CONCURRENTLY` on production tables.

## Dimensional modeling (OLAP)

For analytical workloads: dashboards, reports, ad-hoc queries.

### Star schema

Central **fact table** (events, transactions, measurements) surrounded by **dimension tables** (who, what, where, when).

```
          dim_customer
              |
dim_date -- fact_orders -- dim_product
              |
          dim_region
```

**Fact tables**:
- Contain measurable business events (amounts, counts, durations).
- Granularity must be explicit: one row per order? per order line? per click?
- Keys: surrogate key + references to dimension keys + degenerate dimension keys (order number).

**Dimension tables**:
- Descriptive attributes for filtering, grouping, labeling.
- Relatively small, change slowly.
- Include a surrogate key (dimension_id) for join stability.

### Fact table types

| Type | Granularity | Example |
|------|-------------|---------|
| **Transaction fact** | One row per event | Order placed, payment received, click |
| **Periodic snapshot** | One row per entity per period | Daily account balance, weekly inventory |
| **Accumulating snapshot** | One row per lifecycle, updated as milestones are reached | Order lifecycle (placed → shipped → delivered) |

### Snowflake schema

Dimensions further normalized into sub-dimensions. Use sparingly — increases join complexity for marginal storage savings.

### Conformed dimensions

Dimensions shared across multiple fact tables (e.g., `dim_date`, `dim_customer`). Ensure:
- Same surrogate key generation logic.
- Same attribute definitions and naming.
- Maintained centrally, consumed by all fact tables.

## Slowly Changing Dimensions (SCD)

When dimension attributes change over time.

| Type | Strategy | Use when |
|------|----------|----------|
| **SCD Type 1** | Overwrite old value | History is not needed (e.g., fix a typo in customer name) |
| **SCD Type 2** | Add new row with version, effective_from/to dates | Full history needed (e.g., customer address changes, need to know address at time of order) |
| **SCD Type 3** | Add previous_value column | Only current and one previous value needed |
| **SCD Type 4** | Separate history table | Dimension is large, history queries are rare |

Default: **SCD Type 2** for business-critical dimensions where history matters. Type 1 for corrections.

### SCD Type 2 schema pattern

```sql
CREATE TABLE dim_customer (
    customer_key    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id     bigint NOT NULL,       -- natural key (stable)
    name            text NOT NULL,
    segment         text NOT NULL,
    region          text NOT NULL,
    effective_from  date NOT NULL,
    effective_to    date,                   -- NULL = current
    is_current      boolean NOT NULL DEFAULT true,

    CONSTRAINT uq_customer_current UNIQUE (customer_id) WHERE (is_current)
);
```

## Data vault

Hub-and-spoke model for enterprise data warehouses that need auditability and flexibility.

### Components

- **Hub**: business key + surrogate key + load metadata. Represents a business entity.
- **Link**: many-to-many relationship between hubs. Represents associations.
- **Satellite**: descriptive attributes attached to hub or link, with temporal versioning (load_date, source).

### When to use

- Enterprise DWH with many source systems.
- Auditability and full traceability are regulatory requirements.
- Schema must evolve frequently without breaking existing structures.

### When NOT to use

- Single-source analytical models.
- Team unfamiliar with data vault methodology.
- Simple star schema meets requirements.

## Modeling per storage engine

### PostgreSQL

- 3NF normalized, constraint-heavy, foreign keys enforced.
- Optimize for transactional writes and point reads.
- Use partial indexes for filtered queries on large tables.
- JSONB for genuinely semi-structured data only (not to avoid schema design).

### ClickHouse

- **Table engine**: MergeTree family (ReplacingMergeTree for dedup, AggregatingMergeTree for pre-aggregation).
- **ORDER BY key**: must match primary query filter pattern. Determines data layout on disk.
- **Partition key**: typically time-based (toYYYYMM). Keep partition count bounded (hundreds, not thousands).
- **Denormalize aggressively** — ClickHouse optimizes for wide, flat tables. Joins are expensive.
- **Do NOT use** ClickHouse for: transactional writes, point lookups by arbitrary key, source of truth for mutable data.

### Greenplum

- **Distribution key**: choose for join locality (co-locate frequently joined tables on the same key).
- **Avoid skew**: distribution key must have high cardinality and even distribution.
- **Partition by range**: typically by date for partition pruning and incremental loads.
- **AO (Append-Optimized) tables** for analytical workloads, heap tables for frequently updated data.

### S3 / Parquet

- **Partition layout**: Hive-style paths (`year=YYYY/month=MM/day=DD/`).
- **File sizing**: 128 MB – 1 GB per file within a partition.
- **Compression**: Snappy (default), Zstd (higher ratio when needed).
- **Schema evolution**: add nullable columns only (safe). Type changes require expand/contract.
- **Immutable by default** — overwrite-by-partition for idempotent writes.

## Schema evolution strategy

- **Safe changes**: add nullable columns, add new tables, add indexes.
- **Unsafe changes**: remove columns, rename columns, change types, change partition keys.
- **Strategy**: expand/contract migration.
  1. Expand — add new structure alongside old.
  2. Backfill — populate new structure.
  3. Switch reads — consumers use new structure.
  4. Contract — remove old structure after confidence period.

Track schema version in metadata or partition path when evolution is expected.

## Checklist

- Modeling approach matches access pattern (OLTP vs OLAP).
- Storage engine is chosen per model with explicit justification.
- Primary keys are chosen deliberately (bigint vs UUID vs natural key).
- Constraints enforce invariants at the database level.
- Dimensional model has explicit grain, conformed dimensions, and SCD strategy.
- Schema evolution strategy is defined (expand/contract for breaking changes).
- Engine-specific design rules are followed (ClickHouse ORDER BY, Greenplum distribution, Parquet partitioning).

## Failure modes

- OLTP model used for analytical workloads (slow scans, missing aggregations).
- Dimensional model without explicit grain (ambiguous fact table, wrong aggregation results).
- SCD not applied where history is needed (lost audit trail, incorrect point-in-time analysis).
- ClickHouse ORDER BY key not matching query patterns (full scans instead of indexed reads).
- Greenplum distribution key causing data skew (one segment does all the work).
- Parquet files too small (thousands of tiny files increasing list/read cost) or too large (single file prevents parallelism).
- Schema evolution without expand/contract (breaking consumers on deploy).
- Constraints missing — invariants enforced only in application code, bypassed by direct DB access or data loads.
