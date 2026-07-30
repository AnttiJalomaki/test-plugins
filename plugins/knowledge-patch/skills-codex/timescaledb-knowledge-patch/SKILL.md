---
name: timescaledb-knowledge-patch
description: TimescaleDB
version: 2.28.0
license: MIT
metadata:
  author: Nevaberry
---


# TimescaleDB Knowledge Patch

Use this skill when designing, upgrading, or troubleshooting TimescaleDB
schemas, columnstore storage, continuous aggregates, chunk operations, direct
compression, or background jobs.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-compatibility.md](references/upgrades-and-compatibility.md) | Upgrade blockers, PostgreSQL support, removals, migrations, and downgrade safety |
| [hypertables-and-chunks.md](references/hypertables-and-chunks.md) | Declarative hypertables, reloptions, constraints, triggers, partitioning, and chunk lifecycle |
| [columnstore-and-ingest.md](references/columnstore-and-ingest.md) | Columnstore APIs, compression, sparse indexes, Direct Compress, recompression, and scan controls |
| [continuous-aggregates.md](references/continuous-aggregates.md) | Refresh, invalidations, policies, query features, GapFill, and maintenance |
| [jobs-and-observability.md](references/jobs-and-observability.md) | Background jobs, diagnostics, catalogs, memory, and operational controls |

## Upgrade checks first

### Remove the obsolete Hypercore access method

The experimental `hypercore` table access method was deprecated and then
removed. An upgrade is blocked while a relation still uses it. Before upgrading,
find every such relation and convert it back to heap:

```sql
DO $$
DECLARE
    relid regclass;
BEGIN
    FOR relid IN
        SELECT cl.oid
        FROM pg_class AS cl
        JOIN pg_am AS am ON am.oid = cl.relam
        WHERE am.amname = 'hypercore'
    LOOP
        EXECUTE format('ALTER TABLE %s SET ACCESS METHOD heap', relid);
    END LOOP;
END
$$;
```

Do not confuse the removed access method with current columnstore functionality.

### Respect PostgreSQL support cutoffs

- Upgrade PostgreSQL before moving beyond TimescaleDB 2.19 if the server still
  runs PostgreSQL 14.
- TimescaleDB 2.28.x is the final patch series supporting PostgreSQL 15.
  TimescaleDB 2.29 supports PostgreSQL 16, 17, and 18.

### Clear bloom-index upgrade blockers

- Before a 2.27 upgrade, drop bloom sparse indexes on compressed `int2`
  columns; affected indexes can omit matching rows and block the upgrade.
- Composite bloom filters created by 2.26 need the timescaledb-extras
  `utils/2.27.x-fix-composite-bloom-columns.sql` catalog migration.
- Bloom indexes on chunks compressed before 2.24 normally require
  decompress/recompress because the old hash could change between builds.
  Official APT AMD64 packages may instead enable
  `timescaledb.read_legacy_bloom1_v1` for reads.

### Replace removed facilities

- Adaptive chunking is removed; stop relying on adaptive chunk sizing.
- Replace `time_bucket_ng` and dependencies on `_timescaledb_debug`.
- Replace the deprecated partial continuous-aggregate format with
  `cagg_migrate(...)`.
- Replace experimental policy helpers in `timescaledb_experimental` with the
  Jobs API.
- Stop querying `_timescaledb_catalog.chunk_constraint`; its compatibility view
  is temporary, so use informational views.
- Use trigger-based invalidation. The experimental WAL-based
  continuous-aggregate invalidation path was removed.

## Prefer columnstore terminology

Use the current names in new code:

| Deprecated name | Current name |
| --- | --- |
| `compress_chunk` | `convert_to_columnstore` |
| `decompress_chunk` | `convert_to_rowstore` |
| `add_compression_policy` | `add_columnstore_policy` |
| `remove_compression_policy` | `remove_columnstore_policy` |
| `hypertable_compression_stats` | `hypertable_columnstore_stats` |
| `chunk_compression_stats` | `chunk_columnstore_stats` |
| `hypertable_compression_settings` | `hypertable_columnstore_settings` |
| `chunk_compression_settings` | `chunk_columnstore_settings` |
| `compression_settings` | `columnstore_settings` |
| `timescaledb.compress` | `timescaledb.enable_columnstore` |
| `timescaledb.compress_segmentby` | `timescaledb.segmentby` |
| `timescaledb.compress_orderby` | `timescaledb.orderby` |

The short `tsdb` prefix is accepted in `WITH` and `SET` clauses:

```sql
ALTER TABLE metrics SET (tsdb.enable_columnstore = true);
```

## Create declarative hypertables

Create a hypertable and enable columnstore in the same statement:

```sql
CREATE TABLE metrics (
    time timestamptz NOT NULL,
    device_id bigint,
    value double precision
) WITH (
    tsdb.hypertable,
    tsdb.partition_column = 'time',
    tsdb.columnstore
);
```

The partition column can be omitted when inference is appropriate. Enabling
columnstore in declarative DDL also creates a columnstore policy. Use
`ALTER TABLE ONLY` when changed reloptions should affect only future chunks:

```sql
ALTER TABLE ONLY metrics
SET (timescaledb.orderby = 'time DESC');
```

## Refresh continuous aggregates in bounded work

Manual refresh accepts batching controls:

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

Refresh policies default `buckets_per_batch` to `10`. Non-overlapping ranges
can refresh concurrently, incremental policies are enabled by default, and
forced refreshes consume their invalidations.

## Add an aggregate without rebuilding

Add an aggregate as a stored generated column. Existing materialized rows begin
as `NULL`; a forced refresh backfills the desired range:

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

`VACUUM` and `ANALYZE` accept the continuous aggregate name and redirect to its
materialization hypertable.

## Treat Direct Compress as opt-in

Direct Compress can compress `COPY`, `INSERT`, and continuous-aggregate refresh
input in memory. Enable only the specific path required. Client-sorted modes
must be used only when input ordering is correct.

```sql
SET timescaledb.enable_direct_compress_copy = on;
SET timescaledb.enable_direct_compress_insert = on;
SET timescaledb.enable_direct_compress_on_cagg_refresh = on;
```

Before using client-ordered Direct Compress with `INSERT ... SELECT` from a
compressed hypertable, upgrade past the earlier data-loss path. Use the tuple
sort limits for `COPY` and `INSERT` to bound in-memory sorting.

## Bulk-load invalidation control

`timescaledb.skip_cagg_invalidation` defaults off. It can reduce bulk-load
overhead, but skipped changes require an explicit refresh:

```sql
BEGIN;
SET LOCAL timescaledb.skip_cagg_invalidation = on;
INSERT INTO metrics SELECT * FROM staging_metrics;
COMMIT;
```

## Verify version-sensitive defaults

- Composite bloom indexes are enabled by default.
- UUIDv7 compression is enabled by default.
- Columnar scan filter pushdown is enabled by default.
- Incremental refresh policies are enabled by default.
- Nonblocking recompression is the default; enable exclusive locking only to
  restore legacy behavior.
- Automatic `segmentby` and `orderby` choices can change with data and context.
  Specify them explicitly when layout stability matters.

Consult the topic references before writing migrations or depending on a GUC,
private catalog object, experimental behavior, or automatically selected
columnstore layout.
