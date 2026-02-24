---
name: python-production-readiness
description: Baseline production hardening for backend/data workloads — structured logging, metrics, timeouts/retries/backoff, operational controls, runbook notes, security, and performance hygiene. Use when preparing changes for production rollout or hardening existing systems.
---

# Production Readiness (Observability, Reliability, Security)

This document defines baseline production hardening for backend/data workloads.

## When to use

- Preparing backend/data changes for production rollout.
- Defining observability and operational controls for critical paths.
- Hardening retry/timeout/concurrency strategy.
- Writing incident/runbook guidance and readiness checks.

Read `python-best-practices.md` first, then apply this document for production hardening and operations.

## Logging

- Prefer structured logs with stable fields.
- Include correlation/request IDs if available.
- Log key identifiers (non-PII) needed to debug.
- Never log secrets; redact or hash sensitive values where needed.

## Metrics (minimum set)

Backend services:
- request rate, latency, error rate (by class)
- saturation signals (DB pool, worker queue depth)

Data pipelines:
- success/failure rate
- runtime duration
- data freshness (lag) and completeness signals
- backlog/queue lag for async systems

## Timeouts, retries, backoff

- Always set timeouts on network calls.
- Retries must be bounded and paired with idempotency.
- Use exponential backoff + jitter for transient failures.
- Avoid retry storms: apply bulkheads/circuit breakers where appropriate.

## Operational controls

- Rate limit expensive endpoints.
- Use pools/queues/concurrency limits for batch and Airflow work.
- Provide safe re-run/backfill guidance for pipelines.

## Runbook notes (for critical paths)

For critical flows, document:
- common failure modes
- how to safely re-run/backfill
- how to verify correctness post-incident
- key dashboards/alerts to check

## Security & data protection

- Least privilege for DB/service accounts.
- Secrets never committed; use secret managers/env injection.
- Protect PII: minimize, redact logs, define retention expectations.
- Keep dependencies patched; respond to CVEs.

## Performance hygiene

- Avoid unbounded DB queries and unbounded list endpoints.
- Use indexes and query plans; monitor slow queries.
- Manage connection pools and avoid saturation.

## Checklist

- Structured logs include stable identifiers without secrets/PII.
- Service and pipeline metrics cover rate/latency/errors/freshness.
- Network calls have timeouts and bounded retry/backoff strategy.
- Concurrency controls exist for expensive endpoints and jobs.
- Critical path runbook covers rerun/backfill and verification steps.

## Failure modes

- Insufficient telemetry makes incidents hard to triage.
- Retry storms from unbounded retries without circuit protection.
- Secrets/PII leakage via logs or misconfigured access.
- Resource saturation from missing limits on jobs/endpoints.
- Slow-query and pool exhaustion issues without alerting signals.
