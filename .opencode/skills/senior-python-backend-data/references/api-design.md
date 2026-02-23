---
name: api-design
description: Stable API contract practices for production systems — request/response contracts, backward compatibility, idempotent writes, error models, pagination, timeouts, and observability. Use when designing, changing, or reviewing HTTP/RPC API contracts.
---

# API Design (HTTP/RPC) for Backend/Data Systems

This document defines stable API contract practices for production systems.

## When to use

- Designing or changing HTTP/RPC request/response contracts.
- Evolving error models, pagination, or compatibility policy.
- Implementing retry-safe write endpoints and idempotency semantics.
- Reviewing API changes for production rollout safety.

## Contracts (request/response)

- Validate inputs early; reject invalid data with clear errors.
- Define stable response models; avoid leaking internal structures.
- Use consistent field naming and types; document time zone semantics.
- Include correlation/request IDs if the platform supports it.

## Backward compatibility (default)

- Avoid breaking changes; if unavoidable, version explicitly.
- For additive changes:
  - add optional fields
  - tolerate unknown fields on input when safe
- Deprecate with a policy (announce, monitor usage, remove later).

## Idempotent writes

For endpoints that create/charge/trigger:
- Support idempotency keys.
- Document idempotency semantics:
  - what is considered “same request”
  - how long keys are retained
  - what happens on partial failure

## Errors

- Provide:
  - stable machine-readable error code
  - human-readable message
  - optional details (field errors) without leaking secrets
- Do not return stack traces to clients.
- Map internal exceptions to consistent error shapes.

## Pagination

- Prefer cursor-based pagination for large datasets.
- Enforce maximum page size.
- Avoid unbounded list endpoints in production paths.

## Timeouts and retries

- Servers should be safe under client retries (idempotency).
- Clients should implement bounded retries with backoff for transient errors.
- Avoid retry storms; rate-limit if necessary.

## Observability

- Log at the edge (request start/end) with:
  - correlation id
  - key identifiers (non-PII)
  - latency and status
- Emit metrics for rate/latency/error classes.

## Checklist

- What is the contract (input/output/error model)?
- Is it backward compatible?
- Is the write path idempotent?
- Are pagination and limits safe?
- Are logs/metrics sufficient to debug issues?

## Failure modes

- Breaking contract changes without explicit versioning.
- Retry-unsafe writes creating duplicate side effects.
- Leaking internal exception details or stack traces to clients.
- Unbounded list endpoints causing latency and resource spikes.
- Inconsistent error shapes that break clients and observability.
