---
name: arch-adr-template
description: Architectural Decision Record template for significant design choices — context, NFR impact, alternatives, trade-offs, consequences, delivery impact, validation, and rollout plan. Use when making or documenting architectural decisions.
---

# Architectural Decision Record (ADR)

Write an ADR when any of these are true:
- New bounded context or service boundary defined.
- Storage engine or processing framework chosen for a workload.
- Consistency model selected (strong vs eventual).
- Scaling, caching, or resilience strategy chosen.
- New technology proposed as add-on to the stack.
- Data model approach selected (normalized, dimensional, vault).
- ML serving pattern chosen (online, batch, streaming).
- Migration or evolution strategy defined.
- Significant cost, performance, or complexity trade-off made.

## Template

### Title
Short descriptive name: "Use ClickHouse for analytics serving layer"

### Status
Proposed | Accepted | Superseded by [ADR-xxx]

### Context
- Problem statement — what decision is needed and why.
- Constraints — stack foundations, NFRs, team capabilities, timeline.
- Current state — what exists today (if evolving an existing system).

### Decision
What we will do. One clear paragraph.

### NFR impact
How this decision affects non-functional requirements:

| NFR | Impact |
|-----|--------|
| Latency | ... |
| Throughput | ... |
| Availability | ... |
| Cost | ... |
| Operability | ... |

### Alternatives considered
1–3 realistic options with brief evaluation. Not strawman alternatives.

| Alternative | Pros | Cons | Why rejected |
|-------------|------|------|-------------|
| Option A | ... | ... | ... |
| Option B | ... | ... | ... |

### Trade-offs
Explicit trade-offs accepted with this decision:
- Correctness vs performance
- Simplicity vs flexibility
- Cost vs latency
- Operability vs complexity

### Consequences
What changes for the system, team, and future work:
- What becomes easier.
- What becomes harder.
- What new constraints are introduced.
- What technical debt is accepted (if any).

### Delivery impact
- How does this decision affect the delivery plan?
- What must be built first (prerequisites)?
- What can be parallelized?
- Is incremental delivery possible or is this a prerequisite gate?

### Validation
How we will know this decision is correct:
- Tests that validate the behavior.
- Runtime signals (metrics, logs) that prove it works.
- Success criteria with measurable thresholds.

### Rollout / backout
- Migration plan (expand/contract, strangler, dual-run).
- Feature flags or gradual rollout where applicable.
- Rollback steps if the decision proves wrong.
- Data cleanup if needed.

## Checklist

- Context and constraints are explicit and testable.
- Decision is concrete and scoped to the current problem.
- Alternatives are realistic, not strawman options.
- NFR impact is evaluated across all relevant dimensions.
- Trade-offs are explicit, not hidden.
- Consequences cover both positive and negative.
- Delivery impact is assessed (prerequisites, parallelism, incrementality).
- Validation criteria are measurable at runtime.
- Rollout and backout are operationally executable.

## Failure modes

- ADR states conclusions without constraints or evidence.
- Missing rollout/backout creates deployment ambiguity.
- Trade-offs omit correctness or operability implications.
- Alternatives are strawmen that make the chosen option look obvious.
- Validation is vague ("it should work") instead of measurable.
- Decision scope drifts beyond the actual problem.
- No delivery impact assessment — design is correct but undeliverable.
