# Hypertables, Chunks, and DDL

## Declarative Hypertable Creation

Hypertables can be created through `CREATE TABLE ... WITH` (since 2.20.0).
`tsdb` is a short alias for the `timescaledb` reloption prefix:

```sql
CREATE TABLE metrics (
    time timestamptz NOT NULL,
    device_id text,
    value double precision
) WITH (
    tsdb.hypertable,
    tsdb.partition_column = 'time'
);
```

The declarative API accepts `columnstore` at creation time (since 2.21.0):

```sql
CREATE TABLE metrics (
    time timestamptz NOT NULL,
    device_id text,
    value double precision
) WITH (
    tsdb.hypertable,
    tsdb.partition_column = 'time',
    tsdb.columnstore
);
```

`partition_column` became optional in declarative creation in 2.23.0. A
declarative hypertable created with columnstore enabled automatically receives
a columnstore policy:

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

TimescaleDB Apache 2 Edition does not require an explicit `columnstore=false`
option in declarative DDL (since 2.22.0).

## Reloptions and Chunk Intervals

An existing hypertable's chunk interval can be changed through a reloption
(since 2.20.0):

```sql
ALTER TABLE metrics
SET (timescaledb.chunk_time_interval = '1 day');
```

PostgreSQL and TimescaleDB reloptions can be mixed in one `ALTER TABLE SET`
(since 2.23.0):

```sql
ALTER TABLE metrics SET (
    fillfactor = 90,
    timescaledb.columnstore = true
);
```

Use `ALTER TABLE ONLY` to apply reloption changes to future chunks without
changing existing chunks (since 2.23.0):

```sql
ALTER TABLE ONLY metrics
SET (timescaledb.orderby = 'time DESC');
```

The expert `timescaledb.default_chunk_time_interval` GUC controls the default
interval for new hypertables (since 2.26.0). Leave it unchanged unless
specifically recommended.

Negative `chunk_interval` values are rejected (since 2.27.0). Continuous
aggregate definitions also reject non-positive bucket widths, and `time_bucket`
with `DATE` input rejects a sub-day offset.

Adaptive chunking was removed in 2.28.0. Replace any dependency on adaptive
chunk sizing with an explicit interval before upgrading.

## Chunk Splitting, Merging, and Attachment

Chunk merging is supported (since 2.18.0), but not for multidimensional
hypertables (since 2.20.0). A concurrent merge mode is available (since
2.24.0).

`split_chunk` can divide a large uncompressed chunk at a specified time (since
2.20.0), and supports compressed chunks as well (since 2.21.0).

Uncompressed chunks can be manually attached to or detached from a hypertable
(since 2.21.0), providing PostgreSQL-like partition attachment and detachment.

Creating a child table that inherits from a hypertable is explicitly rejected
(since 2.27.0). Use supported chunk attachment rather than PostgreSQL table
inheritance.

## Partitioning and UUIDv7

UUIDv7 columns can be hypertable partitioning columns (since 2.22.0).
TimescaleDB derives time-based chunk boundaries from the timestamp embedded in
the UUID.

The chunks informational view displays UUIDv7 ranges as timestamps (since
2.24.0). `time_bucket` accepts UUIDv7 and returns a timezone-aware timestamp,
which enables continuous aggregates over UUIDv7-partitioned hypertables.

Retention policies support UUIDv7-partitioned hypertables (since 2.25.0):

```sql
SELECT add_retention_policy('events', INTERVAL '30 days');
```

Cross-type comparisons against a partition column no longer risk wrong results
or crashes (since 2.26.0). On earlier versions, avoid relying on mixed-type
comparisons for partition pruning or correctness.

## Constraints and Column Changes

Compressed hypertables accept `DROP NOT NULL` (since 2.18.0) and compressed
chunks accept `SET NOT NULL` (since 2.19.0):

```sql
ALTER TABLE metrics ALTER COLUMN device_id SET NOT NULL;
```

Columnstore tables support foreign keys. Compressed chunks support `CHECK`
constraints and columns that carry them, and `ADD COLUMN` can include a unique
constraint (since 2.20.0).

`ALTER COLUMN TYPE` on a columnstore-enabled hypertable is allowed when it has
no compressed chunks (since 2.24.0):

```sql
ALTER TABLE metrics ALTER COLUMN value TYPE double precision;
```

Compressed columns accept any immutable constant expression as a default
(since 2.25.0):

```sql
ALTER TABLE metrics ADD COLUMN scale integer DEFAULT (2 * 3);
```

An update that would unsafely modify a unique column on a compressed chunk is
rejected (since 2.28.0).

## Triggers and Events

Hypertables support transition-table triggers (since 2.18.0). Creating such a
trigger directly on a chunk is rejected.

Event triggers can run when chunks are created (since 2.20.0). They are gated
by `timescaledb.enable_event_triggers`, which defaults to `OFF`:

```sql
SET timescaledb.enable_event_triggers = on;
```

`ENABLE TRIGGER` and `DISABLE TRIGGER` work on hypertables (since 2.27.0):

```sql
ALTER TABLE metrics DISABLE TRIGGER metrics_validate;
ALTER TABLE metrics ENABLE TRIGGER metrics_validate;
```

## Durability, Publications, and Permissions

Hypertables can be made unlogged (since 2.23.0), trading durability for faster
large imports:

```sql
ALTER TABLE metrics SET UNLOGGED;
```

When a hypertable belongs to a publication, future chunks are automatically
added to that publication (since 2.25.0).

The database owner can configure hypertables and policies (since 2.28.0).

`MERGE WHEN NOT MATCHED BY SOURCE` works correctly on hypertables (since
2.28.0).

## Metadata and Private Catalogs

The information schema exposes primary-dimension information (since 2.20.0).
Use public informational views rather than binding integrations to private
catalog tables.

`_timescaledb_functions.create_chunk_table` was removed in 2.20.0. Do not call
that internal helper.

`_timescaledb_catalog.chunk_constraint` stopped being a table in 2.28.0. A
temporary compatibility view preserves current query behavior, but that view is
also scheduled for removal. Migrate integrations to TimescaleDB informational
views.
