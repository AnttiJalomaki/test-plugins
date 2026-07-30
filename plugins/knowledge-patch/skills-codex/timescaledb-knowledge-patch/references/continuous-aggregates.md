# Continuous aggregates

## Refresh in bounded and concurrent work

### Policies

Incremental refresh policies split work into smaller configurable batches.
Since 2.19.0, refreshes materialize newest data first so recent results become
current sooner while reducing memory and disk pressure. `refresh_newest_first`
became an explicit policy control in 2.20.0.

Non-overlapping refresh ranges can run concurrently and concurrent refresh
policies can be created since 2.21.0. Incremental policies are enabled by
default. Since 2.25.0, policy `buckets_per_batch` defaults to `10`, producing
smaller transactions unless explicitly overridden.

Refresh policies for compressed continuous aggregates can perform compression
as part of refresh since 2.27.0.

### Manual refresh

`refresh_continuous_aggregate` accepts optional `force` since 2.18.0. A forced
refresh consumes associated invalidations rather than leaving them pending
since 2.26.0.

Since 2.28.0, manual refresh accepts incremental batching controls:

```sql
CALL refresh_continuous_aggregate(
    'hourly_metrics',
    '2026-01-01'::timestamptz,
    '2026-06-01'::timestamptz,
    buckets_per_batch => 24,
    max_batches_per_execution => 5,
    refresh_newest_first => true
);
```

Invalidation-log processing now uses a lighter lock, so that phase does not
block unrelated operations on the same continuous aggregate.

## Manage invalidations

APIs for the hypertable invalidation log and materialization invalidations are
available since 2.20.0. Since 2.21.0, an explicit function or scheduled policy
can process hypertable invalidations, and an option can leave them unprocessed.

### Retired WAL path

The tech-preview `timescaledb.invalidate_using` option in 2.22.0 selected
trigger-based collection or WAL collection through logical decoding. When
omitted on an aggregate, it inherited the hypertable's method. The processing
path supported aggregates over multiple hypertables, with defaults:

- `cagg_processing_wal_batch_size = 10000`
- `cagg_processing_low_work_mem = 38.4MB`
- `cagg_processing_high_work_mem = 51.2MB`

An explicit `timescaledb.enable_cagg_wal_based_invalidation` GUC followed in
2.23.0. The entire WAL-based path was removed in 2.25.0; use trigger-based
invalidation.

### Suppress invalidations for a bulk load

`timescaledb.skip_cagg_invalidation` defaults off since 2.28.0. It skips
continuous-aggregate invalidation tracking for DML and DDL in the current
session or transaction:

```sql
BEGIN;
SET LOCAL timescaledb.skip_cagg_invalidation = on;
INSERT INTO metrics SELECT * FROM staging_metrics;
COMMIT;
```

Explicitly refresh affected aggregates afterward when they must reflect the
skipped changes.

## Define richer aggregates

Continuous aggregates support non-immutable functions since 2.20.0. Window
functions remain experimental and disabled by default:

```sql
SET timescaledb.enable_cagg_window_functions = on;
```

PostgreSQL set-returning functions such as `unnest` are supported in definitions
since 2.23.0:

```sql
CREATE MATERIALIZED VIEW hourly_tags
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', m.time) AS bucket,
       u.tag,
       count(*) AS samples
FROM metrics AS m
CROSS JOIN LATERAL unnest(m.tags) AS u(tag)
GROUP BY bucket, u.tag
WITH NO DATA;
```

`time_bucket` accepts UUIDv7 and returns timezone-aware timestamps since
2.24.0, enabling continuous aggregates over UUIDv7-partitioned hypertables:

```sql
SELECT time_bucket('1 hour', event_id) AS bucket, count(*)
FROM events
GROUP BY bucket;
```

The timezone argument to `time_bucket_gapfill` must be constant since 2.26.0.
Executor parameters sourced from subquery results are accepted as GapFill
arguments since 2.28.0.

For `DATE` input, `time_bucket` rejects sub-day offsets since 2.27.0.
Continuous-aggregate definitions also reject non-positive bucket widths.

## Add aggregates to an existing view

Since 2.28.0, add an aggregate as a stored generated column without rebuilding
the view:

```sql
ALTER MATERIALIZED VIEW hourly_metrics
ADD COLUMN max_value double precision
GENERATED ALWAYS AS (max(value)) STORED;
```

Existing materialized rows initially contain `NULL`; new rows populate the
column. Use a forced refresh to backfill the required range:

```sql
CALL refresh_continuous_aggregate(
    'hourly_metrics',
    '2025-01-01'::timestamptz,
    '2026-01-01'::timestamptz,
    force => true
);
```

## Configure storage and query rewrites

Continuous aggregates accept PostgreSQL storage parameters outside the
TimescaleDB namespace since 2.25.0:

```sql
ALTER MATERIALIZED VIEW hourly_metrics SET (fillfactor = 90);
```

Automatic `segmentby` and `orderby` defaults for compressed continuous
aggregates changed in 2.25.0. Pin settings explicitly when layout matters.
`ALTER TABLE ... RESET` works on their materialization hypertables since
2.27.0.

An exact aggregation match can be rewritten to a continuous aggregate since
2.27.0. Both rewriting and diagnostics default off:

```sql
SET timescaledb.enable_cagg_rewrites = on;
SET timescaledb.cagg_rewrites_debug_info = on;
```

## Maintain and migrate aggregates

`VACUUM` and `ANALYZE` accept a continuous aggregate name and redirect to the
materialization hypertable since 2.28.0:

```sql
VACUUM hourly_metrics;
ANALYZE hourly_metrics;
```

The deprecated partial aggregate format was removed in 2.25.0. Migrate before
that upgrade with `cagg_migrate(<CONTINUOUS_AGGREGATE_NAME>)`. Replace the
removed experimental policy view and helpers with the Jobs API.

`add_continuous_aggregate_policy` accepts `include_tiered_data` since 2.18.0.
Continuous-aggregate background jobs expose the aggregate name in the jobs
informational view since 2.20.0.
