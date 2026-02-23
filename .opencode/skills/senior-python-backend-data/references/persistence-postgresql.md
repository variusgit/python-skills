# PostgreSQL Persistence (Schema, Queries, Migrations)

This document defines PostgreSQL practices for correctness, data integrity, and safe evolution.

## When to use

- Designing/changing PostgreSQL schemas and constraints.
- Implementing transactional write paths and concurrency control.
- Planning/authoring migrations, backfills, and rollback strategies.
- Reviewing query performance and integrity risks in production flows.

## Schema design (encode invariants)

- Use constraints to enforce invariants:
  - `NOT NULL`, `CHECK`, `UNIQUE`, `FOREIGN KEY`
- Prefer explicit columns over “JSON blob by default”.
  - Use `JSONB` only when there is a clear query strategy and constraints are still enforceable.
- Choose primary keys deliberately (UUID vs bigint vs natural keys).
- Index based on query patterns; confirm with `EXPLAIN (ANALYZE, BUFFERS)` for hot paths.

## Transactions & concurrency

- Be explicit about transaction boundaries.
- Do not hold a DB transaction across network calls.
- For concurrency-sensitive flows:
  - choose an isolation level intentionally
  - use locks or `SELECT ... FOR UPDATE` where appropriate
  - ensure the code is safe under retries

## Idempotent writes

If an operation can be retried (API writes, consumers, Airflow tasks):
- Use **idempotency keys** stored in the DB when correctness requires it.
- Prefer “insert once” semantics:
  - `INSERT ... ON CONFLICT DO NOTHING/UPDATE`
  - unique constraint on the idempotency key or natural key
- Record the final outcome for repeated requests if you need “exactly-once visible effect”.

## Query hygiene

- Never ship unbounded production queries; apply limits and pagination strategies.
- Avoid N+1 patterns; batch where possible.
- Use the right indexes; beware of functions on indexed columns that defeat index usage.
- For long-running/reporting queries, consider offloading to the analytics store (Greenplum/ClickHouse).

## Migrations: expand/contract by default

Default strategy for safe migrations:

1. **Expand**
   - add new nullable columns / tables
   - add indexes (concurrently where available/appropriate)
   - add new code paths with backward-compatible reads
2. **Backfill**
   - run a throttled backfill with observability
   - ensure it is restartable/idempotent
3. **Switch reads**
   - move reads to the new schema/path
4. **Contract**
   - enforce constraints (NOT NULL, CHECK)
   - drop old columns/tables only after confidence

Always include:
- rollback/backout plan
- lock/latency risk evaluation (especially for large tables)
- migration runtime expectations

## Backfills

Backfills must be:
- bounded (explicit range)
- throttled (rate limits; avoid saturating DB)
- observable (progress metrics/logs)
- safe to resume (checkpointing or idempotent partition strategy)

## Checklist

- Which invariants are affected?
- Are they enforced in constraints and tested?
- What is the rollback plan?
- What queries are impacted and are they indexed?
- Are timeouts and connection pool settings appropriate?

## Failure modes

- Big-bang migrations without expand/contract and rollback clarity.
- Retry-enabled non-idempotent writes causing duplicate/corrupt state.
- Long transactions across network calls increasing lock contention.
- Missing/incorrect indexes on hot paths causing latency regressions.
- Backfills without bounds/throttling saturating primary workloads.
