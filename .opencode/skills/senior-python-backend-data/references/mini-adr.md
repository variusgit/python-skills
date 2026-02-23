# Mini-ADR (Decision Record Template)

Write a mini-ADR when a change is non-trivial: schema strategy, consistency model, idempotency approach, backfill plan, major performance/cost tradeoff, new dependency.

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
