---
name: python-code-audit
description: Methodology for auditing and refactoring existing Python codebases — assessment approach, quality dimensions, severity classification, tech debt identification, refactoring strategies, and communication of findings. Use when evaluating code quality, planning refactors, or assessing technical debt in projects not originally written by this agent.
---

# Code Audit & Refactoring Methodology

## When to use

- Evaluating quality of an existing codebase (inherited, legacy, or third-party)
- Planning a refactoring effort and need to prioritize what to fix first
- Assessing technical debt and communicating it to stakeholders
- Preparing a codebase for a major change (new feature, migration, scaling)

## Audit approach

Follow four phases in order. Resist the urge to fix things during the assessment phase.

### Phase 1 — Understand context

Before judging code, understand why it exists in its current form:

- **Purpose**: what does this system do, who uses it, what are the SLAs?
- **Constraints**: team size, timeline pressure, runtime limitations, legacy dependencies
- **History**: was this a prototype that grew, a migration from another language, a contractor deliverable?
- **Scale**: request volume, data volume, deployment frequency, team familiarity

Context determines whether a pattern is "wrong" or "a reasonable trade-off given constraints."

### Phase 2 — Assess systematically

Evaluate each quality dimension (see below). Do not cherry-pick — cover all dimensions even if briefly. Note both strengths and weaknesses.

### Phase 3 — Prioritize findings

Classify each finding by severity (see below). Group related findings into themes (e.g., "error handling is systematically weak" rather than listing 20 individual bare excepts).

### Phase 4 — Plan remediation

For each theme, propose a remediation direction: what to fix, in what order, with what approach (incremental vs batch). Identify dependencies between fixes.

## Quality dimensions

Assess the codebase across these dimensions. Each maps to principles in `python-best-practices.md` — the audit checks whether those principles are applied in practice.

| Dimension | What to look for |
|---|---|
| **Readability** | Naming clarity, function size, nesting depth, comment quality, consistency of style |
| **Type safety** | Coverage of type annotations, `Any` usage, Pydantic vs raw dicts at boundaries, basedpyright compatibility |
| **Architecture boundaries** | Separation of domain / I/O / orchestration, framework leakage into domain, import dependency direction |
| **Error handling** | Bare excepts, swallowed errors, error message quality, failure mode explicitness |
| **Testability** | Can domain logic be tested without infrastructure? Are I/O dependencies injectable? Test coverage and quality |
| **Observability** | Structured logging, actionable error messages, metrics presence, log level appropriateness |
| **Security** | Secrets in code/logs, input validation, SQL injection vectors, PII handling, dependency vulnerabilities |
| **Performance patterns** | Unbounded queries, N+1 patterns, missing pagination, memory-unbounded processing, missing timeouts |
| **Idempotency & reliability** | Retry safety, duplicate handling, transaction boundaries, failure recovery paths |
| **Dependency hygiene** | Pinned versions, lock files, unnecessary dependencies, version conflicts, deprecated packages |

## Severity classification

| Severity | Criteria | Action |
|---|---|---|
| **Critical** | Active data loss/corruption risk, security vulnerability, production outage trigger | Fix immediately; block new feature work if needed |
| **High** | Correctness bug under known conditions, missing error handling on critical paths, architectural violation that compounds over time | Fix in the current or next sprint |
| **Medium** | Code smell that reduces maintainability, missing types on public APIs, inconsistent patterns, poor testability | Plan into backlog; fix opportunistically when touching nearby code |
| **Low** | Style inconsistency, naming improvements, minor dead code, documentation gaps | Fix during regular refactoring passes; do not create dedicated tickets |

Severity is about **impact and likelihood**, not about how "ugly" the code looks. A well-named function with a data corruption bug is critical; a poorly-named function that works correctly is low.

## Tech debt identification

Not all imperfect code is tech debt. Distinguish:

- **Deliberate trade-off**: the team knowingly chose a simpler approach with documented limitations. This is acceptable if the limitations are documented and the upgrade path is clear.
- **Accumulated debt**: shortcuts taken under pressure that were never revisited. Identifiable by: no documentation of the trade-off, workarounds scattered across the codebase, patterns that contradict the project's own conventions.
- **Bit rot**: code that was correct when written but has become problematic due to changed requirements, grown scale, or deprecated dependencies.

For each debt item, assess:
- **Cost of keeping it**: what breaks, slows down, or becomes risky if this stays?
- **Cost of fixing it**: effort, risk of regression, required test coverage
- **Trigger for action**: what event (scale threshold, new feature, incident) would make fixing this urgent?

## Refactoring principles

When planning refactoring based on audit findings:

- **Characterization tests first**: before changing behavior, capture current behavior in tests. If you don't know what the code does, you can't safely change it.
- **Incremental over big-bang**: prefer a series of small, independently deployable changes over a single large rewrite. Each step should leave the system in a working state.
- **Preserve working structure unless it is the problem**: do not replace the current architecture just to make it "cleaner"; extract and improve the specific boundary, dependency, or failure mode that is creating risk.
- **Broader restructuring is justified when the current architecture is itself the recurring source of defects, delivery friction, or operational risk.**
- **Strangler fig for service extraction**: when extracting a module into a service, route traffic gradually — old and new paths coexist until the new one is proven.
- **Expand/contract for interface changes**: add the new interface alongside the old one, migrate consumers, then remove the old interface. Never break callers.
- **Parallel run for data migration validation**: when migrating data transformations, run old and new pipelines in parallel and compare outputs before cutting over.
- **One theme at a time**: refactor one quality dimension per pass (e.g., "add error handling" or "extract domain logic"). Mixing themes in one PR makes review and rollback harder.

## Communicating findings

Audit findings are only useful if they lead to action. Structure communication around:

- **What**: the specific pattern or issue observed (with file/module references)
- **Why it matters**: the concrete risk or cost (not "this is not best practice" but "this will cause duplicate records on retry")
- **How to fix**: a direction, not a full implementation — enough for the team to estimate and plan
- **Priority**: severity + recommended timeline

Avoid:
- Listing every instance of a repeated problem — identify the pattern, give 2-3 examples, estimate total count
- Mixing audit findings with feature requests — keep them separate
- Criticizing without context — acknowledge constraints and trade-offs
