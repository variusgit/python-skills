# Messaging & Async Processing (If Present)

Assume **at-least-once delivery** unless proven otherwise.

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
