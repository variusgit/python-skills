---
name: python-production-readiness-criteria
description: Evaluation criteria for production readiness — concrete red flags, correct patterns, and severity guidance for retry/resilience, observability, error handling, degradation, and operational controls. Use during code review passes 5 (Reliability), 6 (Observability), and operational aspects of pass 8.
---

# Production Readiness — Evaluation Criteria

Concrete criteria for evaluating production readiness during code review. This document defines what "correct" looks like, what the red flags are, and how to classify findings by severity.

This is an **evaluation guide**, not a build guide. Check reviewed code against these criteria — do not prescribe specific implementations.

For domain-specific patterns the builder is expected to follow, load the relevant builder reference (see code-review SKILL.md reference routing).

## When to use

- Reviewing any code that will run in production.
- Evaluating reliability, observability, or operational readiness (passes 5, 6, 8).
- Classifying production-readiness findings by severity.

## Retry and resilience

### Correct pattern characteristics

- **Bounded stop** — explicit max attempts (not infinite loops).
- **Exponential backoff with jitter** — prevents thundering herd under load.
- **Retry only transient errors** — ConnectionError, TimeoutError, 5xx. Never 4xx client errors.
- **Reraise after exhaustion** — caller must know the operation failed. Silent absorption is a bug.
- **Target operation is idempotent** — prerequisite for any retry. No idempotency = no retry.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Retry + non-idempotent write (no dedup key/constraint) | **blocker** | Creates duplicates or data corruption |
| Unbounded retries (no max attempts) | **critical** | Retry storm, resource exhaustion |
| Missing reraise — failure silently absorbed | **critical** | Caller assumes success, data inconsistency |
| Retrying client errors (400, 401, 403, 422) | **major** | Will never succeed, wastes resources |
| Fixed delay without jitter | **minor** | Thundering herd under concurrent load |

### Anti-pattern recognition

```python
# RED FLAG: unbounded retry, catches everything, no jitter
while True:
    try:
        result = call_api()
        break
    except Exception:
        time.sleep(1)

# RED FLAG: retry on non-idempotent operation
@retry(stop=stop_after_attempt(3))
def create_order(data):
    db.execute("INSERT INTO orders ...")  # no ON CONFLICT, no idempotency key
```

## Observability — logging

### What to verify

- Structured logging configured (JSON or key=value format, not plain text).
- Correlation/request ID propagated through the call chain (contextvars or middleware).
- Boundary events logged: operation start, end (with duration_ms and status), external call failures.
- Log levels match severity: ERROR for failures needing investigation, WARNING for recoverable issues, INFO for normal operation flow.
- No secrets, tokens, passwords, or PII in log output.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| PII or secrets in log output | **blocker** | Compliance violation, security breach |
| `print()` in production code | **major** | Unstructured, no levels, no routing |
| No correlation ID in service code | **major** | Cross-request incident triage becomes impossible |
| `except Exception: logger.warning(...)` without re-raise | **critical** | "Log and continue" masks invariant violations |
| All log statements at INFO level | **minor** | Cannot filter by severity, alert fatigue |
| Error logs without context (entity ID, operation, what failed) | **major** | Cannot debug without reproducing |

## Observability — metrics

### What to verify for services

- **RED signals** present: Request rate, Error rate, Duration (histogram with meaningful buckets).
- Saturation signals: DB connection pool usage, worker queue depth.
- Label cardinality bounded — no user IDs, session IDs, or full URL paths as label values.

### What to verify for pipelines and data jobs

- Job/task **duration** tracked.
- **Row counts**: input, output, quarantine/rejected.
- **Freshness**: max event_time in output vs wall clock (detects stale data).
- **Completeness**: expected partitions vs actually written.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Unbounded label values (user_id, full URL as metric label) | **critical** | Monitoring system cardinality explosion |
| New service endpoint with no metrics | **major** | Invisible to monitoring, no SLO baseline |
| Data pipeline with no freshness/completeness signal | **major** | Silent data staleness goes undetected |
| Duration metric without appropriate buckets | **minor** | Cannot analyze latency distribution |

## Error handling

### What to verify

- Specific exception types caught — not bare `except` or blanket `except Exception`.
- Errors propagated with context (exception chaining, structured error info).
- Failure modes are explicit: what happens when dependency X is unavailable?
- Partial failure scenarios handled in batch/fan-out operations.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| `except: pass` or `except Exception: pass` | **blocker** | Silently hides data loss and corruption |
| `except Exception: log.warning(...)` without re-raise or explicit recovery | **critical** | "Log and continue" masks broken invariants |
| No timeout on outgoing network call (HTTP, gRPC, S3, DB) | **critical** | Cascading failure, connection pool exhaustion |
| Error message without identifying context (which entity, which operation) | **major** | Cannot debug without reproducing |
| Leaking internal stack traces or error details to API clients | **major** | Information disclosure |

## Circuit breaker and dependency protection

### When to require

- External HTTP/gRPC dependency on a critical path with known availability risk.
- Any call path where prolonged dependency failure would exhaust connection pools or block workers.

### When NOT to require

- **PostgreSQL** — connection pool limits + `statement_timeout` are the correct protection.
- Internal calls within the same process.
- Async operations with DLQ/retry queue (already decoupled).

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| External dependency call with no timeout AND no circuit breaker | **critical** | Cascading failure risk |
| Custom circuit breaker without thread/async safety | **major** | Broken under concurrency, false sense of safety |
| Circuit breaker on PostgreSQL instead of pool limits + timeouts | **minor** | Wrong tool for the problem |

## Graceful degradation

### What to verify

- Feature flags or kill switches for risky new behavior (runtime-switchable, not restart-required).
- Fallback strategy is appropriate for dependency criticality:

| Dependency role | Expected strategy |
|-----------------|-------------------|
| Required for correctness (payments, auth) | Fail fast, reject request |
| Required for completeness (enrichment, notifications) | Queue for retry, return partial response |
| Optional / enhancement (recommendations, analytics) | Return default, skip feature |

- Degradation behavior is tested or at minimum documented.

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Risky new behavior with no kill switch | **major** | Cannot disable without rollback/deploy |
| No fallback for dependency with known availability issues | **major** | Full outage on partial dependency failure |
| Stale feature flags (months old, never removed) | **minor** | Dead code branches, confusion |
| Feature flag without clear owner or removal plan | **minor** | Accumulates tech debt |

## Operational readiness

### What to verify

- Pipeline tasks are idempotent if retries are configured.
- Backfill/re-run paths are documented and safe.
- Alerting signals exist for new behavior or changed failure thresholds.
- Runbook notes present for critical flow changes (failure modes, how to verify, how to rollback).

### Red flags

| Finding | Severity | Why |
|---------|----------|-----|
| Non-idempotent pipeline task with retries enabled | **blocker** | Retries will create duplicates/corruption |
| Critical flow change with no operational notes | **major** | On-call has no guidance during incidents |
| New scheduled job with no failure alerting | **major** | Failures go unnoticed |
| Backfill without throttling or isolation strategy | **major** | Can saturate production workloads |

## Severity quick reference

### Blockers (must fix before merge)

- Retry + non-idempotent write without dedup
- PII or secrets in log output
- Bare `except: pass` hiding failures
- Non-idempotent pipeline task + retries

### Critical (must fix before merge)

- Unbounded retries (retry storm risk)
- Missing reraise after retry exhaustion
- No timeout on external network call
- "Log and continue" masking invariant violations
- Unbounded metric label cardinality

### Major (should fix before merge)

- No correlation ID in service code
- No metrics on new endpoint
- No freshness/completeness signal on data pipeline
- Risky behavior without kill switch
- Critical flow without operational notes
- Error messages without identifying context
- Retrying non-transient errors (4xx)

### Minor (fix or acknowledge)

- Fixed retry delay without jitter
- Wrong log levels
- Stale feature flags
- Duration metric without appropriate buckets
