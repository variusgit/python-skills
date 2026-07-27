---
name: python-data-jobs
description: Practices for batch data jobs and object storage layouts — S3 partitioning, Parquet tuning, PySpark job structure and performance, write patterns, incremental loads, data validation, error handling, and schema management. Use when building, refactoring, or reviewing data jobs and storage designs.
---

# Data Jobs & Storage (S3/Parquet, PySpark Hygiene)

This document defines practices for batch data jobs and object storage layouts that support reprocessing, correctness, and cost-efficiency.

Read `./python-best-practices.md` first, then apply this document for data-job and storage-specific decisions.

## When to use

- Designing or refactoring batch jobs and storage layouts.
- Defining partitioning, watermark, dedup, or replay strategy.
- Planning incremental loads and large backfills.
- Reviewing PySpark/Python data jobs for correctness, performance, and cost.
- Choosing file formats, compression, and schema evolution strategy.

## Core principles

- Prefer **replayable, append-only** storage where feasible.
- Jobs must be safe under retries and re-runs (idempotent outputs).
- Be explicit about **event time vs processing time** and watermarking.
- Make schema and partitioning decisions deliberately and document them.
- Validate data at boundaries (after read, before write).

## S3 layout (canonical object/OLAP layer)

### Path layout patterns

Prefer Hive-style partitioning for compatibility with Spark, Athena, Presto, and catalog tools:

```
s3://bucket/domain/entity/year=YYYY/month=MM/day=DD/
```

Examples:

```
# Event data partitioned by date
s3://data-lake/events/page_views/year=2025/month=01/day=15/

# Domain data partitioned by logical key + date
s3://data-lake/finance/invoices/region=eu/year=2025/month=01/

# Job output with run metadata
s3://data-lake/derived/user_segments/dt=2025-01-15/
```

Rules:
- Align partition keys with query and backfill access patterns.
- Use date-based keys (`dt=`, `year=/month=/day=`) for time-series data.
- Add logical partition keys (region, tenant, source) only when queries filter by them.

### Partition sizing

- Target **128 MB – 1 GB** per Parquet file within a partition.
- Avoid too many small partitions (thousands of near-empty directories increase list/read cost).
- Avoid extremely wide partitions (single partition with hundreds of GB slows consumers).
- If partitions are too small, batch multiple intervals into one partition or compact regularly.

### Immutability

- Avoid in-place mutation unless explicitly justified and safe.
- If mutation is required, document:
  - how readers avoid partial writes (locking, manifest, atomic rename)
  - rollback strategy
- For idempotent writes, prefer overwrite-by-partition over append (see Write patterns).

## Parquet and file format guidance

### Parquet (default choice)

- Columnar format optimized for analytical reads and predicate pushdown.
- Prefer Parquet for all OLAP/batch output unless there is a specific reason otherwise.

### Compression

| Strategy | When to use |
|----------|-------------|
| **Snappy** | Default — fast compression/decompression, good balance |
| **Zstd** | Higher compression ratio needed, acceptable CPU cost |
| **Gzip** | Maximum compression, read-heavy workloads with rare writes |
| **Uncompressed** | Almost never — only for debugging |

### Row group sizing

- Default row group size: 128 MB (matches typical HDFS/S3 read block).
- Align row group size with partition file size for single-row-group-per-file when possible.
- Smaller row groups (32-64 MB) only when predicate pushdown on many columns is critical.

### Schema evolution

- **Safe changes**: add new optional columns (nullable), add metadata.
- **Unsafe changes**: remove columns, rename columns, change types — require coordinated migration.
- Track schema version in file metadata or partition path when evolution is expected.
- When evolving schemas, use `mergeSchema` option in Spark reads or explicit schema-on-read mapping.

### Predicate pushdown

- Structure columns so common filters are low-cardinality and placed early in the schema.
- Partition pruning handles date/region filters at the directory level.
- Column-level pushdown works on min/max statistics within row groups — avoid random UUIDs as the first column.

## PySpark job structure

### Canonical job skeleton

```python
from pyspark.sql import SparkSession, DataFrame
import pyspark.sql.functions as F
from pyspark.sql.types import StructType

def create_spark_session(app_name: str) -> SparkSession:
    return (
        SparkSession.builder
        .appName(app_name)
        .config("spark.sql.sources.partitionOverwriteMode", "dynamic")
        .getOrCreate()
    )

def read_input(spark: SparkSession, path: str, schema: StructType) -> DataFrame:
    return spark.read.schema(schema).parquet(path)

def validate_input(df: DataFrame) -> DataFrame:
    row_count = df.count()
    if row_count == 0:
        raise ValueError(f"Empty input: expected rows, got 0")
    null_key_count = df.filter(F.col("id").isNull()).count()
    if null_key_count > 0:
        raise ValueError(f"Found {null_key_count} rows with null primary key")
    return df

def transform(df: DataFrame) -> DataFrame:
    return (
        df
        .filter(F.col("status") == "active")
        .withColumn("processed_at", F.current_timestamp())
        .dropDuplicates(["id"])
    )

def validate_output(df: DataFrame, input_count: int) -> DataFrame:
    output_count = df.count()
    if output_count == 0:
        raise ValueError("Transform produced zero rows")
    if output_count > input_count:
        raise ValueError(
            f"Output ({output_count}) exceeds input ({input_count}) — possible fan-out or dedup error"
        )
    return df

def write_output(df: DataFrame, path: str, partition_cols: list[str]) -> None:
    (
        df.write
        .mode("overwrite")
        .partitionBy(*partition_cols)
        .parquet(path)
    )

def main():
    spark = create_spark_session("etl_users")
    raw = read_input(spark, "s3://lake/raw/users/dt=2025-01-15/", USER_SCHEMA)
    validated_input = validate_input(raw)
    input_count = validated_input.count()
    transformed = transform(validated_input)
    validate_output(transformed, input_count)
    write_output(transformed, "s3://lake/derived/active_users/", ["dt"])
```

Key points:
- Inputs/outputs are explicit (paths, schemas, partition columns).
- Validation at both boundaries (after read, before write).
- Pure transform function — no I/O, no side effects, testable in isolation.
- Schema passed explicitly — never rely on schema inference in production.

### Input/output contract

Every job must document:
- **Input**: source path pattern, expected schema, partition keys
- **Output**: target path pattern, output schema, partition keys, write mode
- **Dedup key**: which columns define uniqueness
- **Tie-break rule**: when duplicates exist, which record wins (latest `updated_at`, highest `version`, etc.)

## PySpark performance

### Broadcast joins

```python
from pyspark.sql.functions import broadcast

result = large_df.join(broadcast(small_df), "key")
```

- Use `broadcast()` when one side fits in driver memory (< 100 MB, tunable via `spark.sql.autoBroadcastJoinThreshold`).
- Avoids expensive shuffle. Critical for dimension lookups.
- Never broadcast a table that might grow beyond memory limits.

### Data skew handling

```python
# Salt the skewed key to spread across partitions
salt_range = 10
skewed_df = df.withColumn("salt", (F.rand() * salt_range).cast("int"))
salted_key = F.concat(F.col("key"), F.lit("_"), F.col("salt"))

result = (
    skewed_df.withColumn("salted_key", salted_key)
    .join(
        dimension_df.crossJoin(
            spark.range(salt_range).withColumnRenamed("id", "salt")
        ).withColumn("salted_key", F.concat(F.col("key"), F.lit("_"), F.col("salt"))),
        "salted_key",
    )
    .drop("salt", "salted_key")
)
```

Signs of skew:
- A few tasks take 10-100x longer than the median.
- `SpillDisk` or `SpillMemory` metrics are high for specific tasks.

Mitigations:
- Salting (above).
- Isolate the hot key and process it separately.
- Pre-aggregate before join to reduce cardinality.

### Repartition vs coalesce

| Operation | When to use |
|-----------|-------------|
| `repartition(n)` | Increase parallelism or redistribute by key (full shuffle) |
| `repartition(n, "col")` | Co-partition for downstream joins on `col` |
| `coalesce(n)` | Reduce partitions without full shuffle (only merge, not redistribute) |

- Before writing, `coalesce` to control output file count and target file size.
- Avoid `repartition(1)` for large datasets — creates a single-task bottleneck.

### UDF hygiene

- **Avoid UDFs** when a built-in Spark function exists — UDFs disable Catalyst optimization and require serialization.
- If unavoidable, prefer `pandas_udf` (vectorized) over row-level Python UDFs.
- Never use UDFs for simple operations like null checks, string manipulation, or date arithmetic.

```python
# Bad — UDF for something Spark handles natively
@udf(returnType=StringType())
def upper(s):
    return s.upper() if s else None

# Good — native Spark function
df.withColumn("name_upper", F.upper(F.col("name")))
```

### Caching strategy

- `cache()` / `persist()` only when a DataFrame is reused multiple times in the same job.
- Always `unpersist()` when done to free memory.
- Never cache data that is read once and written once (the default ETL flow).
- For iterative algorithms, use `checkpoint()` to break lineage and prevent stack overflow in complex plans.

## Write patterns

### Overwrite-by-partition (default for idempotent writes)

```python
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

(
    df.write
    .mode("overwrite")
    .partitionBy("dt")
    .parquet("s3://lake/derived/entity/")
)
```

- `dynamic` mode overwrites only the partitions present in the DataFrame, leaving others untouched.
- Safe for retries: re-running with the same input produces the same output.
- Requires deterministic partition selection — no `processing_time` in partition key.

### Staging + atomic commit

For cases where overwrite-by-partition is insufficient (multi-table consistency, manifest-based reads):

```
1. Write to staging:  s3://lake/staging/job_id=abc123/entity/dt=2025-01-15/
2. Validate output (schema, row count, constraints).
3. On success: move/copy to final path or update catalog pointer.
4. On failure: delete staging prefix and fail the job.
```

- Readers never see partial writes.
- Staging prefix includes job/run ID for traceability.
- Clean up old staging data on a TTL basis.

### Schema evolution on write

```python
(
    df.write
    .mode("overwrite")
    .option("mergeSchema", "true")
    .partitionBy("dt")
    .parquet(output_path)
)
```

- Use `mergeSchema` only when adding nullable columns (safe evolution).
- For breaking changes (type change, column removal), use expand/contract migration:
  1. Write new schema alongside old.
  2. Migrate readers.
  3. Drop old schema.

## Data validation

### Pre-write validation (required)

```python
def validate_output(df: DataFrame, expectations: dict) -> None:
    count = df.count()
    if count == 0:
        raise ValueError("Output is empty")

    if "min_rows" in expectations and count < expectations["min_rows"]:
        raise ValueError(f"Row count {count} below minimum {expectations['min_rows']}")

    null_counts = {
        col: df.filter(F.col(col).isNull()).count()
        for col in expectations.get("not_null_columns", [])
    }
    violations = {col: n for col, n in null_counts.items() if n > 0}
    if violations:
        raise ValueError(f"Null constraint violations: {violations}")

    if "unique_columns" in expectations:
        total = df.count()
        distinct = df.dropDuplicates(expectations["unique_columns"]).count()
        if distinct != total:
            raise ValueError(
                f"Uniqueness violation on {expectations['unique_columns']}: "
                f"{total - distinct} duplicates"
            )
```

### Post-write validation

After writing, verify the output is readable and consistent:
- Re-read and compare row count to pre-write count.
- Spot-check schema matches expectations.
- For critical pipelines, compare aggregate metrics (sums, distinct counts) before and after.

### Quarantine pattern (corrupt/invalid records)

```python
valid_df = df.filter(F.col("id").isNotNull() & F.col("amount") > 0)
quarantine_df = df.subtract(valid_df)

write_output(valid_df, "s3://lake/derived/orders/", ["dt"])

if quarantine_df.count() > 0:
    write_output(quarantine_df, "s3://lake/quarantine/orders/", ["dt"])
    log.warning("Quarantined %d invalid records", quarantine_df.count())
```

Rules:
- Never silently drop records — quarantine and alert.
- Define thresholds: if quarantine rate exceeds X%, fail the job.
- Quarantined data must be reviewed and either fixed or acknowledged.

### Data quality metrics

Track per job run:
- **Row count** (input, output, quarantine)
- **Null rate** per critical column
- **Freshness** — max event_time in output vs wall clock
- **Completeness** — expected partitions vs actual partitions written
- Log these as structured metrics for alerting and dashboards.

## Incremental loads

### Full-refresh vs incremental — decision guide

| Criteria | Full refresh | Incremental |
|----------|-------------|-------------|
| Source size | Small-medium (fits in time budget) | Large (full scan is too slow/expensive) |
| Mutation pattern | Source mutates in place | Source is append-only or has change tracking |
| Complexity | Low (simpler, safer) | Higher (watermarks, dedup, late data) |
| Correctness risk | Low (always consistent) | Higher (drift, missed updates) |
| Default choice | **Start here** unless scale forces incremental | Migrate to incremental when full-refresh budget is exceeded |

### Watermark pattern

```python
def get_high_watermark(spark: SparkSession, state_path: str) -> str | None:
    try:
        state = spark.read.json(state_path)
        return state.collect()[0]["high_watermark"]
    except Exception:
        return None

def save_high_watermark(spark: SparkSession, state_path: str, value: str) -> None:
    spark.createDataFrame([{"high_watermark": value}]).write.mode("overwrite").json(state_path)

def incremental_read(spark: SparkSession, source_path: str, watermark_col: str, hwm: str | None) -> DataFrame:
    df = spark.read.parquet(source_path)
    if hwm:
        df = df.filter(F.col(watermark_col) > hwm)
    return df

# Usage
hwm = get_high_watermark(spark, "s3://lake/state/orders_hwm/")
new_data = incremental_read(spark, "s3://lake/raw/orders/", "updated_at", hwm)

if new_data.count() > 0:
    transformed = transform(new_data)
    write_output(transformed, "s3://lake/derived/orders/", ["dt"])
    new_hwm = new_data.agg(F.max("updated_at")).collect()[0][0]
    save_high_watermark(spark, "s3://lake/state/orders_hwm/", str(new_hwm))
```

Key rules:
- Watermark must be monotonic (event_time, updated_at, sequence_id).
- Save watermark **after** successful write — not before.
- If watermark state is lost, fallback to full-refresh (safe reset).

### Late-arriving data

- Define a **reprocessing window**: how far back to look beyond the watermark (e.g., 3 days).
- For time-partitioned data, re-read and overwrite recent partitions within the window.
- Dedup across the window boundary: records in both old and new partitions.
- Document the maximum acceptable lateness and what happens beyond it.

## Error handling in data jobs

### Corrupt records

- Never silently drop or skip corrupt records without quarantine and alerting.
- Use Spark's `PERMISSIVE` mode with `columnNameOfCorruptRecord` to capture unparseable rows.
- Define a quarantine threshold — if > N% records are corrupt, fail the job.

### Partial failures

- If a job writes multiple partitions and one fails: do not leave partial output.
- Use staging + commit pattern to ensure atomicity across partitions.
- For multi-step pipelines, ensure each step is independently rerunnable.

### Retry safety

- Write patterns must be idempotent (overwrite-by-partition or staging+commit).
- Never append without dedup when retries are possible — creates duplicate records.
- Watermark/state updates must happen **after** successful output write.

## Schema management

### Versioning strategy

- Track schema version as metadata in output files or partition paths.
- When schema changes: increment version, update documentation, notify consumers.
- For Parquet: store schema version in file-level metadata key-value pairs.

### Compatibility rules

- **Backward compatible** (safe): add nullable columns, add metadata.
- **Forward compatible**: readers tolerate unknown columns (ignore extras).
- **Breaking**: remove columns, rename columns, change types — requires coordinated migration.

### Migration pattern for breaking changes

1. **Expand**: write outputs in both old and new schema (dual-write or new column alongside old).
2. **Migrate consumers**: update all readers to use new schema.
3. **Contract**: stop writing old schema, clean up old columns after confidence period.

## Checklist

- [ ] Is the job deterministic and idempotent (safe to re-run)?
- [ ] What is the replay/reprocessing source of truth?
- [ ] Are partition keys aligned with query and backfill patterns?
- [ ] Are file sizes within target range (128 MB – 1 GB)?
- [ ] Is the write pattern safe for retries (overwrite-by-partition or staging+commit)?
- [ ] Are inputs validated after read (schema, nulls, constraints)?
- [ ] Are outputs validated before write (row count, uniqueness, quality)?
- [ ] Are corrupt/invalid records quarantined with alerting?
- [ ] Is the watermark/incremental strategy documented and resumable?
- [ ] How are late-arriving records handled?
- [ ] Is schema evolution safe (no breaking changes without migration)?
- [ ] Are data quality metrics tracked (row count, null rate, freshness)?

## Failure modes

- Non-deterministic transforms causing non-reproducible outputs.
- Partition strategy creating too many tiny files and high list/read cost.
- Backfills without throttling/resume strategy causing cluster instability.
- Missing schema/data validations allowing silent data corruption.
- Incremental logic scanning full datasets due to weak partition pruning.
- Silently dropping corrupt records without quarantine or alerting.
- Append-without-dedup under retries creating duplicate records.
- Watermark saved before output write — failed write loses data on retry.
- UDFs on hot paths defeating Catalyst optimization and causing 10x slowdown.
- Data skew causing a single task to run 100x longer than the median.
- Schema evolution breaking downstream consumers without coordinated migration.
- Missing file size control producing thousands of tiny files per partition.
