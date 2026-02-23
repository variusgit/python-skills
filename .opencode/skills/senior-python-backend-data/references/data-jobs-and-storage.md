# Data Jobs & Storage (S3/Parquet, PySpark Hygiene)

This document defines practices for batch data jobs and object storage layouts that support reprocessing and correctness.

## Core principles

- Prefer **replayable, append-only** storage where feasible.
- Jobs must be safe under retries and re-runs (idempotent outputs).
- Be explicit about **event time vs processing time** and watermarking.
- Make schema and partitioning decisions deliberately and document them.

## S3 layout (canonical object/OLAP layer)

### Partitioning
- Partition by access patterns (commonly date or logical partition keys).
- Avoid too many small partitions; avoid extremely wide partitions.
- Prefer stable partition keys that support backfills.

### Write strategy
- Use write patterns that are safe for retries:
  - overwrite-by-partition with deterministic partition selection, or
  - write to a staging prefix and commit atomically via pointer/manifest
- Include metadata when useful:
  - schema version, job version, run id, data interval

### Immutability
- Avoid in-place mutation unless explicitly justified and safe.
- If mutation is required, document:
  - how readers avoid partial writes
  - rollback strategy

## PySpark jobs (data-adjacent backend work)

- Make inputs/outputs explicit (paths, schemas, partitioning).
- Prefer deterministic transformations and stable ordering assumptions.
- Define deduplication keys and tie-break rules.
- Handle late/out-of-order data intentionally (watermarks, reprocessing window).
- Validate outputs:
  - schema checks (types, nullability)
  - basic constraints (uniqueness, ranges) where feasible
  - row-count sanity checks if applicable

## Incremental loads & backfills

- Define:
  - watermark source (event_time, updated_at, partition availability)
  - reprocessing window for late data
  - dedup strategy across partitions
- Backfills must be throttled, observable, and restartable.

## Cost/performance hygiene

- Avoid tiny files; compact when needed.
- Avoid scanning entire datasets for incremental work; use partitions and manifests.
- Prefer columnar formats (Parquet) and predicate pushdown-friendly schemas.

## Practical checklist (per job)

- Is it deterministic and idempotent?
- What is the replay source of truth?
- What are partition keys and why?
- How are late records handled?
- What validations catch silent corruption?
