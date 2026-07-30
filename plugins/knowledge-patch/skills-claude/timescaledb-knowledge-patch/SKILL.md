---
name: timescaledb-knowledge-patch
description: TimescaleDB
version: 2.28.0
license: MIT
metadata:
  author: Nevaberry
---


# TimescaleDB Knowledge Patch

Load this skill when writing, reviewing, debugging, or upgrading TimescaleDB
schema, SQL, configuration, background jobs, compression, columnstore, chunks,
or continuous aggregates.

Prefer public APIs and informational views over private catalog objects. Check
the installed TimescaleDB and PostgreSQL versions before applying
version-dependent advice.

## Reference Index

| Reference | Topics |
| --- | --- |
| [Columnstore and Compression](references/columnstore-and-compression.md) | Columnstore APIs, compression settings, Hypercore, Direct Compress, sparse indexes, recompression, and scan behavior |
| [Continuous Aggregates](references/continuous-aggregates.md) | Refresh controls, policies, invalidations, query features, maintenance, and migrations |
| [Hypertables, Chunks, and DDL](references/hypertables-chunks-and-ddl.md) | Declarative creation, reloptions, constraints, chunk operations, partitioning, triggers, and publications |
| [Operations, Jobs, and Query Behavior](references/operations-jobs-and-query-behavior.md) | Images, jobs, event triggers, memory and cache controls, GapFill, query correctness, and observability |
| [Upgrades and Compatibility](references/upgrades-and-compatibility.md) | PostgreSQL compatibility, upgrade blockers, removals, downgrade preparation, and catalog migrations |

## Upgrade Blockers and Removals

### Remove the `hypercore` access method before upgrading

The experimental `hypercore` table access method is gone. An upgrade is blocked
while a relation still uses it. Find those relations and return them to heap:

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

Do this before attempting the extension upgrade. See
[Upgrades and Compatibility](references/upgrades-and-compatibility.md).

### Clear affected `int2` bloom indexes

An upgrade can be blocked by bloom sparse indexes on compressed `int2` columns
because those indexes can omit matching rows. Drop affected indexes before the
upgrade. Separately, composite bloom filters created with old metadata need the
timescaledb-extras catalog migration described in
[Upgrades and Compatibility](references/upgrades-and-compatibility.md).

### Rebuild legacy bloom indexes when package hashing changed

Old bloom indexes may silently miss rows after a package change. Decompress and
recompress affected chunks to rebuild them. The official APT AMD64 package has
a narrowly scoped server setting for reading legacy indexes; do not treat that
exception as a general migration.

### Replace removed features

Do not build new dependencies on these removed or transitional interfaces:

- Adaptive chunking has been removed.
- WAL-based continuous-aggregate invalidation has been removed; use
  trigger-based invalidation.
- `time_bucket_ng` and the `_timescaledb_debug` schema have been removed.
- The partial continuous-aggregate format must be migrated with
  `cagg_migrate(...)`.
- `_timescaledb_catalog.chunk_constraint` is only temporarily represented by a
  compatibility view; move integrations to informational views.
- The experimental policy helpers and view are replaced by the Jobs API.

The detailed timing and replacement guidance is in
[Upgrades and Compatibility](references/upgrades-and-compatibility.md).

## Columnstore API Migration

Use columnstore terminology in current code:

| Deprecated name | Preferred name |
| --- | --- |
| `decompress_chunk` | `convert_to_rowstore` |
| `compress_chunk` | `convert_to_columnstore` |
| `add_compression_policy` | `add_columnstore_policy` |
| `remove_compression_policy` | `remove_columnstore_policy` |
| `hypertable_compression_stats` | `hypertable_columnstore_stats` |
| `chunk_compression_stats` | `chunk_columnstore_stats` |
| `timescaledb.compress` | `timescaledb.enable_columnstore` |
| `timescaledb.compress_segmentby` | `timescaledb.segmentby` |
| `timescaledb.compress_orderby` | `timescaledb.orderby` |

Settings and information-view renames are listed completely in
[Columnstore and Compression](references/columnstore-and-compression.md).

## Continuous-Aggregate Refreshes

### Batch manual refresh work

Manual refresh supports batching, an execution cap, and newest-first ordering:

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

Policy refreshes are incremental by default and default to ten buckets per
batch. Non-overlapping ranges can refresh concurrently.

### Backfill a newly added aggregate

Add an aggregate as a stored generated column. Existing materialized rows start
as `NULL`; a forced refresh fills the selected history:

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

### Suppress invalidations only with an explicit refresh plan

For a bulk operation, `timescaledb.skip_cagg_invalidation` can avoid invalidation
tracking:

```sql
BEGIN;
SET LOCAL timescaledb.skip_cagg_invalidation = on;
INSERT INTO metrics SELECT * FROM staging_metrics;
COMMIT;
```

Changes made this way are not tracked. Explicitly refresh every affected
continuous aggregate afterward when it must be current.

## Declarative Hypertables and Columnstore

The declarative API can create a hypertable with columnstore enabled:

```sql
CREATE TABLE metrics (
    time timestamptz NOT NULL,
    device_id bigint,
    value double precision
) WITH (
    tsdb.hypertable,
    tsdb.columnstore
);
```

`tsdb` is an alias for the `timescaledb` reloption prefix. A declarative
columnstore hypertable automatically receives a columnstore policy. Use
`ALTER TABLE ONLY` when changed reloptions should affect future chunks only:

```sql
ALTER TABLE ONLY metrics
SET (timescaledb.orderby = 'time DESC');
```

See [Hypertables, Chunks, and DDL](references/hypertables-chunks-and-ddl.md)
for constraints, triggers, split/merge behavior, and partitioning rules.

## Direct Compress Safety

Direct Compress paths are experimental and independently controlled for
`COPY`, `INSERT`, and continuous-aggregate refresh. Client-sorted modes are safe
only when input is genuinely sorted. Earlier implementations had a data-loss
path for client-ordered `INSERT ... SELECT` from a compressed hypertable;
upgrade before using that combination.

Direct Compress can record continuous-aggregate invalidations at commit and can
compress refresh output. Tuple-sort limits bound in-memory sorting. Review every
GUC and default in
[Columnstore and Compression](references/columnstore-and-compression.md)
before enabling it.

## Sparse-Index and Scan Controls

Columnstore uses bloom sparse indexes, including composite indexes. Keep these
controls distinct:

- `timescaledb.enable_sparse_index_bloom` controls default bloom creation.
- `timescaledb.enable_composite_bloom_indexes` defaults to `true`.
- `timescaledb.enable_columnar_scan_filter_pushdown` defaults to on.
- `timescaledb.read_legacy_bloom1_v1` is a narrow upgrade compatibility
  setting, not a normal default.

Use `EXPLAIN` to inspect batch pruning and false-positive statistics. Detailed
upgrade caveats are in
[Columnstore and Compression](references/columnstore-and-compression.md).

## Operational Checks

- Use the official `timescale/timescaledb-ha` image rather than a retired
  Bitnami build.
- Treat `VACUUM FULL` on columnstore data as potentially including
  recompression work.
- `VACUUM` and `ANALYZE` on a continuous aggregate are redirected to its
  materialization hypertable.
- Keep `time_bucket_gapfill` timezone arguments constant.
- Leave the expert default-chunk-interval GUC unchanged unless specifically
  directed.
- When disabling continuous-aggregate invalidations or compressed statistics
  caching, document the compensating refresh or observability plan.

Use [Operations, Jobs, and Query Behavior](references/operations-jobs-and-query-behavior.md)
for configuration defaults and query-behavior details.
