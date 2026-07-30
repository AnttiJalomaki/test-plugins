# Upgrades and compatibility

## Resolve upgrade blockers

### Hypercore access method

The `hypercore` table access method arrived as an experimental conversion target
in 2.18.0:

```sql
ALTER TABLE metrics SET ACCESS METHOD hypercore;
```

It was deprecated in 2.21.0 and removed in 2.22. An upgrade to 2.22 or later is
blocked while any relation still uses it. Convert every such relation to heap
before upgrading:

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

Current columnstore behavior does not require the removed access method.

### Bloom sparse indexes

- Bloom indexes on chunks compressed before 2.24.0 are disabled after upgrade
  because their former hash scheme could vary with build options and silently
  miss rows after a package change. Decompress and recompress affected chunks.
  Chunks compressed after upgrading need no action.
- For official APT packages on AMD64, the hash did not change. Those deployments
  can instead set `timescaledb.read_legacy_bloom1_v1 = on` in server
  configuration to use legacy indexes for `SELECT`.
- The 2.27.0 upgrade is blocked by bloom sparse indexes on compressed `int2`
  columns because such indexes can omit matching rows. Drop them manually
  before upgrading.
- Composite bloom filters created by 2.26 cannot be used automatically by 2.27
  because metadata naming changed. Run timescaledb-extras
  `utils/2.27.x-fix-composite-bloom-columns.sql`. It changes only catalog
  metadata and does not require recompression.

### PostgreSQL versions

- TimescaleDB 2.19.0 is the final minor release supporting PostgreSQL 14. Move
  to PostgreSQL 15, 16, or 17 before upgrading TimescaleDB beyond it.
- TimescaleDB 2.23.0 supports PostgreSQL 15, 16, 17, and 18, including all
  existing features on PostgreSQL 18. At that point PostgreSQL 15 support was
  announced through June 2026 but no first unsupported TimescaleDB release had
  been named.
- TimescaleDB 2.28.0 resolves that boundary: the 2.28.x series is the last to
  support PostgreSQL 15, while 2.29 supports PostgreSQL 16, 17, and 18.

## Replace removed and deprecated interfaces

### Columnstore terminology

Columnstore aliases introduced in 2.18.0 replace compression-named interfaces,
which are deprecated for removal in the next major release:

| Deprecated | Replacement |
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

### Continuous-aggregate and experimental APIs

- Before 2.25.0, migrate a deprecated partial-format continuous aggregate with
  `cagg_migrate(<CONTINUOUS_AGGREGATE_NAME>)`.
- Replace the experimental `timescaledb_experimental.policies` view and
  `add_policies`, `alter_policies`, `show_policies`, `remove_policies`, and
  `remove_all_policies` with the Jobs API.
- The experimental WAL-based invalidation facility was removed in 2.25.0.
  Return affected continuous aggregates to trigger-based invalidation.
- `time_bucket_ng` and the `_timescaledb_debug` schema were removed in 2.25.0.
  Migrate callers and tooling.

### Catalog and chunk interfaces

- `_timescaledb_functions.create_chunk_table` was removed in 2.20.0. Do not
  build chunk-management code around this internal helper.
- `_timescaledb_catalog.chunk_constraint` ceased to be a table in 2.28.0. A
  temporary compatibility view preserves current queries, but that view will
  also be removed. Migrate integrations to TimescaleDB informational views.
- Adaptive chunking was removed in 2.28.0 as a backward-incompatible change.
  Stop relying on adaptive chunk sizing before upgrading.

## Plan downgrade-safe storage

The custom boolean compression algorithm was early access and off by default in
2.19.0. Older versions cannot read its compressed type. Before downgrading data
written with it to a pre-2.19 release, run timescaledb-extras
`utils/2.19.0-downgrade_new_compression_algorithms.sql`.

```sql
SET timescaledb.enable_bool_compression = on;
```

The algorithm became enabled by default in 2.20.0. Specialized UUID compression
was experimental and off by default in 2.22.0, with backward compatibility not
guaranteed; it became enabled by default for UUIDv7 in 2.23.0. Compression
settings support `ALTER TABLE RESET`, but downgrades are blocked when the
`orderby` setting is `NULL` (since 2.22.0).

## Change container sources

TimescaleDB stopped building Bitnami images in 2.18.0. Container deployments
should use the official `timescale/timescaledb-ha` image.

## Account for corrected unsafe behavior

- Client-ordered Direct Compress could lose data when an `INSERT ... SELECT`
  source was itself a compressed hypertable. This was fixed in 2.26.0; avoid
  the combination on earlier versions.
- Cross-type comparisons against partition columns could return wrong results
  or crash before the 2.26.0 fix.
- Updates that would unsafely modify unique columns on compressed chunks are
  rejected in 2.28.0 rather than being allowed to proceed.
