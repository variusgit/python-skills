# Analytics Store (Greenplum or ClickHouse)

This document provides minimal guidance for data-adjacent backend work that touches an MPP analytics store.

## When to use the analytics store

Use Greenplum/ClickHouse for:
- large scans and analytical queries
- aggregation-heavy workloads
- reporting datasets
Avoid using it as the source of truth for transactional workflows; use PostgreSQL for that.

Read `python-best-practices.md` first, then use this document for analytics-store engine decisions.

## Ingestion patterns

- Prefer **append + dedup** or **partition overwrite** strategies depending on the engine.
- Keep ingestion idempotent:
  - stable primary/dedup keys
  - deterministic partition selection
- Track schema versions and job versions.

## Schema and performance basics

### ClickHouse
- Choose table engines intentionally (MergeTree family commonly).
- Define `ORDER BY` keys for query patterns.
- Use partition keys aligned with time/range queries.
- Avoid high-cardinality partitions that explode part counts.

### Greenplum
- Choose distribution keys for join locality.
- Be careful with skew and large data movement.
- Use partitioning for pruning and incremental loads.

## Data quality

- Validate row counts and basic constraints on load.
- Maintain freshness metrics and completeness checks.
- Document canonical datasets and ownership.

## Checklist

- Is the load idempotent?
- Do keys/partitioning align with query patterns?
- Are there freshness/completeness signals?
- Is the dataset contract documented?

## Failure modes

- Treating MPP store as transactional source of truth.
- Poor partition/distribution keys causing skew and heavy shuffles.
- Non-idempotent ingestion producing duplicates on retries.
- Missing freshness/completeness checks hiding stale/broken datasets.
- Engine defaults chosen without query-pattern validation.
