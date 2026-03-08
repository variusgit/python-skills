---
name: arch-system-design
description: Architectural patterns for high-load systems — NFR framework, capacity planning, scaling strategies, caching, async processing, resilience, consistency patterns, and CQRS/event sourcing. Use when designing systems with significant load, latency, or availability requirements.
---

# System Design (High-Load, Scaling, Resilience)

Architectural patterns and decision frameworks for designing systems that handle significant load, maintain low latency, and recover gracefully from failures.

## When to use

- Designing systems with quantified load expectations (requests/sec, data volume, user count).
- Choosing scaling, caching, or consistency strategies.
- Designing resilience and failure recovery.
- Evaluating architectural trade-offs for performance, cost, and complexity.

## Non-functional requirements framework

Define NFRs before choosing patterns. Vague NFRs ("fast", "scalable") lead to wrong decisions.

### Template per component or endpoint

| Dimension | Questions to answer |
|-----------|-------------------|
| **Latency** | p50, p95, p99 targets? Which operations are latency-sensitive? |
| **Throughput** | Peak requests/sec? Events/sec? Data volume per day? Growth rate? |
| **Availability** | SLO target (99.9%? 99.95%)? Acceptable downtime window? |
| **Consistency** | Per operation: strong or eventual? What staleness is acceptable? |
| **Durability** | What data loss is acceptable? (zero for financial, minutes for analytics) |
| **Cost** | Compute/storage budget? Cost per request/GB acceptable? |
| **Security** | Authentication/authorization model? Data classification? Compliance? |

### Load estimation

Before choosing patterns, estimate:
1. **Current load** — measure or estimate from business metrics.
2. **Peak load** — typical peak-to-average ratio is 2-5x for web services, 10-50x for batch.
3. **Growth projection** — 6 months, 1 year, 3 years. Design for 1 year, have a plan for 3.
4. **Hot spots** — which entities/keys will concentrate load?

## Scaling strategies

### Vertical scaling

Scale up a single node (bigger CPU, more RAM, faster storage).

- **When**: load is moderate, data fits on one node, simplicity is valuable.
- **Limit**: hardware ceiling, single point of failure, cost curve is superlinear.
- **Typical for**: PostgreSQL primary (before sharding), small-medium services.

### Horizontal scaling — stateless services

Run multiple identical instances behind a load balancer.

- **Prerequisite**: service must be stateless (no in-process state between requests).
- **Session state**: externalize to Redis/PostgreSQL or use stateless tokens (JWT).
- **Scaling signal**: CPU/memory utilization, request queue depth, latency percentile.
- **K8s integration**: Horizontal Pod Autoscaler (HPA) based on CPU, memory, or custom metrics.

### Read/write separation

Separate read and write paths when read load dominates (>10:1 read/write ratio).

| Pattern | How | Trade-off |
|---------|-----|-----------|
| **Read replicas** | PostgreSQL streaming replication, reads from replicas | Replication lag (seconds), stale reads |
| **Materialized views** | Pre-computed query results, refreshed periodically | Staleness, storage cost, refresh overhead |
| **CQRS** | Separate read model optimized for queries | Complexity, eventual consistency between models |
| **Cache layer** | Hot data in Redis/Memcached | Invalidation complexity, memory cost |

### Data partitioning (sharding)

Split data across multiple nodes when a single node cannot hold the dataset or handle the write load.

| Strategy | How | Best for |
|----------|-----|----------|
| **Range partitioning** | By time range, ID range | Time-series, sequential access patterns |
| **Hash partitioning** | Hash(key) % N | Even distribution, point lookups |
| **Tenant partitioning** | One partition per tenant/customer | Multi-tenant SaaS, data isolation |
| **Geographic** | By region/location | Data locality, compliance requirements |

Decision criteria:
- Partition key must align with access patterns (queries that span partitions are expensive).
- Rebalancing strategy must exist (what happens when you add a partition).
- Cross-partition operations require explicit handling (scatter-gather, saga).

### Async processing

Decouple producers from consumers with queues/events when:
- The caller does not need an immediate response.
- Processing time is unpredictable or long.
- Load must be smoothed (backpressure).
- Multiple consumers need the same event.

Key decisions:
- **Queue vs topic** — point-to-point (one consumer) vs fan-out (multiple consumers).
- **Ordering** — global ordering is expensive; prefer partition-scoped ordering by entity key.
- **Delivery guarantee** — assume at-least-once; design consumers to be idempotent.
- **Backpressure** — what happens when consumers are slower than producers? Queue depth limits, rate limiting, or scaling consumers.

## Caching architecture

### Caching strategies

| Strategy | How | Best for |
|----------|-----|----------|
| **Cache-aside** | App checks cache first; on miss, reads DB, writes to cache | General-purpose, most common |
| **Read-through** | Cache auto-loads from DB on miss | Simpler app code, cache library handles loading |
| **Write-through** | Writes go to cache AND DB synchronously | Strong consistency between cache and DB |
| **Write-behind** | Writes go to cache, async flush to DB | High write throughput, risk of data loss |

### Cache invalidation

| Approach | Trade-off |
|----------|-----------|
| **TTL-based** | Simple; staleness bounded by TTL; may serve stale data |
| **Event-based** | Fresh; requires event infrastructure; more complex |
| **Version-based** | Fresh; requires version tracking; good for immutable data |
| **Manual/explicit** | Precise; error-prone; doesn't scale |

### What to cache

- **Hot lookup data** — user profiles, config, permissions (high read, low write).
- **Computed results** — aggregations, search results, recommendations.
- **Session state** — if externalizing from stateless services.
- **NOT**: data that changes frequently AND must be consistent, transactional data.

### Sizing

- Estimate working set size (how much data is accessed in a typical window).
- Monitor hit rate — below 80% usually means cache is too small or access pattern is too random.
- Plan for cold start (cache is empty after restart).

## Resilience patterns

### Timeouts

Every external call must have an explicit timeout. No infinite waits.

| Call type | Typical timeout |
|-----------|----------------|
| HTTP API (user-facing path) | 1–5 seconds |
| HTTP API (internal service) | 5–15 seconds |
| Database query (OLTP) | 1–5 seconds |
| Database query (batch/reporting) | 30 seconds – 5 minutes |
| Message publish | 5–10 seconds |

### Retry with backoff

- Bounded attempts (3-5 max).
- Exponential backoff with jitter (prevent thundering herd).
- Retry only transient errors (5xx, timeout, connection refused). Never retry 4xx.
- Target operation must be idempotent.

### Circuit breaker

Open the circuit after N consecutive failures; reject calls immediately for a recovery period; probe with single requests to detect recovery.

- **When**: external dependency with known availability risk on a critical path.
- **Not needed for**: PostgreSQL (use connection pool limits + statement_timeout), internal calls, async with DLQ.

### Bulkhead

Isolate failure domains so one degraded dependency does not saturate all resources.

- Separate connection pools per external dependency.
- Separate thread/worker pools for different workload classes.
- Rate limit expensive operations independently.

### Backpressure

When a system is overwhelmed, actively push back rather than accept and fail:
- Return HTTP 429 (Too Many Requests) with Retry-After header.
- Limit queue depth; reject new messages when full.
- Shed low-priority traffic first.

### Graceful degradation

When a non-critical dependency is unavailable:
- **Optional features**: skip and return partial response.
- **Enrichment**: return un-enriched data with a flag.
- **Recommendations**: return popular/default items instead.
- **Critical dependencies (payments, auth)**: fail fast, do not degrade.

## Consistency patterns

### Strong consistency

All reads see the latest write. Required when:
- Financial transactions (balances, transfers).
- Inventory with limited stock.
- Uniqueness constraints that must hold globally.

Implementation: single leader (PostgreSQL primary), transactions, advisory locks.

### Eventual consistency

Reads may see stale data, but will converge. Acceptable when:
- Analytics and reporting (seconds/minutes of staleness is fine).
- Search indexes (near-real-time is sufficient).
- Notifications and non-critical updates.
- Cross-service data replication.

Implementation: async replication, event-driven updates, materialized views with refresh.

### Bounded staleness

Middle ground: reads may be stale, but by no more than N seconds. Useful for:
- User-facing dashboards (stale by ≤30s is acceptable).
- Cross-region reads (replication lag is bounded).

### CQRS (Command Query Responsibility Segregation)

Separate write model (optimized for correctness) from read model (optimized for queries).

**When to use**:
- Read and write patterns are fundamentally different (complex writes, diverse read views).
- Read load is 10-100x higher than write load.
- Multiple read projections are needed from the same data.

**When NOT to use**:
- Simple CRUD with similar read/write patterns.
- Small data volume where a single model works fine.
- Team lacks experience with eventual consistency.

**Trade-offs**: eventual consistency between models, increased complexity, need to handle projection lag.

### Saga pattern

Coordinate multi-step business processes across bounded contexts without distributed transactions.

**When to use**:
- Business process spans multiple services/contexts.
- Each step must be independently compensatable.
- Distributed transactions are not feasible.

**Orchestration vs choreography**:
- **Orchestration** (central coordinator): simpler to understand, single point of control, easier monitoring. Prefer when process is complex.
- **Choreography** (event-driven): more decoupled, no single point of failure, harder to debug. Prefer when steps are simple and independent.

**Trade-offs**: compensating actions must be defined for every step, eventual consistency, debugging complexity.

## Checklist

- NFRs are quantified per component/endpoint.
- Load estimation covers current, peak, and projected growth.
- Scaling strategy is chosen based on load profile (not just "use microservices").
- Caching strategy matches access patterns; invalidation is explicit.
- Consistency model is chosen per operation with explicit staleness bounds.
- Resilience patterns (timeouts, retries, circuit breaker) are designed for external dependencies.
- Backpressure and graceful degradation strategies are defined.
- Async processing has explicit ordering, delivery, and idempotency semantics.

## Failure modes

- Scaling strategy does not match actual access patterns (hash partitioning with range queries).
- Cache without invalidation strategy serving stale data indefinitely.
- Missing timeouts causing cascading failures across services.
- CQRS applied to simple CRUD, adding complexity without benefit.
- Saga without compensating actions — partial failures leave system in inconsistent state.
- Ignoring cold-start scenarios (empty cache, new replica, restarted service).
- Premature optimization — adding caching/sharding before measuring actual bottlenecks.
