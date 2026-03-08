---
name: arch-ddd-patterns
description: Domain-Driven Design for system decomposition — ubiquitous language, bounded context identification, aggregates, context mapping patterns, domain events, and integration strategies. Use when decomposing complex domains into services and defining system boundaries.
---

# Domain-Driven Design (DDD) Patterns

Strategic and tactical DDD patterns for decomposing complex systems into well-bounded, autonomous components.

## When to use

- Decomposing a complex domain into services or modules.
- Defining system boundaries, data ownership, and integration contracts.
- Identifying aggregates and transaction boundaries.
- Designing inter-context communication (sync, async, events).
- Evaluating whether DDD tactical patterns apply to a specific problem.

## When to apply DDD

| Domain complexity | Strategic DDD (contexts, mapping) | Tactical DDD (aggregates, entities, events) |
|-------------------|----------------------------------|---------------------------------------------|
| **Complex** — many business rules, multiple interacting domains | Yes | Yes |
| **Moderate** — some business logic, clear data model | Yes (for boundaries) | Selectively (for core domain) |
| **Simple CRUD** — thin logic, mostly data in/out | Optional | No (over-engineering) |
| **Data pipelines** — ETL/ELT, batch processing | Yes (for data ownership boundaries) | Rarely (no rich domain logic) |

Default: always apply strategic DDD (bounded contexts) for system decomposition. Apply tactical DDD only where domain logic is complex enough to benefit.

## Strategic DDD

### Ubiquitous language

A shared vocabulary between engineers, domain experts, and the codebase.

Rules:
- Terms must mean the same thing in conversations, documentation, code, and APIs.
- When different contexts use the same word with different meaning (e.g., "Account" in billing vs authentication), they are separate bounded contexts.
- Code should use domain terms in class names, function names, and API endpoints — not technical jargon.
- Maintain a glossary for each bounded context.

### Subdomain classification

| Type | Description | Investment |
|------|-------------|------------|
| **Core** | Competitive advantage, unique business logic | Maximum investment, custom development, DDD tactical patterns |
| **Supporting** | Necessary but not unique, enables core | Moderate investment, may use frameworks |
| **Generic** | Commodity functionality (auth, email, payments) | Minimum investment, buy or use OSS |

Classify before designing — invest design effort proportional to subdomain importance.

### Bounded context identification

A bounded context is a boundary within which a domain model is consistent and complete.

Identification heuristics:
- **Language boundary** — when the same word means different things, you have two contexts.
- **Data ownership** — each context owns its data and exposes it through interfaces.
- **Team boundary** — contexts often align with team ownership (Conway's law).
- **Change frequency** — components that change together belong in the same context; components that change independently are separate.
- **Consistency boundary** — operations that must be transactionally consistent belong in the same context.

### What a bounded context must define

- **Name** and description (using ubiquitous language).
- **Owned data** — which entities and tables belong exclusively to this context.
- **Public interface** — APIs, events, or shared contracts that other contexts consume.
- **Private internals** — domain model, storage schema, internal workflows (hidden from other contexts).
- **Independence** — can be deployed, scaled, and evolved independently.

## Context mapping

Relationships between bounded contexts. Document explicitly — implicit relationships become accidental coupling.

### Patterns

| Pattern | Relationship | When to use |
|---------|-------------|-------------|
| **Customer–Supplier** | Upstream (supplier) provides what downstream (customer) needs | Supplier respects customer's requirements; good for internal teams |
| **Conformist** | Downstream conforms to upstream's model without influence | Upstream is external or too powerful to negotiate with (third-party API) |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model into its own | Upstream model is incompatible with downstream's domain; isolate legacy systems |
| **Published Language** | Shared interchange format (protobuf, JSON schema, Avro) | Multiple consumers need the same data in a standard format |
| **Shared Kernel** | Two contexts share a small, jointly-owned model | Only when tight collaboration exists and shared model is small and stable |
| **Open Host Service** | Upstream provides a well-defined API for multiple consumers | Upstream serves many downstreams with a stable contract |
| **Separate Ways** | No integration — contexts operate independently | Contexts have no meaningful dependency |

### Decision guide

- **Default**: Customer–Supplier for internal context relationships.
- **External systems**: Conformist + ACL (accept their model, translate at the boundary).
- **Avoid Shared Kernel** unless absolutely necessary — it creates coupling and coordination overhead.
- **Published Language** for event-driven integration between contexts.

## Tactical DDD

Apply within a bounded context for complex domain logic. Skip for simple CRUD or data pipeline contexts.

### Entities

Objects with identity that persists across time and state changes.

- Identity is stable (ID, not attributes).
- State changes are governed by business rules.
- Example: `Order`, `Customer`, `Account`.

### Value objects

Objects defined by their attributes, no identity. Immutable.

- Two value objects with same attributes are equal.
- Example: `Money(amount=100, currency="USD")`, `Address`, `DateRange`.
- Prefer value objects over primitives for domain concepts.

### Aggregates

Cluster of entities and value objects treated as a single unit for data changes.

Rules:
- **One aggregate root** — external references point only to the root.
- **Transaction boundary** — changes within an aggregate are transactionally consistent.
- **Cross-aggregate references** — by ID only, not by object reference.
- **Keep aggregates small** — large aggregates cause contention and complexity.
- **Cross-aggregate consistency** — eventual, via domain events. Strong consistency across aggregates requires explicit justification.

Aggregate sizing heuristic: if two entities must change together in a single transaction to maintain an invariant, they belong in the same aggregate. If they can change independently, separate them.

### Domain events

Something that happened in the domain that other parts of the system may care about.

- Named in past tense: `OrderPlaced`, `PaymentReceived`, `CustomerAddressChanged`.
- Immutable — facts about what happened, not commands.
- Contain enough information for consumers to act without calling back to the source.
- Published after the aggregate state change is committed.

### Domain services

Operations that don't naturally belong to a single entity or value object.

- Stateless — all inputs are explicit parameters.
- Named with domain verbs: `TransferFunds`, `CalculateShippingCost`.
- Use when the operation spans multiple aggregates or requires external information.

### Repositories

Abstraction over data access for aggregates.

- One repository per aggregate root.
- Interface defined in the domain layer, implementation in infrastructure.
- Hides storage details (SQL, S3, cache) from domain logic.

## Integration between contexts

### Synchronous (API calls)

- One context calls another's API to get data or trigger an action.
- **When**: caller needs an immediate response; latency is acceptable.
- **Risk**: temporal coupling (if downstream is down, upstream fails).
- **Mitigation**: timeouts, circuit breaker, fallback strategy.

### Asynchronous (domain events)

- Context publishes events; interested contexts subscribe and react.
- **When**: eventual consistency is acceptable; contexts should be decoupled.
- **Benefit**: no temporal coupling, contexts evolve independently.
- **Risk**: eventual consistency complexity, event schema evolution.
- **Delivery**: assume at-least-once; consumers must be idempotent.

### Database integration (anti-pattern)

- One context reads/writes another context's database directly.
- **Never do this** — it breaks encapsulation, creates invisible coupling, and makes independent evolution impossible.
- If you find this in an existing system, plan ACL or API extraction.

### Decision guide

| Criteria | Sync (API) | Async (events) |
|----------|-----------|----------------|
| Consistency | Strong (immediate) | Eventual |
| Coupling | Temporal (both must be up) | Decoupled |
| Complexity | Lower | Higher (event infrastructure, idempotency) |
| Default for queries | Yes (read data from source) | No |
| Default for notifications | No | Yes (announce what happened) |

## Checklist

- Bounded contexts are identified with explicit names, owned data, and public interfaces.
- Context map documents relationships between all contexts.
- Ubiquitous language is established per context, reflected in code and APIs.
- Subdomains are classified (core, supporting, generic) with proportional investment.
- Aggregates define clear transaction boundaries and invariant enforcement.
- Cross-aggregate consistency uses domain events (eventual), not distributed transactions.
- Integration between contexts uses APIs or events, never direct database access.
- Anti-corruption layers isolate incompatible external models.

## Failure modes

- **Missing context boundaries** — monolithic domain model shared across teams, impossible to evolve independently.
- **Too many contexts** — premature decomposition creating excessive coordination overhead. Start with fewer, split when needed.
- **Shared database across contexts** — invisible coupling, broken encapsulation, migration nightmare.
- **Large aggregates** — transaction contention, complex loading, slow operations. Keep aggregates small.
- **No ubiquitous language** — code uses different terms than business, causing misunderstandings and bugs.
- **Tactical DDD for CRUD** — applying entities, aggregates, repositories to thin logic that doesn't benefit from the abstraction.
- **Sync integration everywhere** — cascading failures when one context is down. Default to async where eventual consistency is acceptable.
- **Events without schema contract** — consumers break silently when producer changes event structure.
