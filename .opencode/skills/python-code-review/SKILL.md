---
name: python-code-review
description: Senior-level Python code review for backend and data systems — systematic review of correctness, contracts, security, performance, reliability, observability, test adequacy, migration safety, and code quality. Use when reviewing PRs, pre-merge checks, or self-reviewing changes before shipping.
compatibility: opencode
---

# Python Code Review (Senior-Level)

This is a **task skill** for a dedicated code review agent. It defines *how to systematically review* Python backend and data code changes with senior-level rigor — catching correctness bugs, contract violations, security issues, and operability gaps before they reach production.

## What this skill does

- Reviews code changes (PRs, diffs, patches) through a structured multi-pass process.
- Evaluates correctness, safety, and production-readiness — not just style.
- Identifies risks that automated tools (linters, type checkers) cannot catch.
- Produces actionable findings with clear severity, location, and recommendation.

## When to use

- Reviewing a pull request or code diff.
- Self-reviewing changes before merging.
- Auditing existing code for production-readiness.
- Evaluating architectural changes or refactors for risk.

Do not use this skill for writing code, designing systems, or writing tests — it is strictly an evaluation tool.

## Review mindset

- **Think in invariants**: what must always be true? Does this change preserve or break it?
- **Think in failure modes**: what happens when this fails? Retries? Partial writes? Timeouts?
- **Think in contracts**: does this change what callers/consumers expect? Is it backward-compatible?
- **Think in blast radius**: if this is wrong, what breaks? How fast do we detect it? Can we roll back?
- **Substance over style**: focus on correctness, safety, and clarity — not formatting preferences.

## Review workflow

Review in this order. Each pass has a specific focus — do not mix concerns across passes.

### Pass 1: Understand the change

Before evaluating anything:

1. Read the full diff. Understand what changed and why.
2. Identify the scope: which modules, layers, and boundaries are touched.
3. Determine the change type: new feature / refactor / bugfix / migration / config change / dependency update.
4. Note what is NOT changed but might be affected (callers, consumers, downstream).

Do not start evaluating until you understand the intent and scope. If intent is unclear or a pattern looks deliberately unconventional, ask the author rather than assuming it's a mistake.

5. Calibrate review depth to scope:
   - **Standard** (single-service, straightforward change): focus passes 2-4 and 7-8. Evaluate passes 5-6 proportionally — don't demand circuit breakers, correlation IDs, or runbook notes for a simple endpoint.
   - **Complex** (distributed, multi-service, migration, high-risk data): all passes at full depth.

### Pass 2: Correctness and contracts

The highest-priority pass. Bugs here cause incidents.

**Invariants:**
- Are existing invariants preserved? Are new ones explicitly stated?
- Are edge cases handled: empty inputs, None/null, boundary values, concurrent access?
- Are state transitions valid? Can the system reach an invalid state?

**Contracts (APIs, schemas, events):**
- Are request/response contracts stable? Any breaking changes?
- Are error codes/shapes consistent with existing patterns?
- If contracts change: is backward compatibility maintained? Is versioning explicit?
- For event schemas: are producers and consumers compatible after this change?

**Idempotency:**
- Are write operations safe under retries? (API handlers, consumers, Airflow tasks, batch jobs)
- Is there an idempotency key? Is it enforced (DB constraint, dedup logic)?
- "Retry + non-idempotent write" = correctness bug. Flag it.

**Data integrity:**
- Are DB constraints correct (NOT NULL, UNIQUE, CHECK, FK)?
- Are transactions scoped correctly? No transaction held across network calls?
- For concurrent access: is the isolation level intentional? Are locks used where needed?
- For migrations: expand/contract strategy? Rollback plan? Lock/latency risk on large tables?

### Pass 3: Security and data protection

**Input handling:**
- Are all external inputs validated before use?
- SQL injection, command injection, path traversal — are they prevented?
- Are deserialization inputs from untrusted sources handled safely?

**Secrets and PII:**
- No secrets (API keys, passwords, tokens) hardcoded or committed.
- No PII in logs, error messages, or API responses.
- Secrets loaded from env vars or secret managers, never from code.

**Access control:**
- Least privilege applied? (DB accounts, API scopes, file permissions)
- Authentication/authorization checks present where required?
- No sensitive data exposed in health endpoints or debug outputs.

### Pass 4: Performance and resource usage

**Queries and data access:**
- No unbounded queries (missing LIMIT, no pagination).
- No N+1 query patterns.
- Indexes exist for hot-path query patterns (verify with EXPLAIN if critical).
- Large scans offloaded to analytics store, not the transactional DB.

**Memory and compute:**
- No unbounded data loading into memory (use streaming/chunking for large datasets).
- No blocking calls in async code paths.
- Time/space complexity is appropriate for expected data volume.

**External calls:**
- Timeouts set on all outgoing HTTP/gRPC/DB calls.
- Connection pools sized and bounded.
- No resource leaks (unclosed connections, file handles, sessions).

### Pass 5: Reliability and error handling

**Error handling:**
- No bare `except` or silently swallowed exceptions.
- Specific exceptions caught; errors propagated with context.
- Failure modes are explicit — what happens when X fails?

**Retries and resilience:**
- Retries are bounded with backoff and jitter.
- Retry targets are idempotent (see Pass 2).
- Circuit breaker or bulkhead for critical external dependencies.
- No retry storms (unbounded retries without backoff).

**Graceful degradation:**
- What happens if a dependency is unavailable? Crash? Degrade? Queue?
- Are timeouts appropriate? (not too short, not infinite)
- Partial failures handled explicitly (batch processing, fan-out operations).

### Pass 6: Observability and operability

**Logging:**
- Structured logs with stable fields (correlation ID, entity ID, operation).
- Appropriate log levels (not everything is INFO).
- No secrets/PII in log output.
- Error logs include enough context to debug without reproducing.

**Metrics and alerting:**
- Key signals covered: request rate, latency, error rate, saturation.
- For data pipelines: freshness, completeness, duration, backlog.
- Are there alerting signals for the new behavior?

**Operational controls:**
- Backfill/rerun paths are safe and documented for pipelines.
- Feature flags or kill switches for risky new behavior.
- Runbook notes for critical flows (how to verify, how to rollback).

### Pass 7: Test adequacy

Evaluate whether the tests are sufficient for the change — do not write tests.

**Coverage of new behavior:**
- Are new invariants and business rules tested?
- Are edge cases and failure paths tested (not just happy path)?
- For idempotent operations: is "same input twice" tested?

**Test quality:**
- Tests verify behavior and contracts, not implementation details.
- No shared mutable state between tests.
- No real network calls in unit tests; time/randomness controlled.
- Test names describe the behavior being verified.

**Missing tests (flag explicitly):**
- New code paths without any test coverage.
- Changed error handling without error-path tests.
- Migration/backfill logic without validation tests.
- Contract changes without contract/compatibility tests.

### Pass 8: Code quality and maintainability

**Structure and boundaries:**
- Business logic separated from I/O, handlers, and orchestration.
- No framework objects leaking into domain logic.
- I/O dependencies injectable (testable without mocks on internals).

**Readability:**
- Naming is clear and consistent with codebase conventions.
- No unnecessary complexity or over-abstraction.
- No dead code, commented-out code, or debug artifacts.
- Functions are focused (single responsibility, reasonable length).

**Dependencies:**
- New dependencies justified? Lightweight alternatives considered?
- Dependencies added via the repo's established toolchain (not manual edits).
- No version conflicts or security vulnerabilities in new deps.

**Airflow-specific (if applicable):**
- DAG files are thin — business logic in importable modules, not in DAG definitions.
- No I/O at module import time (DAG parsing must be side-effect free).
- Retries/timeouts set; pools/queues for expensive tasks.
- Backfill safety: idempotent tasks, bounded concurrency, observable progress.

**Data jobs (if applicable):**
- Inputs/outputs explicit (paths, schemas, partitioning).
- Transforms are deterministic; dedup keys and tie-break rules defined.
- Output validation: schema checks, row-count sanity, constraint checks.
- Partition strategy supports replay and incremental loads.

## Severity classification

Classify every finding:

| Severity | Meaning | Action |
|----------|---------|--------|
| **blocker** | Will cause data loss, security breach, or production incident | Must fix before merge |
| **critical** | Correctness bug, contract violation, or missing error handling that will fail in production | Must fix before merge |
| **major** | Significant risk: missing tests for new invariants, performance regression, operability gap | Should fix before merge |
| **minor** | Code quality, naming, minor improvements | Fix or acknowledge |
| **nit** | Style preference, optional improvement | Author's discretion |

**Rule**: never block a PR on nits alone. Blockers and criticals must be resolved. Majors should be resolved or explicitly accepted with justification.

## Review output format

Structure findings as:

```
### [severity] Short description

**Location**: `file.py:L42` (or function/class name)
**Issue**: What is wrong and why it matters.
**Recommendation**: Concrete fix or approach.
```

Group findings by severity (blockers first). End with a summary:

```
## Summary

- **Blockers**: N
- **Critical**: N
- **Major**: N
- **Minor/Nit**: N

**Verdict**: [Approve / Request changes / Needs discussion]

**Key risks**: (1-3 sentences on the most important concerns)
```

## Review anti-patterns (avoid)

- **Rubber-stamping**: approving without understanding the change.
- **Nit-picking without substance**: blocking on style when there are correctness concerns.
- **Rewriting in review**: suggesting a complete rewrite instead of actionable incremental feedback.
- **Scope creep**: demanding unrelated improvements in the same PR.
- **Assuming tests pass = correct**: tests validate what they test; review validates what they don't.
- **Ignoring what's NOT in the diff**: callers, consumers, and downstream effects of the change.
- **Style wars**: arguing formatting when linters handle it.
- **Disproportionate rigor**: demanding distributed-system patterns (circuit breakers, saga orchestration, correlation IDs, graceful shutdown) for simple single-service changes. Calibrate review depth to system complexity.

## Reference routing (progressive disclosure)

Load references to deepen evaluation accuracy. Do not load all at once — pick based on what the change touches.

### Production readiness criteria (load for any production-bound review)
- @.opencode/skills/python-code-review/references/python-production-readiness-criteria.md

Concrete red flags, correct pattern characteristics, and severity guidance for passes 5 (Reliability), 6 (Observability), and operational aspects of pass 8.

### Domain-specific review criteria (load when change touches DB, Airflow, APIs, data jobs, or messaging)
- @.opencode/skills/python-code-review/references/python-review-domain-criteria.md

Concrete red flags, correct vs incorrect patterns, and severity guidance for passes 2 (Correctness) and domain-specific aspects of pass 8 — PostgreSQL persistence, API contracts, Airflow DAGs, data jobs/storage, and messaging.

## Checklist (final gate)

Before submitting the review, verify:

- [ ] I understood the intent and full scope of the change.
- [ ] I checked correctness: invariants, edge cases, idempotency, data integrity.
- [ ] I checked contracts: backward compatibility, error shapes, schema changes.
- [ ] I checked security: input validation, secrets, PII, access control.
- [ ] I checked performance: unbounded queries, resource leaks, timeouts.
- [ ] I checked reliability: error handling, retry safety, failure modes.
- [ ] I checked observability: logs, metrics, operational controls.
- [ ] I evaluated test adequacy for the change.
- [ ] I checked code quality: boundaries, readability, dependencies.
- [ ] Every finding has a severity and a concrete recommendation.
- [ ] My review is actionable — the author knows exactly what to fix.

## Failure modes (of bad reviews)

- **Missed invariant violation** — approved a change that breaks a business rule under edge conditions.
- **Missed contract break** — approved a breaking API/schema/event change without versioning.
- **Missed security issue** — approved code with hardcoded secrets, SQL injection, or PII logging.
- **Missed resource leak** — approved code with unclosed connections or unbounded memory growth.
- **Over-focused on style** — spent review budget on naming nits while missing a concurrency bug.
- **No severity assessment** — listed findings without indicating which ones block the merge.
- **Non-actionable feedback** — "this doesn't look right" without explaining what's wrong and how to fix it.
