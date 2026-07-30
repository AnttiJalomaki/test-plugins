# Columnstore and ingest

## Use the columnstore APIs

The current API uses columnstore terms. The full deprecated-name mapping is in
[upgrades-and-compatibility.md](upgrades-and-compatibility.md). Secondary
indexes, including indexes over `orderby` columns, have been supported on
columnstore data since 2.18.0:

```sql
CREATE INDEX metrics_device_id_idx ON metrics (device_id);
```

Columnstore tables support foreign keys, while compressed chunks support
`CHECK`, `SET NOT NULL`, `DROP NOT NULL`, and unique constraints on added
columns as described in the hypertable reference.

## Control compression and recompression

### Locking, truncation, and batch size

Recompression has been nonblocking by default since 2.19.0, allowing concurrent
`INSERT`, `UPDATE`, and `DELETE`. Set
`timescaledb.enable_exclusive_locking_recompression` to `on` only to restore
legacy exclusive locking.

Poor-compression-ratio warnings are enabled by default since 2.20.0 through
`timescaledb.enable_compression_ratio_warnings`.
`timescaledb.compress_truncate_behaviour` defaults to `truncate_only`;
compression can use `DELETE` if the locks required for `TRUNCATE` are
unavailable. Compression also supports batch-size limiting.

New GUCs in 2.18.0 control the `hypercore_use_access_method` default and
segmentwise recompression. The access-method setting belongs to the obsolete
experimental Hypercore path; consult the upgrade reference before using it.

### In-memory and layout-changing recompression

Since 2.24.0, `convert_to_columnstore` accepts `recompress := true` for entirely
in-memory recompression:

```sql
SET timescaledb.enable_in_memory_recompression = on;
SELECT convert_to_columnstore('metrics_chunk'::regclass, recompress := true);
```

Since 2.25.0, in-memory recompression supports unordered chunks and
recompression is allowed after `orderby` or index settings change. `VACUUM
FULL` also recompresses affected chunks and can therefore include substantial
recompression work.

### Automatic layout

- Default compression settings are applied when compression runs rather than
  earlier in the table lifecycle (since 2.22.0). Compression settings also
  support `ALTER TABLE RESET`; downgrades are blocked while `orderby` is
  `NULL`.
- The boolean compression algorithm became enabled by default in 2.20.0.
- Automatic `segmentby` and `orderby` selection improved in 2.20.0. An explicit
  `orderby` prevents choosing a default `segmentby`.
- Automatic `segmentby` excludes date and time columns since 2.24.0.
- Compressed continuous aggregates received new automatic `segmentby` and
  `orderby` defaults in 2.25.0.
- Direct Compress defers automatic `segmentby` selection, analyzes the data,
  and chooses the default during flush since 2.27.0.

Specify layout explicitly whenever a stable physical layout is required.

## Configure sparse indexes

Columnstore chunks create `bloom1` sparse indexes by default since 2.20.0:

```sql
SET timescaledb.enable_sparse_index_bloom = off;
```

Since 2.22.0, `ALTER TABLE` can configure sparse indexes explicitly, including
multi-column indexes, instead of relying only on internal heuristics.

Composite bloom indexes are created by default since 2.26.0 and controlled by
`timescaledb.enable_composite_bloom_indexes`, whose default is `true`.
Multi-column predicates can be pushed into compressed scans for `SELECT` and
`UPSERT`; `EXPLAIN` reports batch-pruning and false-positive statistics.

Before an upgrade, check the bloom-index blockers and metadata migrations in
the upgrade reference.

## Use UUID compression

Specialized UUID compression was experimental and off by default in 2.22.0:

```sql
SET timescaledb.enable_uuid_compression = on;
```

It works best with UUIDv7 but supports other UUID versions, and its early
format did not guarantee backward compatibility. UUIDv7 compression became
enabled by default in 2.23.0.

## Configure Direct Compress

Direct Compress writes in-memory compressed ingest directly to disk rather than
waiting for a background compression job.

### `COPY`

The tech-preview `COPY` path arrived in 2.21.0 and is off by default. Batch
sorting defaults on; client-sorted mode defaults off and is safe only when the
input is correctly sorted:

```sql
SET timescaledb.enable_direct_compress_copy = on;
SET timescaledb.enable_direct_compress_copy_sort_batches = on;
SET timescaledb.enable_direct_compress_copy_client_sorted = off;
```

### `INSERT`

Direct Compress supports `INSERT`, including direct chunk inserts, since
2.23.0:

```sql
SET timescaledb.enable_direct_compress_insert = on;
SET timescaledb.enable_direct_compress_insert_sort_batches = on;
SET timescaledb.enable_direct_compress_insert_client_sorted = off;
```

The client-ordered path could lose data for `INSERT ... SELECT` from a
compressed hypertable before its fix in 2.26.0. Avoid that combination on
earlier versions.

### Continuous-aggregate sources and refresh

Since 2.24.0, Direct Compress supports hypertables that feed continuous
aggregates and records invalidation ranges for compressed batches at transaction
commit. `timescaledb.direct_compress_copy_tuple_sort_limit` and
`timescaledb.direct_compress_insert_tuple_sort_limit` cap tuples sorted at once.

Continuous-aggregate refresh has an experimental Direct Compress path since
2.25.0:

```sql
SET timescaledb.enable_direct_compress_on_cagg_refresh = on;
```

Refresh policies for compressed continuous aggregates can perform compression
during refresh since 2.27.0.

## Tune compressed scans and statistics

Vectorized aggregation and compressed-table `WHERE` comparisons follow
PostgreSQL NaN behavior after fixes in 2.18.0.

`timescaledb.enable_columnar_scan_filter_pushdown` defaults on since 2.27.0:

```sql
SET timescaledb.enable_columnar_scan_filter_pushdown = off;
```

`compressed_data_column_size` returns `bigint` since 2.27.0. Update SQL casts
and client decoders that assumed a narrower type.

`timescaledb.stats_max_chunks` sets the per-database capacity of the in-memory
compressed-chunk statistics cache since 2.28.0. It defaults to `1024`; zero
disables the cache:

```sql
SET timescaledb.stats_max_chunks = 0;
```
