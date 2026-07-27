---
name: python-messaging
description: Async messaging practices for at-least-once delivery — consumer idempotency, retry/DLQ strategies, command vs event semantics, outbox pattern, and ordering guarantees. Use when implementing or reviewing async consumers, producers, or event-driven flows.
---

# Messaging & Async Processing

Assume **at-least-once delivery** unless proven otherwise.

## When to use

- Implementing or reviewing async consumers/producers.
- Defining retry, DLQ, dedup, and ordering semantics.
- Designing reliable event publishing with transactional consistency.
- Investigating duplicate processing or message-loss incidents.

Read `./python-best-practices.md` first, then apply this document for messaging-specific delivery semantics.

## Idempotency (required)

Consumers and handlers must be idempotent:
- define a dedup key (event_id, idempotency_key, natural key)
- store dedup state when correctness requires it
- make side effects safe under retries (DB upserts, idempotent external calls)

## Retries and DLQ

- Use bounded retries with backoff and jitter.
- Define poison-message handling:
  - max attempts
  - DLQ routing
  - quarantine and manual remediation path

## Commands vs events

- **Command**: intent to do something (may fail)
- **Event**: fact that something happened (should be immutable)

Document semantics and consumers.

## Exactly-once visible effects (when needed)

If you must ensure DB updates and event publishing are consistent:
- consider the **outbox pattern**
  - write domain change + outbox record in one DB transaction
  - async publisher delivers outbox records to the broker
- ensure outbox is idempotent (unique keys, retry-safe publish)

## Ordering

- Do not assume global ordering.
- If ordering matters:
  - scope it to a key (aggregate/entity id)
  - choose a broker/topic partitioning strategy accordingly
  - enforce monotonicity in consumers if needed

## Checklist

- Delivery semantics are explicit (at-least-once by default).
- Consumer side effects are idempotent under retries/replays.
- Retry policy is bounded and poison-message path is defined.
- Ordering requirements are scoped and enforced by key.
- Outbox/consistency strategy is documented when required.

## Failure modes

- Assuming exactly-once semantics without technical guarantees.
- Non-idempotent handlers causing duplicate side effects.
- Infinite retries without DLQ/quarantine path.
- Implicit ordering assumptions breaking under partition rebalance.
- Publishing events outside transaction boundaries causing drift.
