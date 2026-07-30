# Upgrades and Compatibility

## PostgreSQL Compatibility

TimescaleDB 2.19.0 is the last minor release that supports PostgreSQL 14.
Upgrade PostgreSQL before moving beyond that series; later releases support
PostgreSQL 15, 16, and 17.

TimescaleDB 2.23.0 adds PostgreSQL 18 support with existing features available
there. It supports PostgreSQL 15, 16, 17, and 18. PostgreSQL 15 support was
announced through June 2026, although the first release to remove it was not
then specified.

TimescaleDB 2.28.0 is the final minor series supporting PostgreSQL 15. Its
2.28.x patches retain PostgreSQL 15 support; TimescaleDB 2.29 supports only
PostgreSQL 16, 17, and 18. Upgrade PostgreSQL before moving beyond 2.28.x.

## Pre-upgrade Checklist

### Remove the Hypercore access method

The experimental `hypercore` table access method appeared in 2.18.0, was
deprecated in 2.21.0, and was removed in 2.22.0. An upgrade to 2.22 or later is
blocked while a relation still uses it.

Convert all such relations back to heap:

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

### Handle bloom sparse indexes

Bloom indexes on chunks compressed before 2.24.0 used a hash that could vary
with build options and silently miss matching rows after a package change. They
are disabled on upgrade. Decompress and recompress affected chunks to rebuild
them; chunks compressed afterward need no work.

For the official APT package on AMD64, the hash did not change. That narrow
deployment may enable legacy indexes for `SELECT` in server configuration:

```ini
timescaledb.read_legacy_bloom1_v1 = on
```

Do not apply this exception to other packages or architectures.

Composite bloom filters created in 2.26.0 use metadata names that 2.27.0 does
not automatically recognize. Run the timescaledb-extras migration
`utils/2.27.x-fix-composite-bloom-columns.sql`. It only renames catalog columns
and requires no recompression.

Bloom sparse indexes on compressed `int2` columns can omit matching rows. The
2.27.0 upgrade is blocked while affected indexes exist; drop them manually
before upgrading.

### Migrate old continuous aggregates and policies

The deprecated partial continuous-aggregate format was scheduled for removal
after 2.24.0. Migrate every remaining aggregate:

```sql
SELECT cagg_migrate('<CONTINUOUS_AGGREGATE_NAME>');
```

Replace `timescaledb_experimental.policies` and the experimental
`add_policies`, `alter_policies`, `show_policies`, `remove_policies`, and
`remove_all_policies` helpers with the Jobs API.

### Stop relying on adaptive chunking

Adaptive chunking was removed as a backward-incompatible change in 2.28.0.
Move to an explicit chunk interval before upgrading.

## Downgrade Preparation

### Boolean compressed data

Custom boolean compression was early access in 2.19.0. Earlier versions cannot
read its compressed type. Before downgrading data created with that algorithm,
run:

`utils/2.19.0-downgrade_new_compression_algorithms.sql`

The script is provided by timescaledb-extras.

### Compression settings

Default compression settings are applied at compression time and can be reset
with `ALTER TABLE RESET` (since 2.22.0). A downgrade is blocked when an
`orderby` setting is `NULL`; resolve those settings before attempting it.

### Experimental UUID compression

UUID compression was introduced in 2.22.0 without a backward-compatibility
guarantee. Treat data written with that early implementation as a downgrade
risk and test conversion before rolling back.

## Removed or Transitional Interfaces

### Columnstore names replace compression names

Columnstore-named aliases were introduced in 2.18.0, with old compression names
deprecated for removal in the next major release. Migrate function, view, and
reloption names. The full mapping is in
[Columnstore and Compression](columnstore-and-compression.md).

### WAL-based continuous-aggregate invalidation

WAL-based invalidation was experimental in 2.22.0, gained an explicit
enablement GUC in 2.23.0, and was removed in 2.25.0. Return deployments to
trigger-based invalidation and remove
`timescaledb.enable_cagg_wal_based_invalidation` and WAL-specific options.

### Removed experimental APIs

`time_bucket_ng` and the `_timescaledb_debug` schema were removed in 2.25.0.
Migrate SQL, monitoring, and tooling that reference them.

### Removed internal chunk helper

`_timescaledb_functions.create_chunk_table` was removed in 2.20.0. Replace any
call to this private helper with supported hypertable and chunk APIs.

### Private chunk-constraint catalog

`_timescaledb_catalog.chunk_constraint` is not a table in 2.28.0. A temporary
compatibility view preserves current query behavior, but it too will be
removed. Migrate integrations to TimescaleDB informational views.

### Background-job catalog location

`bgw_job` moved into `_timescaledb_catalog` in 2.25.0, with an alias retained
for compatibility. Avoid hard-coding either private catalog location; use
supported informational views or APIs.

## Behavior Changes Worth Regression Testing

- Recompression became nonblocking by default in 2.19.0. Enable
  `enable_exclusive_locking_recompression` only to recover legacy exclusive
  locking.
- Boolean compression became enabled by default in 2.20.0.
- Incremental continuous-aggregate refresh policies became enabled by default
  in 2.21.0.
- UUIDv7 compression became enabled by default in 2.23.0.
- The refresh-policy `buckets_per_batch` default became `10` in 2.25.0.
- Compressed continuous aggregates received new automatic `segmentby` and
  `orderby` defaults in 2.25.0.
- `time_bucket_gapfill` requires a constant timezone in 2.26.0.
- `compressed_data_column_size` returns `bigint` in 2.27.0.
- Unsafe unique-column updates on compressed chunks are rejected in 2.28.0.
