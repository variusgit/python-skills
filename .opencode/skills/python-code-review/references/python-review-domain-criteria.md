---
name: python-review-domain-criteria
description: Domain-specific evaluation criteria for reviewing PostgreSQL persistence, API contracts, Airflow DAGs, data jobs/storage, and messaging code — concrete red flags, correct patterns, and severity guidance. Use during code review passes 2 (Correctness) and 8 (Code quality) when the change touches these domains.
---

# Domain-Specific Review Criteria

Concrete evaluation criteria for domain-specific code review. This document defines what "correct" looks like, what the red flags are, and how to classify findings by severity — for PostgreSQL, API contracts, Airflow, data jobs, and messaging code.

This is an **evaluation guide**, not a build guide. Check reviewed code against these criteria — do not prescribe specific implementations.

## When to use

- Reviewing changes that touch PostgreSQL schemas, queries, migrations, or transactions.
- Reviewing API contract changes (endpoints, payloads, errors, pagination).
- Reviewing Airflow DAG definitions, task wiring, or backfill logic.
- Reviewing data jobs (PySpark, batch processing) or storage layout changes.
- Reviewing messaging consumers, producers, or event-driven flows.

Load selectively — only the sections relevant to the change under review.

---

## PostgreSQL: schema and constraints

### Correct pattern characteristics

- `NOT NULL` is the default; columns are nullable only when null has explicit business meaning.
- `CHECK` constraints enforce value-domain invariants (status enums, positive amounts, valid ranges).
- `UNIQUE` constraints enforce business-key uniqueness and idempotency keys.
- `FOREIGN KEY` constraints enforce referential integrity between entities.
- Constraints are named explicitly (`CONSTRAINT uq_...`) for readable error messages.
- `text + CHECK` used for enum-like values — not `CREATE TYPE ... AS ENUM`.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Business invariant enforced only in application code, no DB constraint | **critical** | Bypassed by direct DB access, data loads, or bugs in other code paths |
| Nullable column without documented business reason for null | **major** | Ambiguous semantics — callers must guess whether null means "unknown", "not applicable", or "bug" |
| `CREATE TYPE ... AS ENUM` in production | **major** | Adding values requires DDL; removing values is not supported; migration tooling is fragile |
| Missing `UNIQUE` constraint on idempotency key | **blocker** | Retries will create duplicates |
| No `CHECK` on status/type column that has a finite set of valid values | **major** | Invalid states can be persisted |
| JSONB used to avoid schema design for regularly queried/constrained fields | **major** | Query performance degrades; constraints cannot be enforced |

---

## PostgreSQL: transactions and concurrency

### Correct pattern characteristics

- Transactions are short — computation done before, DB operations executed, committed quickly.
- No transaction held open across network calls (HTTP, gRPC, queue publish).
- Isolation level is chosen intentionally: READ COMMITTED (default), REPEATABLE READ (consistent multi-step reads), SERIALIZABLE (critical financial operations).
- `SELECT ... FOR UPDATE` used for read-modify-write on specific rows.
- Lock ordering is consistent across the codebase to avoid deadlocks.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Transaction held open across HTTP/gRPC/queue call | **critical** | Connection pool exhaustion, increased lock contention, cascading failures |
| No explicit isolation level for multi-step read-modify-write | **major** | Race conditions under concurrent access |
| Check-then-act pattern instead of atomic operation | **critical** | Race condition between check and write |
| No `statement_timeout` on production query paths | **critical** | Runaway queries saturate the database |
| Missing `lock_timeout` on DDL or locking operations | **major** | Blocks other transactions indefinitely |

### Anti-pattern recognition

```python
# RED FLAG: check-then-act race condition
existing = db.query("SELECT 1 FROM orders WHERE idempotency_key = %s", key)
if not existing:
    db.execute("INSERT INTO orders ...")  # two concurrent requests both pass the check

# CORRECT: atomic upsert
db.execute("""
    INSERT INTO orders (idempotency_key, ...) VALUES (%s, ...)
    ON CONFLICT (idempotency_key) DO NOTHING
""", key)
```

---

## PostgreSQL: migrations

### Correct pattern characteristics

- Expand/contract strategy: add nullable columns first, backfill, enforce constraints, drop old.
- `CREATE INDEX CONCURRENTLY` — never plain `CREATE INDEX` on production tables.
- `NOT NULL` added via NOT VALID CHECK constraint + VALIDATE, not direct `ALTER COLUMN SET NOT NULL` on large tables.
- Rollback plan documented for every migration.
- Backfills are batched, throttled, resumable, and observable.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| `CREATE INDEX` without `CONCURRENTLY` on a production table | **blocker** | Blocks all writes for duration of index creation |
| `ALTER TABLE ... SET NOT NULL` directly on a large table | **critical** | Full table scan holding ACCESS EXCLUSIVE lock |
| `ALTER TABLE ... ALTER COLUMN TYPE` without expand/contract | **critical** | Full table rewrite, extended downtime |
| `RENAME COLUMN` without dual-write migration | **critical** | Breaks application code instantly |
| Migration without rollback plan | **major** | No recovery path if migration causes issues |
| Backfill in a single transaction (no batching) | **critical** | Long lock hold, high memory usage, no resume on failure |
| Backfill without throttling (no sleep between batches) | **major** | Saturates production database |

### Anti-pattern recognition

```sql
-- RED FLAG: blocks all writes on production table
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- CORRECT: non-blocking index creation
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- RED FLAG: full table scan with exclusive lock on large table
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;

-- CORRECT: two-step approach
ALTER TABLE orders ADD CONSTRAINT chk_region_nn CHECK (region IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_region_nn;
```

---

## PostgreSQL: query patterns

### Correct pattern characteristics

- All list endpoints use cursor-based pagination (not OFFSET).
- Queries have `LIMIT` on production paths — no unbounded result sets.
- Batch inserts use multi-row VALUES or COPY — not single-row INSERT in a loop.
- Idempotent writes use `INSERT ... ON CONFLICT` — not check-then-act.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| OFFSET-based pagination on a large table | **major** | O(n) scan cost, degrades as pages increase |
| No LIMIT on a query in a user-facing path | **critical** | Unbounded result set causes memory/latency spikes |
| Single-row INSERT in a loop | **major** | Network round-trip per row; 100-1000x slower than batch |
| N+1 query pattern (query per item in a loop) | **major** | Linear growth in DB round-trips |
| Missing index on a hot-path filter/join column | **major** | Full table scan on every request |

---

## API contracts

### Correct pattern characteristics

- Request/response models are explicit (Pydantic, dataclass, or protobuf) — not raw dicts.
- Error responses have stable machine-readable code, human-readable message, optional details — no stack traces.
- Backward compatibility is the default: new fields are optional, unknown fields tolerated on input.
- Breaking changes are versioned explicitly (URL path or header).
- Write endpoints support idempotency keys where retries are possible.
- List endpoints have cursor-based pagination with enforced max page size.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Breaking API change without explicit versioning | **blocker** | Silently breaks existing clients |
| Internal stack trace or exception details returned to client | **critical** | Information disclosure, security risk |
| Inconsistent error shape across endpoints | **major** | Clients cannot reliably parse errors |
| List endpoint without pagination or LIMIT | **critical** | Unbounded response, latency spike, OOM risk |
| Write endpoint without idempotency key (where retries are possible) | **critical** | Duplicates on retry |
| Raw dict used as API response (no schema validation) | **major** | Contract drift — response shape changes silently |

### Anti-pattern recognition

```python
# RED FLAG: raw dict response, no schema, leaks internals
@router.post("/orders")
async def create_order(request: Request):
    data = await request.json()  # no input validation
    order = db.insert(data)
    return order.__dict__  # leaks internal fields, no contract

# CORRECT: explicit models, validated input/output
@router.post("/orders", response_model=OrderResponse, status_code=201)
async def create_order(payload: OrderCreate, session=Depends(get_db)):
    order = await order_service.create(session, payload)
    return order
```

---

## Airflow DAGs

### Correct pattern characteristics

- DAG files are thin — business logic in importable modules under `dags/libs/` or similar.
- No I/O at module import time (no network, DB, or S3 calls during DAG parsing).
- `default_args` include `retries` (>= 1) and `retry_delay`.
- Tasks have explicit `execution_timeout`.
- Pools/queues set for tasks that access scarce resources (DB, APIs, Spark clusters).
- Sensors use `mode='reschedule'` to free worker slots while waiting.
- Tasks are idempotent — safe to retry and backfill.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Business logic in DAG file (not in importable module) | **major** | Untestable, slows DAG parsing, mixes orchestration with logic |
| Network/DB/S3 call at module import time | **blocker** | Breaks DAG parsing for entire Airflow instance if call fails |
| No retries or `execution_timeout` on tasks | **major** | Transient failures kill the DAG run; zombie tasks hold slots indefinitely |
| Non-idempotent task with retries enabled | **blocker** | Retries create duplicates or corrupt data |
| Sensor with `mode='poke'` and long timeout | **major** | Worker slot occupied for hours doing nothing |
| `catchup=True` without understanding implications | **critical** | Accidental massive backfill on deploy |
| `depends_on_past=True` without justification | **major** | Creates bottleneck — one failed run blocks all subsequent |
| No pool/queue for tasks accessing scarce resources | **major** | Uncontrolled concurrency can overwhelm external systems |
| Hardcoded dates instead of Jinja macros (`{{ ds }}`) | **major** | Breaks backfill and idempotency |

### Anti-pattern recognition

```python
# RED FLAG: import-time I/O — breaks DAG parsing if DB is down
import psycopg
conn = psycopg.connect("postgresql://...")  # executes on import
TABLES = conn.execute("SELECT ...").fetchall()

# CORRECT: defer I/O to task execution
def get_tables():
    with psycopg.connect("postgresql://...") as conn:
        return conn.execute("SELECT ...").fetchall()

# RED FLAG: business logic in DAG file
@task()
def transform(data):
    # 50 lines of transformation logic here
    ...

# CORRECT: logic in importable module
from dags.libs.transforms import transform_customers

@task()
def transform(data):
    return transform_customers(data)
```

---

## Data jobs and storage

### Correct pattern characteristics

- Inputs/outputs are explicit: source path, schema, partition keys, output path, write mode.
- Schema is passed explicitly — never relies on schema inference in production.
- Transforms are pure and deterministic: no `processing_time` in partition keys, no non-deterministic functions in output.
- Dedup key and tie-break rule are defined when duplicates are possible.
- Validation at both boundaries: after read (input schema, nulls, constraints) and before write (row count, uniqueness, quality).
- Write pattern is idempotent: overwrite-by-partition (`partitionOverwriteMode=dynamic`) or staging+commit.
- Corrupt/invalid records are quarantined — never silently dropped.
- File sizes target 128 MB – 1 GB per Parquet file within a partition.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Append-without-dedup when retries are possible | **blocker** | Creates duplicate records on retry |
| No input or output validation | **critical** | Silent data corruption propagates to downstream |
| Schema inference in production job (`spark.read.parquet(path)` without `.schema()`) | **major** | Schema drift goes undetected, breaks consumers |
| Silently dropping corrupt records without quarantine | **critical** | Data loss goes undetected |
| Non-deterministic transform (using `current_timestamp()` in partition key) | **critical** | Reprocessing produces different output, breaks idempotency |
| Thousands of tiny files per partition (< 1 MB each) | **major** | High list/read cost, slow consumers |
| UDF where a built-in Spark function exists | **major** | Disables Catalyst optimizer, 10x slowdown |
| `repartition(1)` on a large dataset | **major** | Single-task bottleneck, potential OOM |
| Watermark/state saved before output write completes | **critical** | Failed write loses data on retry — watermark advanced past unwritten data |

### Anti-pattern recognition

```python
# RED FLAG: no schema, no validation, append on retry-enabled job
df = spark.read.parquet(input_path)  # schema inference
df.write.mode("append").parquet(output_path)  # duplicates on retry

# CORRECT: explicit schema, validation, idempotent write
df = spark.read.schema(EXPECTED_SCHEMA).parquet(input_path)
validate_input(df)
df.write.mode("overwrite").partitionBy("dt").parquet(output_path)
```

```python
# RED FLAG: silently drops bad records
df = df.filter(F.col("id").isNotNull())  # where do dropped records go?

# CORRECT: quarantine pattern
valid = df.filter(F.col("id").isNotNull())
quarantine = df.filter(F.col("id").isNull())
if quarantine.count() > 0:
    quarantine.write.mode("overwrite").parquet(quarantine_path)
    log.warning("Quarantined %d records", quarantine.count())
```

---

## Messaging and async processing

### Correct pattern characteristics

- Delivery semantics are explicit: at-least-once assumed unless proven otherwise.
- Consumer side effects are idempotent: dedup key defined, dedup state persisted when correctness requires it.
- Retry policy is bounded with backoff and jitter.
- Poison-message path is defined: max attempts → DLQ → manual remediation.
- Ordering requirements are scoped to a key (entity/aggregate ID), not assumed globally.
- Events are immutable facts (past tense: `OrderPlaced`); commands are requests (may fail).
- When DB consistency and event publishing must be atomic — outbox pattern is used.
- Event publishing happens after DB transaction commits, not inside it.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Non-idempotent consumer with at-least-once delivery | **blocker** | Duplicate processing on redelivery |
| Event published inside DB transaction (not outbox) | **critical** | Transaction rollback leaves published event orphaned; or failed publish leaves DB state without notification |
| No DLQ or poison-message handling | **critical** | Bad message blocks consumer indefinitely or retries forever |
| Unbounded retries on consumer without backoff | **critical** | Consumer retry storm, resource exhaustion |
| Global ordering assumed without partition-key scoping | **major** | Breaks silently under partition rebalance or scaling |
| Event schema changed without versioning | **critical** | Silently breaks consumers |
| Consumer does network call inside message handler without timeout | **major** | Slow dependency blocks message processing |

### Anti-pattern recognition

```python
# RED FLAG: DB write + event publish not atomic
def handle_order(order_data):
    db.execute("INSERT INTO orders ...")
    kafka_producer.send("order_created", order_data)  # fails → DB has order, no event
    # OR: event sent, then DB fails → event without order

# CORRECT: outbox pattern
def handle_order(order_data):
    with db.transaction():
        db.execute("INSERT INTO orders ...")
        db.execute("INSERT INTO outbox (event_type, payload) VALUES ...")
    # separate publisher reads outbox and sends to broker
```

---

## Severity quick reference (domain-specific)

### Blockers (must fix before merge)

- Missing UNIQUE constraint on idempotency key
- `CREATE INDEX` without CONCURRENTLY on production table
- Import-time I/O in Airflow DAG module
- Non-idempotent Airflow task with retries
- Append-without-dedup on retry-enabled data job
- Non-idempotent consumer with at-least-once delivery
- Breaking API change without versioning

### Critical (must fix before merge)

- Business invariant enforced only in app code, no DB constraint
- Transaction held across network calls
- Check-then-act race condition instead of atomic operation
- No statement_timeout on production queries
- Accidental catchup=True massive backfill
- No input/output validation in data jobs
- Silently dropping corrupt records
- Non-deterministic transform breaking idempotency
- Watermark saved before output write
- Event published inside transaction (no outbox)
- No DLQ / poison-message handling
- Event schema change without versioning
- Backfill in single transaction without batching

### Major (should fix before merge)

- Nullable column without documented reason
- PostgreSQL ENUM type instead of text+CHECK
- OFFSET pagination on large tables
- N+1 queries, missing indexes
- Business logic in DAG file
- No retries/timeouts on Airflow tasks
- Schema inference in production Spark job
- UDF where built-in function exists
- No pool/queue for resource-intensive Airflow tasks
- Global ordering assumption in messaging

### Minor (fix or acknowledge)

- Unnamed constraints
- JSONB for semi-structured data without GIN index
- Sensor with poke mode on short timeout
