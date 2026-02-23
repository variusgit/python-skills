# Python Testing Playbook (Backend + Data + Airflow)

This document defines practical, senior-level testing standards for production Python backend and data work.

## When to use

- Writing or refactoring backend/data Python code that changes behavior.
- Adding or changing API, DB, Airflow, or pipeline contracts.
- Stabilizing flaky tests or redesigning CI quality gates.
- Reviewing risky changes where correctness depends on invariants.

## Goals

- **Catch invariant violations early** (unit tests for domain logic + constraints).
- **Prevent regressions** with deterministic tests and clear failure messages.
- **Keep tests fast**: prefer unit tests; run integration tests intentionally.
- **Make refactors safe**: tests should validate behavior, not implementation details.

## The test pyramid (default)

1. **Unit tests (majority)**  
   Pure functions and domain logic; no external systems.
2. **Integration tests (some)**  
   DB queries, transactions, migrations, Airflow operator integration; run in CI with containers or ephemeral services.
3. **Contract tests (as needed)**  
   API payloads/error models, event schemas, backward compatibility, migrations with consumers.
4. **End-to-end tests (few, high value)**  
   Only for critical flows where integration risk is high.

## Determinism rules (non-negotiable)

- No real network calls in unit tests.
- Control time (freeze or inject a clock); do not depend on wall-clock.
- Control randomness (seed or avoid).
- Avoid shared mutable state between tests.
- Tests must pass when run in any order.

## Structure & naming

Recommended layout:

- `tests/unit/...`
- `tests/integration/...`
- `tests/contract/...` (optional)

Test names should describe behavior:
- `test_creating_order_is_idempotent_when_same_key_reused`
- `test_backfill_skips_already_materialized_partitions`

## Unit testing patterns

### What to unit test
- invariants (success + failure cases)
- state transitions
- idempotency behavior (same input twice → same persisted effect)
- boundary validation and normalization (types, time zones, precision)

### What not to unit test
- third-party libraries
- framework wiring (unless there's custom logic)
- one-off glue code with no branching (prefer integration tests)

### Fixtures
- Keep fixtures local to the test module when possible.
- Prefer **small, explicit fixtures** over “god fixtures”.
- Avoid fixtures with hidden side effects (disk/network/database).

### Mocking strategy
Mock at **system boundaries**:
- HTTP clients
- message brokers
- S3/storage clients
- external APIs

Avoid mocking pure domain functions; test them directly.

## Integration tests

Use integration tests when:
- SQL queries are non-trivial
- transaction boundaries matter
- uniqueness/locking/concurrency behavior matters
- migrations/backfills are risky

### DB integration testing (PostgreSQL)
- Use ephemeral Postgres (container) in CI.
- Apply migrations in test setup.
- Validate constraints and indexes for critical queries.
- Test both “happy path” and expected constraint violations.

## Contract tests

Add contract tests when:
- API response shape changes
- error codes or semantics change
- event schema changes (producer/consumer compatibility)

Contracts should be machine-checkable:
- JSON schema / OpenAPI snapshots
- schema versioning rules
- compatibility test suites

## Airflow testing (critical)

### 1) DAG import/parse tests (must-have)
**Rule:** DAG modules must be importable with **no I/O**.

Test:
- importing all DAG modules succeeds
- a `DagBag` loads with **zero import errors**
- `dagbag.dags` contains expected DAG IDs

### 2) DAG structure tests
Validate:
- schedule / dataset triggers / timezone
- default_args (retries/timeouts) are present where required
- task IDs are stable and descriptive
- dependencies and critical task ordering
- pools/queues/concurrency limits for expensive tasks

### 3) Task/business logic tests
Keep business logic **out of DAG files**:
- put logic in importable modules under `src/`
- unit test it separately
- in DAG tests, only validate wiring and config

### 4) Operator/Sensor tests (as needed)
For custom operators/sensors:
- unit test input validation and result interpretation
- integration test with a stubbed backend where meaningful

## Data pipeline testing (batch jobs)

When building data jobs (PySpark or Python batch):
- Add unit tests for transformations (small in-memory dataframes / samples).
- Add validation tests for **schema + constraints** (nullability, uniqueness, ranges).
- For incremental loads, test watermark logic and partition selection.
- For deduplication, test keys and tie-breaking rules.

## CI quality gates (recommended defaults)

Always run:
- `ruff` (lint + format)
- `basedpyright` (or configured type checker)
- `pytest -q` for unit tests

Additionally run (depending on repo):
- integration tests (DB / services) on main branch or per PR if fast enough
- DAG parse tests (always for DAG repos)
- contract checks for public APIs/events

## Checklist

- Unit tests cover core invariants and idempotency paths.
- Integration tests cover DB/transaction/migration risks where applicable.
- Contract tests protect API/event compatibility when boundaries change.
- DAG parse/structure tests are present for Airflow repositories.
- CI gates run lint, type-check, and deterministic pytest suites.

## Failure modes

- **Flaky tests**: isolate time/network; remove sleeps; control randomness.
- **Slow tests**: split integration from unit; cache heavy fixtures; shrink datasets.
- **Brittle tests**: assert behavior and contracts, not internal call order.
