---
name: python-postgresql
description: PostgreSQL practices for schema design, constraint enforcement, indexing, transactions, concurrency, idempotent writes, query patterns, connection management, expand/contract migrations, and backfill safety. Use when designing or changing schemas, writing migrations, or reviewing query/transaction correctness.
---

# PostgreSQL Persistence (Schema, Queries, Migrations)

This document defines PostgreSQL practices for correctness, data integrity, and safe evolution.

Read `./python-best-practices.md` first, then apply this document for persistence-specific decisions.

## When to use

- Designing/changing PostgreSQL schemas and constraints.
- Implementing transactional write paths and concurrency control.
- Planning/authoring migrations, backfills, and rollback strategies.
- Reviewing query performance and integrity risks in production flows.
- Configuring connection management and timeouts.

## Schema design (encode invariants)

### Constraints

Use constraints to enforce invariants at the database level — they are the last line of defense:

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
- Default to `NOT NULL` — make columns nullable only when null has explicit business meaning.
- Use `CHECK` for value-domain invariants (status enums, positive amounts, valid ranges).
- Use `UNIQUE` for business-key uniqueness and idempotency enforcement.
- Use `FOREIGN KEY` for referential integrity between entities.
- Name constraints explicitly (`CONSTRAINT uq_...`) for readable error messages and migration references.

### Primary key choice

| Type | When to use | Trade-offs |
|------|-------------|------------|
| `bigint GENERATED ALWAYS AS IDENTITY` | Default for internal tables | Compact, fast joins, sequential — but exposes ordering and count |
| `uuid` (v4 or v7) | Public-facing IDs, distributed systems, cross-service references | No ordering info leaked — but larger index, random I/O with v4 |
| Natural key (e.g., `(tenant_id, email)`) | When a stable business key exists and uniqueness is guaranteed | No surrogate — but migrations harder if the natural key changes |

Guidance:
- Prefer `bigint` as internal PK + a separate `uuid` column for external/API exposure.
- If using UUID as PK, prefer UUIDv7 (time-ordered) to reduce index fragmentation.
- Never use mutable business fields as PK.

### Audit columns

Include on all mutable tables:

```sql
created_at  timestamptz NOT NULL DEFAULT now(),
updated_at  timestamptz NOT NULL DEFAULT now()
```

Update `updated_at` via application code or trigger. These columns are essential for incremental loads, debugging, and audit trails.

### JSONB guidance

- Use JSONB when schema is genuinely semi-structured (user preferences, dynamic metadata, third-party payloads).
- Do NOT use JSONB to avoid schema design — if you query or constrain a field regularly, it should be a column.
- Index JSONB with GIN when querying by keys:

```sql
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata jsonb_path_ops);
```

- Combine JSONB with CHECK constraints where possible:

```sql
CHECK (metadata ? 'version')  -- key must exist
```

### Enum types vs lookup tables

- Use `text` + `CHECK` constraint for small, stable enum sets (status, type).
- Use a lookup/reference table when the set changes at runtime or needs its own attributes.
- Avoid PostgreSQL `CREATE TYPE ... AS ENUM` in production — adding values requires DDL, removing values is not supported, and migration tooling is fragile.

## Indexing

### Index types

| Type | Best for | Example |
|------|----------|---------|
| **B-tree** (default) | Equality, range, sorting, uniqueness | `CREATE INDEX idx_orders_user_id ON orders(user_id)` |
| **GIN** | JSONB containment, full-text search, array membership | `USING GIN (metadata jsonb_path_ops)` |
| **GiST** | Geometric, range types, nearest-neighbor | `USING GiST (location)` |
| **BRIN** | Very large tables with natural physical ordering (time-series) | `USING BRIN (created_at)` |

Default to B-tree unless the query pattern specifically benefits from another type.

### Composite indexes

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

- Column order matters: put the most selective (highest cardinality filter) first.
- A composite index on `(a, b)` supports queries filtering on `a` or `(a, b)`, but NOT on `b` alone.
- For sorting: index column order must match the `ORDER BY` clause.

### Partial indexes

```sql
CREATE INDEX idx_orders_pending ON orders(created_at)
    WHERE status = 'pending';
```

- Use when queries consistently filter on a condition that excludes most rows.
- Significantly smaller and faster than full-table indexes.
- Common patterns: active records, unprocessed items, non-deleted rows.

### Covering indexes (INCLUDE)

```sql
CREATE INDEX idx_orders_user_covering ON orders(user_id) INCLUDE (status, amount_cents);
```

- Allows index-only scans by including non-key columns in the index.
- Use for hot queries where avoiding the heap lookup matters for latency.

### CREATE INDEX CONCURRENTLY

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

- **Always use CONCURRENTLY** for indexes on production tables — regular `CREATE INDEX` takes an exclusive lock that blocks all writes.
- CONCURRENTLY takes longer but does not block reads or writes.
- Cannot run inside a transaction block.
- If it fails partway, a broken index is left behind — drop it and retry.

### Verify with EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';
```

Key things to check:
- **Seq Scan** on a large table = missing index.
- **Index Scan** or **Index Only Scan** = index is used.
- **Rows** estimated vs actual — large mismatch = stale statistics (`ANALYZE`).
- **Buffers shared hit vs read** — high read count = data not cached, may need optimization.

## Transactions & concurrency

### Isolation levels

| Level | Behavior | When to use |
|-------|----------|-------------|
| **READ COMMITTED** (default) | Each statement sees latest committed data | Default for most OLTP — safe, predictable |
| **REPEATABLE READ** | Transaction sees a snapshot from its start | Reports or multi-step reads that must be consistent |
| **SERIALIZABLE** | Full serializability — detects all anomalies | Critical financial operations where correctness outweighs throughput |

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- all queries in this transaction see the same snapshot
SELECT balance FROM accounts WHERE id = 1;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

Guidance:
- Stick with READ COMMITTED unless you have a specific anomaly to prevent.
- SERIALIZABLE can cause serialization failures — code must handle retries.

### Explicit transaction boundaries in Python

```python
import psycopg

def transfer_funds(conn_info: str, from_id: int, to_id: int, amount: int) -> None:
    with psycopg.connect(conn_info) as conn:
        with conn.transaction():
            with conn.cursor() as cur:
                cur.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id),
                )
                cur.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id),
                )
```

Rules:
- Keep transactions short — do all computation before, execute DB operations, commit.
- **Never hold a transaction open across network calls** (HTTP, gRPC, queue publish).
- On error, the context manager rolls back automatically.

### SELECT ... FOR UPDATE

```sql
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
-- row is locked until transaction commits
UPDATE orders SET status = 'confirmed' WHERE id = 42;
```

- Use when you need read-modify-write on a specific row without racing.
- `FOR UPDATE SKIP LOCKED` — skip already-locked rows (useful for worker queues).
- `FOR UPDATE NOWAIT` — fail immediately if row is locked (avoid waiting).

### Advisory locks

```sql
SELECT pg_advisory_lock(hashtext('process_batch_42'));
-- critical section
SELECT pg_advisory_unlock(hashtext('process_batch_42'));
```

- Use for application-level mutual exclusion (batch processing, singleton jobs).
- Lighter than row locks — no table involved.
- Always release explicitly or use session-level locks that release on disconnect.

### Deadlock avoidance

- Lock resources in a consistent global order (e.g., always lock by ascending `id`).
- Keep transactions short to minimize lock hold time.
- If deadlocks occur, PostgreSQL will detect and abort one transaction — code must retry.

## Idempotent writes

### INSERT ... ON CONFLICT (primary pattern)

```sql
-- Insert or silently skip if duplicate
INSERT INTO orders (idempotency_key, user_id, amount_cents, status)
VALUES ('key-abc-123', 42, 5000, 'pending')
ON CONFLICT (idempotency_key) DO NOTHING;

-- Insert or update specific fields on duplicate
INSERT INTO orders (idempotency_key, user_id, amount_cents, status)
VALUES ('key-abc-123', 42, 5000, 'pending')
ON CONFLICT (idempotency_key) DO UPDATE SET
    updated_at = now()
WHERE orders.status = 'pending';  -- only update if still pending
```

Rules:
- Requires a `UNIQUE` constraint on the conflict target.
- `DO NOTHING` for true insert-once semantics.
- `DO UPDATE` with a `WHERE` clause to prevent overwriting finalized states.

### Idempotency table pattern

For operations with external side effects (send email, charge payment):

```sql
CREATE TABLE idempotency_keys (
    key         text PRIMARY KEY,
    result      jsonb,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

Flow:
1. Check if key exists → if yes, return stored result.
2. Execute the operation.
3. Store the result with the key in the same transaction as the DB mutation.
4. Return the result.

This ensures "exactly-once visible effect" even under retries.

### Race condition: check-then-act vs try-insert

```python
# BAD — race condition between check and insert
existing = db.query("SELECT 1 FROM orders WHERE idempotency_key = %s", key)
if not existing:
    db.execute("INSERT INTO orders ...")  # two concurrent requests both pass the check

# GOOD — atomic upsert, no race
db.execute("""
    INSERT INTO orders (idempotency_key, ...) VALUES (%s, ...)
    ON CONFLICT (idempotency_key) DO NOTHING
""", key)
```

Always prefer the atomic database operation over application-level check-then-act.

## Query patterns

### Cursor-based pagination

```sql
-- First page
SELECT id, name, created_at FROM users
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (using last row's values as cursor)
SELECT id, name, created_at FROM users
WHERE (created_at, id) < ('2025-01-15T10:00:00Z', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

- Use tuple comparison for stable, index-friendly pagination.
- Requires a unique tiebreaker column (`id`) when the sort column has duplicates.
- Avoid `OFFSET` for large datasets — it scans and discards rows.

### Batch operations

```sql
-- Batch insert with VALUES list
INSERT INTO events (type, payload, created_at)
VALUES
    ('click', '{"page": "/home"}', now()),
    ('click', '{"page": "/about"}', now()),
    ('view', '{"page": "/pricing"}', now());

-- Batch update with FROM
UPDATE orders SET status = 'cancelled'
FROM (VALUES (1), (2), (3)) AS batch(id)
WHERE orders.id = batch.id AND orders.status = 'pending';
```

- Batch inserts: 100-1000 rows per statement is typically optimal.
- Avoid single-row INSERT in a loop — network round-trips dominate.
- For very large batches, use `COPY` for maximum throughput.

### Query timeouts

```sql
-- Per-session timeout
SET statement_timeout = '5s';

-- Per-transaction timeout
SET LOCAL statement_timeout = '30s';
```

```python
# In application code (psycopg)
with conn.cursor() as cur:
    cur.execute("SET LOCAL statement_timeout = '5s'")
    cur.execute("SELECT * FROM large_table WHERE ...")
```

- Set `statement_timeout` for all production queries — prevent runaway queries from saturating the DB.
- Use shorter timeouts for user-facing paths (1-5s), longer for background/batch (30s-5min).
- Configure a global default in `postgresql.conf` as a safety net.

## Connection management

### Connection pooling

- Use a connection pooler (PgBouncer or SQLAlchemy's built-in pool) — do not open/close connections per request.
- PgBouncer in `transaction` mode is the most common production setup.

### Pool sizing

Rule of thumb: `connections = (CPU cores * 2) + effective_spindle_count`

- For most cloud DBs: 10-20 connections per application instance.
- Total connections across all instances must stay below PostgreSQL `max_connections`.
- Monitor `numbackends` vs `max_connections` — alert at 80%.

### Timeout configuration

| Setting | Purpose | Typical value |
|---------|---------|---------------|
| `connect_timeout` | Time to establish connection | 5s |
| `statement_timeout` | Max query execution time | 5s (OLTP), 5min (batch) |
| `idle_in_transaction_session_timeout` | Kill idle-in-transaction sessions | 60s |
| `lock_timeout` | Max time waiting for a lock | 5s |

Set these in the application connection string or per-session — do not rely on server defaults alone.

## Migrations: expand/contract by default

### Strategy

1. **Expand**
   - Add new nullable columns / tables.
   - Add indexes concurrently (`CREATE INDEX CONCURRENTLY`).
   - Add new code paths with backward-compatible reads.
2. **Backfill**
   - Run a throttled backfill with observability.
   - Ensure it is restartable/idempotent.
3. **Switch reads**
   - Move reads to the new schema/path.
4. **Contract**
   - Enforce constraints (`ALTER TABLE ... SET NOT NULL`, add `CHECK`).
   - Drop old columns/tables only after confidence period.

Always include:
- Rollback/backout plan.
- Lock/latency risk evaluation (especially for large tables).
- Migration runtime expectations.

### Dangerous DDL operations

| Operation | Risk | Safe alternative |
|-----------|------|-----------------|
| `CREATE INDEX` (without CONCURRENTLY) | Blocks all writes for duration | `CREATE INDEX CONCURRENTLY` |
| `ALTER TABLE ... SET NOT NULL` | Scans entire table, holds lock | Add CHECK constraint first: `ALTER TABLE t ADD CONSTRAINT chk CHECK (col IS NOT NULL) NOT VALID; ALTER TABLE t VALIDATE CONSTRAINT chk;` |
| `ALTER TABLE ... ALTER COLUMN TYPE` | Rewrites entire table | Add new column, backfill, switch reads, drop old |
| `ALTER TABLE ... ADD COLUMN ... DEFAULT` (PG < 11) | Rewrites entire table | PG 11+: instant for non-volatile defaults |
| `DROP COLUMN` | Takes ACCESS EXCLUSIVE lock briefly | Generally safe, but verify no views/functions depend on it |
| `RENAME COLUMN` | Breaks application code instantly | Add new column, dual-write, migrate reads, drop old |

### Migration pattern: adding a NOT NULL column to a large table

```sql
-- Step 1: Expand — add nullable column
ALTER TABLE orders ADD COLUMN region text;

-- Step 2: Backfill (in batches, application-side)
-- See Backfills section below

-- Step 3: Add NOT VALID check constraint (instant, no scan)
ALTER TABLE orders ADD CONSTRAINT chk_orders_region_nn
    CHECK (region IS NOT NULL) NOT VALID;

-- Step 4: Validate constraint (scans table but doesn't block writes)
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_region_nn;

-- Step 5: Optionally set NOT NULL (now instant because constraint is validated)
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT chk_orders_region_nn;
```

## Backfills

### Requirements

Backfills must be:
- **Bounded** — explicit range (ID range, date range).
- **Throttled** — rate limits; avoid saturating the primary DB.
- **Observable** — progress metrics/logs.
- **Resumable** — checkpointing or idempotent partition strategy.

### Batched backfill pattern

```python
import time
import psycopg

BATCH_SIZE = 1000
SLEEP_SECONDS = 0.5

def backfill_region(conn_info: str, start_id: int = 0) -> None:
    with psycopg.connect(conn_info) as conn:
        cursor_id = start_id
        total = 0

        while True:
            with conn.transaction():
                with conn.cursor() as cur:
                    cur.execute("""
                        UPDATE orders SET region = derive_region(user_id)
                        WHERE id > %s AND region IS NULL
                        ORDER BY id
                        LIMIT %s
                    """, (cursor_id, BATCH_SIZE))

                    updated = cur.rowcount
                    if updated == 0:
                        break

                    cur.execute(
                        "SELECT MAX(id) FROM orders WHERE id > %s ORDER BY id LIMIT %s",
                        (cursor_id, BATCH_SIZE),
                    )
                    cursor_id = cur.fetchone()[0]

            total += updated
            print(f"Backfilled {total} rows, cursor at id={cursor_id}")
            time.sleep(SLEEP_SECONDS)
```

Key points:
- Process in batches with `LIMIT` — never update the entire table in one transaction.
- Commit after each batch — allows checkpointing and reduces lock duration.
- Sleep between batches to reduce DB load.
- Track progress via cursor ID — resumable from any point.
- Use `WHERE region IS NULL` to make it idempotent — safe to re-run.

## Checklist

- [ ] Which invariants are affected? Are they enforced in constraints?
- [ ] Are primary keys chosen deliberately (bigint vs UUID vs natural)?
- [ ] Are indexes aligned with query patterns? Verified with EXPLAIN?
- [ ] Are transactions scoped correctly? No transaction across network calls?
- [ ] Are write operations idempotent under retries?
- [ ] Is the migration using expand/contract? Rollback plan documented?
- [ ] Are dangerous DDL operations handled safely (CONCURRENTLY, NOT VALID)?
- [ ] Are backfills batched, throttled, and resumable?
- [ ] Are query timeouts and connection pool settings configured?
- [ ] Are cursor-based pagination strategies used for list endpoints?

## Failure modes

- Big-bang migrations without expand/contract and rollback clarity.
- Retry-enabled non-idempotent writes causing duplicate/corrupt state.
- Long transactions across network calls increasing lock contention.
- Missing/incorrect indexes on hot paths causing latency regressions.
- Backfills without bounds/throttling saturating primary workloads.
- `CREATE INDEX` without CONCURRENTLY blocking writes on production tables.
- Check-then-act race conditions instead of atomic ON CONFLICT operations.
- OFFSET-based pagination causing O(n) scans on large tables.
- Missing `statement_timeout` allowing runaway queries to saturate DB.
- Pool exhaustion from leaked connections or missing idle timeout.
- `ALTER COLUMN TYPE` or `SET NOT NULL` on large tables without the NOT VALID trick.
- Deadlocks from inconsistent lock ordering across transactions.
