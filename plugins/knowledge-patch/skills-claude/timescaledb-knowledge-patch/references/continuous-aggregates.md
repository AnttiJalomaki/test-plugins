# Continuous Aggregates

## Refresh APIs and Scheduling

`refresh_continuous_aggregate` accepts `force` (since 2.18.0). A forced refresh
consumes its associated invalidations rather than leaving them pending (since
2.26.0).

Continuous-aggregate policies can set `include_tiered_data` (since 2.18.0).
They can split refresh work into smaller batches, and refresh newest data before
older data, reducing memory and disk pressure while making recent results
current sooner (since 2.19.0). The `refresh_newest_first` policy argument
explicitly controls that ordering (since 2.20.0).

Non-overlapping ranges can refresh concurrently, and concurrent refresh
policies can be created (since 2.21.0). Incremental policy refresh is enabled by
default. The policy default for `buckets_per_batch` is `10` (since 2.25.0), so
policies use smaller transactions unless explicitly overridden.

Manual refresh can also batch its work (since 2.28.0):

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

Manual batching uses `buckets_per_batch`, `max_batches_per_execution`, and
`refresh_newest_first`. Invalidation-log processing uses a lighter lock, so
that phase no longer blocks unrelated operations on the same aggregate.

Refresh policies for compressed continuous aggregates can perform compression
during refresh (since 2.27.0). The experimental Direct Compress refresh path is
documented in
[Columnstore and Compression](columnstore-and-compression.md).

## Query Definitions and Rewrites

Continuous aggregates can use non-immutable functions (since 2.20.0).
Window-function support is experimental and disabled by default:

```sql
SET timescaledb.enable_cagg_window_functions = on;
```

Definitions can use set-returning functions such as `unnest` (since 2.23.0):

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

`time_bucket` accepts UUIDv7 and returns timezone-aware timestamps (since
2.24.0), enabling continuous aggregates over a UUIDv7-partitioned hypertable:

```sql
SELECT time_bucket('1 hour', event_id) AS bucket, count(*)
FROM events
GROUP BY bucket;
```

Queries whose aggregation exactly matches a continuous aggregate can be
rewritten to use it (since 2.27.0). Rewriting and diagnostics are independently
disabled by default:

```sql
SET timescaledb.enable_cagg_rewrites = on;
SET timescaledb.cagg_rewrites_debug_info = on;
```

## Altering and Maintaining Aggregates

Continuous aggregates accept ordinary PostgreSQL storage parameters (since
2.25.0):

```sql
ALTER MATERIALIZED VIEW hourly_metrics SET (fillfactor = 90);
```

`ALTER TABLE ... RESET` works on materialization hypertables (since 2.27.0), so
their reloptions can be restored to defaults.

An aggregate can be added to an existing continuous aggregate as a stored
generated column without rebuilding the view (since 2.28.0). Existing
materialized rows initially contain `NULL`; newly materialized rows populate
the column. Force a refresh over the desired historical range to backfill:

```sql
ALTER MATERIALIZED VIEW hourly_metrics
ADD COLUMN max_value double precision
GENERATED ALWAYS AS (max(value)) STORED;

CALL refresh_continuous_aggregate(
    'hourly_metrics',
    '2025-01-01'::timestamptz,
    '2026-01-01'::timestamptz,
    force => true
);
```

`VACUUM` and `ANALYZE` accept the continuous-aggregate name and redirect to the
materialization hypertable (since 2.28.0):

```sql
VACUUM hourly_metrics;
ANALYZE hourly_metrics;
```

## Invalidation Processing

APIs for the hypertable invalidation log and materialization invalidations are
available (since 2.20.0).

Hypertable invalidations can be processed by an explicit function or scheduled
policy, with an option to leave them unprocessed (since 2.21.0). Invalidation
processing supports continuous aggregates that involve multiple hypertables
(since 2.22.0).

### WAL-based invalidation was temporary

The tech-preview `timescaledb.invalidate_using` option introduced in 2.22.0
selected trigger collection or WAL collection through logical decoding. When
omitted on an aggregate, it inherited the source hypertable's method.

The WAL processor used these defaults:

- `cagg_processing_wal_batch_size`: `10000`
- `cagg_processing_low_work_mem`: `38.4MB`
- `cagg_processing_high_work_mem`: `51.2MB`

An explicit enablement GUC,
`timescaledb.enable_cagg_wal_based_invalidation`, followed in 2.23.0. The whole
experimental WAL path was removed in 2.25.0. Current deployments must use
trigger-based invalidation; remove the option and GUC from configuration.

### Deliberately skipping invalidations

`timescaledb.skip_cagg_invalidation` skips invalidation tracking for DML and DDL
in the current session or transaction and defaults to off (since 2.28.0):

```sql
BEGIN;
SET LOCAL timescaledb.skip_cagg_invalidation = on;
INSERT INTO metrics SELECT * FROM staging_metrics;
COMMIT;
```

This can reduce bulk-load overhead. Changes made while it is enabled are not
tracked, so explicitly refresh affected aggregates when they must be current.

## Removed Formats and Policy Helpers

The deprecated partial continuous-aggregate format was slated for removal after
2.24.0. Migrate any remaining aggregate:

```sql
SELECT cagg_migrate('<CONTINUOUS_AGGREGATE_NAME>');
```

Replace the experimental `timescaledb_experimental.policies` view and
`add_policies`, `alter_policies`, `show_policies`, `remove_policies`, and
`remove_all_policies` functions with the Jobs API. These helpers were likewise
slated for removal after 2.24.0.
