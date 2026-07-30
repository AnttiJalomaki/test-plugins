# Columnstore and Compression

## Columnstore APIs and Settings

### Prefer columnstore naming

Columnstore-named aliases were introduced in 2.18.0 and the older compression
names were deprecated for removal in the next major release:

| Deprecated | Preferred |
| --- | --- |
| `decompress_chunk` | `convert_to_rowstore` |
| `compress_chunk` | `convert_to_columnstore` |
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

`tsdb` is accepted as a short alias for the `timescaledb` reloption prefix
(since 2.19.0):

```sql
ALTER TABLE metrics SET (tsdb.enable_columnstore = true);
```

Existing hypertables also accept `columnstore` as an alias for
`enable_columnstore` (since 2.20.0). Mixed PostgreSQL and TimescaleDB reloptions
can be set in one statement (since 2.23.0):

```sql
ALTER TABLE metrics SET (
    fillfactor = 90,
    timescaledb.columnstore = true
);
```

### Hypercore access-method lifecycle

The `hypercore` table access method arrived in 2.18.0 and allowed conversion
with `ALTER TABLE ... SET ACCESS METHOD`. It also allowed secondary indexes on
columnstore data, including on `orderby` columns:

```sql
ALTER TABLE metrics SET ACCESS METHOD hypercore;
CREATE INDEX metrics_device_id_idx ON metrics (device_id);
```

That access method was deprecated in 2.21.0 and removed in 2.22.0. Do not use it
in current schemas. An upgrade is blocked while any relation still uses it;
convert every such relation to heap before upgrading.

The `hypercore_use_access_method` GUC introduced in 2.18.0 controlled whether
the access method was the default. It is relevant only while operating a
version where that experimental method exists.

## Compression Algorithms and Layout Selection

### Boolean compression

Custom boolean compression was early access and disabled by default in 2.19.0:

```sql
SET timescaledb.enable_bool_compression = on;
```

Its compressed type cannot be read by earlier versions. Before downgrading data
created with it below that version, run the timescaledb-extras script
`utils/2.19.0-downgrade_new_compression_algorithms.sql`.

Boolean compression became enabled by default in 2.20.0. That release also
improved automatic `segmentby` and `orderby` selection. An explicit `orderby`
prevents selection of a default `segmentby`.

### UUID compression

Specialized UUID compression was experimental, disabled by default, and not
guaranteed to be backward compatible in 2.22.0. It works best for UUIDv7 but
also supports other UUID versions:

```sql
SET timescaledb.enable_uuid_compression = on;
```

UUIDv7 columnstore compression became enabled by default in 2.23.0.

### Automatic layout changes

Automatic `segmentby` selection excludes date and time columns (since 2.24.0).
Compressed continuous aggregates received new automatic `segmentby` and
`orderby` defaults in 2.25.0, so an aggregate relying on automatic selection
can receive a different layout after creation or migration.

Direct Compress defers automatic `segmentby` selection until flush, analyzes
the data, and then sets the default (since 2.27.0). Do not assume that the
layout has been chosen before the flush.

## Columnstore DDL and Recompression

Columnstore tables support foreign keys, compressed chunks support `CHECK`
constraints and columns carrying them, and `ADD COLUMN` can include a unique
constraint (since 2.20.0). Compressed chunks support both dropping `NOT NULL`
(since 2.18.0) and setting `NOT NULL` (since 2.19.0):

```sql
ALTER TABLE metrics ALTER COLUMN device_id SET NOT NULL;
```

Any immutable constant expression can be a default for a compressed column
(since 2.25.0):

```sql
ALTER TABLE metrics ADD COLUMN scale integer DEFAULT (2 * 3);
```

`ALTER COLUMN TYPE` is allowed on a columnstore-enabled hypertable only when it
has no compressed chunks (since 2.24.0):

```sql
ALTER TABLE metrics ALTER COLUMN value TYPE double precision;
```

Updates that would unsafely modify unique columns on compressed chunks are
rejected (since 2.28.0).

### Compression and recompression controls

Segmentwise recompression received a GUC in 2.18.0. Recompression stopped
blocking concurrent `INSERT`, `UPDATE`, and `DELETE` by default in 2.19.0.
`enable_exclusive_locking_recompression` defaults to `OFF`; enable it only to
restore legacy exclusive locking.

Poor-compression-ratio warnings default to enabled through
`timescaledb.enable_compression_ratio_warnings` (since 2.20.0).
`timescaledb.compress_truncate_behaviour` controls final truncation and defaults
to `truncate_only`. Compression can fall back to `DELETE` when the locks needed
for `TRUNCATE` are unavailable, and compression supports a batch-size limit.

Default compression settings are applied at compression time (since 2.22.0).
Compression settings support `ALTER TABLE RESET`; a downgrade is blocked when
the `orderby` setting is `NULL`.

`convert_to_columnstore` accepts `recompress := true` for in-memory
recompression (since 2.24.0), controlled by
`timescaledb.enable_in_memory_recompression`:

```sql
SET timescaledb.enable_in_memory_recompression = on;
SELECT convert_to_columnstore('metrics_chunk'::regclass, recompress := true);
```

In-memory recompression supports unordered chunks, and recompression is allowed
after `orderby` or index settings change (since 2.25.0). `VACUUM FULL` also
recompresses affected chunks, so budget for recompression work.

## Direct Compress

Direct Compress is an experimental path that compresses input in memory and
writes it directly to disk rather than waiting for a background compression
job.

### `COPY`

Direct compression for `COPY` was introduced off by default in 2.21.0. Batch
sorting defaults on; client-sorted mode defaults off and must be enabled only
when input is correctly sorted:

```sql
SET timescaledb.enable_direct_compress_copy = on;
SET timescaledb.enable_direct_compress_copy_sort_batches = on;
SET timescaledb.enable_direct_compress_copy_client_sorted = off;
```

### `INSERT`

Direct compression for `INSERT`, including insert directly into a chunk, was
added in 2.23.0:

```sql
SET timescaledb.enable_direct_compress_insert = on;
SET timescaledb.enable_direct_compress_insert_sort_batches = on;
SET timescaledb.enable_direct_compress_insert_client_sorted = off;
```

Client-ordered Direct Compress had a data-loss path for an
`INSERT ... SELECT` whose source was a compressed hypertable. Avoid that
combination before 2.26.0; upgrade before using it.

### Continuous-aggregate integration and memory

Direct Compress supports hypertables feeding continuous aggregates and records
their invalidation ranges at transaction commit (since 2.24.0).
`timescaledb.direct_compress_copy_tuple_sort_limit` and
`timescaledb.direct_compress_insert_tuple_sort_limit` cap how many tuples the
respective paths sort at once.

Continuous-aggregate refresh can use Direct Compress (since 2.25.0), disabled
by default:

```sql
SET timescaledb.enable_direct_compress_on_cagg_refresh = on;
```

Refresh policies for compressed continuous aggregates can run compression as
part of the refresh (since 2.27.0).

## Sparse Indexes and Columnar Scans

### Bloom sparse indexes

Columnstore chunks create `bloom1` sparse indexes by default (since 2.20.0).
Disable automatic creation with:

```sql
SET timescaledb.enable_sparse_index_bloom = off;
```

Sparse indexes can be configured explicitly with `ALTER TABLE`, including over
multiple columns (since 2.22.0).

Bloom indexes on chunks compressed before 2.24.0 are disabled after upgrade
because the former hash could vary with build options and silently miss
matching rows after a package change. Decompress and recompress those chunks.
Chunks compressed after the upgrade need no action.

The hash did not change for the official APT package on AMD64. In that narrow
case, legacy indexes may instead be enabled for `SELECT` in server
configuration:

```ini
timescaledb.read_legacy_bloom1_v1 = on
```

Composite bloom indexes are created by default (since 2.26.0), controlled by
`timescaledb.enable_composite_bloom_indexes`, whose default is `true`.
Multi-column predicates can be pushed into compressed scans for both `SELECT`
and `UPSERT`. `EXPLAIN` reports batch-pruning and false-positive statistics.

TimescaleDB 2.27.0 cannot automatically use composite bloom filters created by
2.26.0 because their metadata names changed. Run the timescaledb-extras script
`utils/2.27.x-fix-composite-bloom-columns.sql`; it updates only catalogs and
does not require recompression.

Bloom sparse indexes on compressed `int2` columns can omit matching rows. The
2.27.0 upgrade is blocked while affected indexes exist; drop them first.

### Scan filter pushdown and size reporting

`timescaledb.enable_columnar_scan_filter_pushdown` controls pushing
columnar-scan filters into the compressed scan and defaults to on (since
2.27.0):

```sql
SET timescaledb.enable_columnar_scan_filter_pushdown = off;
```

`compressed_data_column_size` returns `bigint` (since 2.27.0). Update explicit
casts and client decoders that assumed a narrower integer.
