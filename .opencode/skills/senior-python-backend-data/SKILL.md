---
name: senior-python-backend-data
description: Ship and review production Python backend + data workflow changes (APIs, PostgreSQL, Airflow, Spark/S3) with senior-level correctness and operability. Use for backend features/refactors, API contract changes, schema migrations, Airflow DAGs/backfills, data jobs/storage layouts, or production incident triage.
compatibility: opencode
metadata:
  audience: "senior"
  language: "python"
  domains: "backend,data-engineering,orchestration"
  primary_tools: "python,airflow,sql,ruff,basedpyright,pyspark"
---

# Senior Python Backend/Data (Production Delivery Skill)

This is a **task skill**: a reusable set of instructions that tells an agent *how to execute* senior-level Python backend/data work in a production ecosystem.

## What this skill does

- Designs and implements production-grade Python backend and data systems (services, batch jobs, pipelines).
- Enforces architecture boundaries (domain logic vs I/O vs orchestration) to keep code testable and maintainable.
- Builds and operates Airflow workflows (DAG semantics, retries/timeouts, safe backfills, operational controls).
- Evolves persistence safely (constraints, transactions, expand/contract migrations).
- Hardens for production (observability, security/PII, performance and reliability hygiene).

## When to use this skill

Use this skill when the task involves any of the following:

- Writing/refactoring **Python backend** code that runs in production (APIs, workers, scheduled jobs)
- Designing or changing **API contracts** (payloads, errors, pagination, idempotency semantics)
- **Database** schema/query work (PostgreSQL), migrations, backfills, data integrity concerns
- Authoring/reviewing **Airflow DAGs**, tasks, sensors, schedules, or backfill/reprocessing plans
- Building/refactoring **data jobs** (e.g., PySpark) and **storage layouts** (S3/Parquet partitions)
- Troubleshooting **production incidents** in backend/data workloads (timeouts, retries, data gaps, bad backfills)

Do not use this skill for front-end-only changes or generic brainstorming unrelated to production backend/data delivery.

## Fixed foundations (given stack constraints)

Assume these are in place and prefer integrating with them rather than replacing them:

- Python 3.10+
- Apache Airflow 2.10+
- PostgreSQL (primary relational DB)
- Greenplum **or** ClickHouse (MPP/analytics)
- PySpark 3.3+ (large-scale data processing)
- S3-compatible object storage (canonical object/OLAP layer)

If you propose additional tools (FastAPI, SQLAlchemy, Redis, Kafka, etc.), treat them as **add-ons** and explain integration and operational impact.

## Scope of responsibility (what you own per task)

For any change you implement or review, you own:

- Correctness of business rules in the code you touch
- Explicit contracts at boundaries (HTTP, DB, queues, storage)
- Safe persistence evolution (schemas, migrations, backfills when needed)
- Airflow DAG/task correctness with idempotency and backfill safety
- Production operability for shipped behavior (logs/metrics, runbook notes for critical flows)

You must surface:
- invariants affected by the change
- key trade-offs (latency/cost/complexity/operability)
- validation plan (tests + runtime signals)

## Operating principles (senior bar)

- **Correctness over convenience**: assume retries, duplicates, partial failures, concurrency, and race conditions.
- **Explicit contracts**: APIs, schemas, events are contracts; default to backward compatibility.
- **Invariants are first-class**: state them explicitly; enforce them in code + storage; test them.
- **Idempotency by default** for retryable/replayable operations (API writes, consumers, Airflow tasks, backfills).
- **Operability is part of done**: actionable logs/metrics, bounded retries/timeouts, safe re-run paths.
- **Incremental delivery**: small reversible changes; expand/contract migrations; feature flags when appropriate.

## How to work (agent workflow)

1. **Classify the task**
   - API / DB & migrations / Airflow / data job & storage / messaging async
2. **Identify invariants and risks**
   - data loss/corruption, downtime, backfill blast radius, security/PII, cost/performance regression
3. **Load only the relevant reference docs** (see “Reference routing”)
4. Produce an **implementation-grounded plan**
   - concrete steps, failure modes, validation, rollback/backout
   - ask only minimal clarifying questions needed to proceed safely
5. Implement with the “Definition of done” checklist below
6. **Verify after every change** (non-negotiable)
   - Run `ruff check` and `ruff format --check`; fix all violations before proceeding
   - Run `basedpyright`; resolve all type errors before proceeding
   - Run `pytest -q`; all existing tests must pass before proceeding
   - If tests fail, do not author or modify tests in this skill; delegate all test creation/modification to the testing agent
   - Do not move to the next step if any check fails
7. Close the loop
   - migration/backfill steps documented, mini-ADR for non-trivial decisions

## Output contract (default response shape)

When responding, prefer this structure (omit irrelevant sections):

- **Assumptions & constraints**
- **Plan**
- **Implementation details** (code-level notes and key decisions)
- **Rollout / migration / backfill safety**
- **Observability & ops notes**
- **Risks & mitigations**

## Definition of done (must satisfy)

- **Correctness**: invariants enforced; idempotent semantics where needed; explicit failure modes.
- **Maintainability**: clear module boundaries; readable naming; minimal “magic”.
- **Testability**: domain logic is separable and injectable; integration boundaries are explicit.
- **Observability**: structured logs; actionable errors; key metrics where meaningful.
- **Security**: secrets/PII protected; least privilege; safe input handling.
- **Operations**: timeouts/retries configured; backfill safety controls; runbook notes for critical flows.
- **Verified**: `ruff`, `basedpyright`, and `pytest` pass with zero errors after every change.

## Mini-ADR trigger (decision record)

Create a mini-ADR when any of these are true:
- new external dependency or integration
- schema migration with rollback considerations
- idempotency/replay/backfill semantics are changed
- significant performance/cost trade-off
- compatibility policy changes for APIs/events

Use:
- @.opencode/skills/senior-python-backend-data/references/python-adr-template.md

## Reference routing (progressive disclosure)

This skill stays high-level on purpose. Load deep technical detail **only when relevant**.

### Mandatory first read (always)
- @.opencode/skills/senior-python-backend-data/references/python-best-practices.md

Read `./references/python-best-practices.md` first for every backend/data task. Then load only the most relevant task-specific reference(s).

### Airflow DAG authoring / schedules / backfills / sensors
- @.opencode/skills/senior-python-backend-data/references/python-airflow-patterns.md

### PostgreSQL persistence (constraints, transactions, migrations, query hygiene)
- @.opencode/skills/senior-python-backend-data/references/python-postgresql.md

### Services & API design (FastAPI, aiohttp, gRPC, contracts, microservice patterns)
- @.opencode/skills/senior-python-backend-data/references/python-services-api.md

### Data jobs & storage (S3 layout, Parquet partitions, incremental loads, PySpark hygiene)
- @.opencode/skills/senior-python-backend-data/references/python-data-jobs.md

### Messaging / async processing (at-least-once, dedup, outbox, DLQ)
- @.opencode/skills/senior-python-backend-data/references/python-messaging.md

### MPP / analytics (Greenplum or ClickHouse)
- @.opencode/skills/senior-python-backend-data/references/python-analytics-mpp.md

**Conflict rule:** treat references as source of truth. If two references conflict, prefer the most task-specific doc; otherwise prefer `python-best-practices.md`. If the conflict impacts correctness/safety, call it out and choose the safer default.

## Anti-patterns (avoid)

- Business logic embedded in HTTP handlers or Airflow DAG files (especially at import/parse time).
- Bare `except`, swallowed failures, or “log and continue” that violates invariants.
- Non-idempotent tasks/handlers paired with retries (creates duplicates/corruption).
- Big-bang migrations without expand/contract and without a rollback plan.
- Unbounded list endpoints or unbounded DB queries in production paths.
- Backfills without throttling, isolation, and validation.
- Logging secrets/PII or leaking internal stack traces to clients.
