# Operations, Jobs, and Query Behavior

## Container Images

TimescaleDB stopped building Bitnami images in 2.18.0. Move container
deployments to the official `timescale/timescaledb-ha` image.

## Background Jobs

Background jobs can have custom names (since 2.20.0). Continuous-aggregate jobs
include the aggregate name in the jobs informational view.

Job-history configuration can separately cap the number of successful and
failed executions retained for each job (since 2.23.0).

The `bgw_job` table moved into `_timescaledb_catalog` in 2.25.0, with a
`bgw_job` alias retained for compatibility. Prefer supported views and APIs
over a hard dependency on its private catalog location. Background worker jobs
can be configured with `work_mem`.

Background jobs no longer use advisory locks and support graceful cancellation
(since 2.26.0).

## Query Behavior and Correctness

### Compressed data

Vectorized aggregation handles NaN correctly, and `WHERE` comparisons on
compressed tables follow PostgreSQL's NaN comparison behavior (since 2.18.0).

Multi-column predicates can be pushed into compressed scans for `SELECT` and
`UPSERT` through composite bloom indexes (since 2.26.0). Use `EXPLAIN` to see
batch-pruning and false-positive statistics.

`timescaledb.enable_columnar_scan_filter_pushdown` controls whether
columnar-scan filters are pushed down and defaults to on (since 2.27.0):

```sql
SET timescaledb.enable_columnar_scan_filter_pushdown = off;
```

`compressed_data_column_size` returns `bigint` (since 2.27.0). Adjust explicit
casts and client result decoding that assumed a narrower type.

Updates that would unsafely modify unique columns on compressed chunks are
rejected (since 2.28.0).

### Partition comparisons and `MERGE`

Cross-type comparisons against a partitioning column no longer risk wrong
results or crashes (since 2.26.0).

Hypertables correctly handle `MERGE WHEN NOT MATCHED BY SOURCE` (since 2.28.0).

## Time Bucketing and GapFill

The timezone argument to `time_bucket_gapfill` must be constant (since 2.26.0).
Queries that derive it from a non-constant expression are rejected.

For `DATE` input, `time_bucket` rejects a sub-day offset (since 2.27.0).
Continuous-aggregate definitions reject non-positive time-bucket widths.

GapFill arguments may come from subquery results represented as executor
parameters (since 2.28.0), enabling parameterized query shapes that were
previously rejected.

`time_bucket` also accepts UUIDv7 and returns timezone-aware timestamps. See
[Hypertables, Chunks, and DDL](hypertables-chunks-and-ddl.md).

## Event Triggers

Event triggers can run when chunks are created (since 2.20.0), gated by
`timescaledb.enable_event_triggers`, which defaults to `OFF`:

```sql
SET timescaledb.enable_event_triggers = on;
```

Transition-table and hypertable trigger behavior is documented in
[Hypertables, Chunks, and DDL](hypertables-chunks-and-ddl.md).

## Maintenance and Observability

Index creation reports progress (since 2.27.0), making long-running builds
observable.

`VACUUM FULL` recompresses affected chunks (since 2.25.0). Include
recompression in runtime, locking, and disk-space estimates.

`VACUUM` and `ANALYZE` accept a continuous aggregate and redirect the operation
to its materialization hypertable (since 2.28.0):

```sql
VACUUM hourly_metrics;
ANALYZE hourly_metrics;
```

`timescaledb.stats_max_chunks` sets the per-database capacity of the in-memory
compressed-chunk statistics cache (since 2.28.0). It defaults to `1024`; set it
to `0` to disable the cache:

```sql
SET timescaledb.stats_max_chunks = 0;
```

## Expert and Diagnostic Configuration

`timescaledb.enable_compression_ratio_warnings` defaults to enabled (since
2.20.0), warning about poor compression ratios.

`timescaledb.default_chunk_time_interval` controls the default time interval
for newly created hypertables (since 2.26.0). It is an expert setting; leave it
unchanged unless specifically recommended.

Exact continuous-aggregate query rewrites and diagnostic output are disabled by
default (since 2.27.0):

```sql
SET timescaledb.enable_cagg_rewrites = on;
SET timescaledb.cagg_rewrites_debug_info = on;
```

The GUCs for Direct Compress, recompression, sparse indexes, and invalidation
collection are grouped with their feature guidance in the other references.
