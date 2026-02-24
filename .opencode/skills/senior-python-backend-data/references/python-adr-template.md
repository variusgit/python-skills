---
name: python-adr-template
description: Lightweight decision record template for non-trivial changes — context, decision, alternatives, trade-offs, consequences, validation, and rollout/backout plan. Use when a change involves schema strategy, consistency model, new dependency, backfill plan, or significant performance/cost trade-off.
---

# Mini-ADR (Decision Record Template)

Write a mini-ADR when a change is non-trivial: schema strategy, consistency model, idempotency approach, backfill plan, major performance/cost tradeoff, new dependency.

## When to use

- Architecture or dependency choices with meaningful trade-offs.
- Schema/idempotency/backfill semantics that affect correctness.
- Performance/cost/operability decisions with long-lived impact.
- Compatibility-policy changes for APIs/events.

Read `./python-best-practices.md` first, then use this template to record non-trivial decisions.

## Template

- **Context**
  - problem statement
  - constraints (stack foundations, data correctness requirements, SLO/SLA)
- **Decision**
  - what we will do (one paragraph)
- **Alternatives**
  - 1–3 realistic options considered
- **Trade-offs**
  - correctness / latency / cost / operability / complexity
- **Consequences**
  - what changes for operations, maintenance, and future work
- **Validation**
  - tests
  - runtime signals (logs/metrics) that prove it works
- **Rollout/Backout**
  - migration plan, feature flags, rollback steps

## Checklist

- Context and constraints are explicit and testable.
- Decision is concrete and scoped to current problem.
- Alternatives and trade-offs are realistic, not strawman options.
- Validation and runtime signals are defined.
- Rollout and backout are operationally executable.

## Failure modes

- ADR states conclusions without concrete constraints/evidence.
- Missing rollout/backout creates release-time ambiguity.
- Trade-offs omit correctness or operability implications.
- Validation is vague and not measurable at runtime.
- Decision scope drifts beyond the actual change.
