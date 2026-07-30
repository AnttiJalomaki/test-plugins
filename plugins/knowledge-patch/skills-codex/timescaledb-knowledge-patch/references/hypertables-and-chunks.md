# Hypertables and chunks

## Create and configure hypertables

### Declarative DDL

The `CREATE TABLE ... WITH` hypertable API is available since 2.20.0.
Columnstore can be enabled at creation time since 2.21.0:

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

`partition_column` is optional since 2.23.0, and declarative columnstore
creation automatically creates a columnstore policy. In the Apache 2 Edition,
an explicit `columnstore=false` has not been required since 2.22.0.

The `tsdb` reloption prefix is an alias for `timescaledb` in `WITH` and `SET`
clauses (since 2.19.0):

```sql
ALTER TABLE metrics SET (tsdb.enable_columnstore = true);
```

For existing hypertables, `columnstore` aliases `enable_columnstore` and
`timescaledb.chunk_time_interval` changes the chunk interval (since 2.20.0):

```sql
ALTER TABLE metrics SET (timescaledb.columnstore = true);
ALTER TABLE metrics SET (timescaledb.chunk_time_interval = '1 day');
```

A single `ALTER TABLE SET` can combine PostgreSQL and TimescaleDB reloptions
(since 2.23.0):

```sql
ALTER TABLE metrics SET (
    fillfactor = 90,
    timescaledb.columnstore = true
);
```

Use `ALTER TABLE ONLY` to apply a setting only to future chunks:

```sql
ALTER TABLE ONLY metrics
SET (timescaledb.orderby = 'time DESC');
```

The expert `timescaledb.default_chunk_time_interval` GUC controls the default
interval for new hypertables (since 2.26.0). Leave it unchanged unless
specifically recommended. Negative `chunk_interval` values are rejected since
2.27.0.

## Manage constraints, columns, and table state

- Columnstore tables accept foreign keys (since 2.20.0).
- Compressed chunks accept `CHECK` constraints and columns carrying them, and
  `ADD COLUMN` can include a unique constraint (since 2.20.0).
- Compressed hypertables accept `DROP NOT NULL` (since 2.18.0), and compressed
  chunks accept `SET NOT NULL` (since 2.19.0).
- Compressed columns accept any immutable constant expression as a default
  (since 2.25.0):

```sql
ALTER TABLE metrics ADD COLUMN scale integer DEFAULT (2 * 3);
```

- `ALTER COLUMN TYPE` is allowed with columnstore enabled when the hypertable
  has no compressed chunks (since 2.24.0).
- Hypertables can be unlogged since 2.23.0, trading durability for faster large
  imports:

```sql
ALTER TABLE metrics SET UNLOGGED;
```

- The database owner can configure hypertables and policies (since 2.28.0).

## Manage chunks

### Split, merge, attach, and detach

Chunk merging is supported since 2.18.0 and has a concurrent mode since 2.24.0.
Merging is not supported for multidimensional hypertables.

`split_chunk` can divide a large uncompressed chunk at a specified time since
2.20.0 and supports compressed chunks since 2.21.0. Uncompressed chunks can
also be manually attached to or detached from a hypertable since 2.21.0,
similar to PostgreSQL partition attachment and detachment.

### Dimension and range metadata

Primary-dimension information is exposed in the information schema since
2.20.0. UUIDv7 columns can partition hypertables since 2.22.0; chunk boundaries
derive time from the UUID's embedded timestamp. Since 2.24.0, UUIDv7 ranges in
the chunks informational view display as timestamps.

Retention policies support UUIDv7-partitioned hypertables since 2.25.0:

```sql
SELECT add_retention_policy('events', INTERVAL '30 days');
```

## Use triggers correctly

Hypertables support transition-table triggers since 2.18.0, but creating such a
trigger directly on a chunk is rejected. Event triggers can run for chunk
creation since 2.20.0; they are disabled by default:

```sql
SET timescaledb.enable_event_triggers = on;
```

`ENABLE TRIGGER` and `DISABLE TRIGGER` operate on hypertables since 2.27.0:

```sql
ALTER TABLE metrics DISABLE TRIGGER metrics_validate;
ALTER TABLE metrics ENABLE TRIGGER metrics_validate;
```

Creating a child table that inherits from a hypertable is explicitly rejected
since 2.27.0.

## Integrate PostgreSQL operations

- When a hypertable is in a publication, chunks created after 2.25.0 are added
  to that publication automatically.
- Hypertables correctly support `MERGE WHEN NOT MATCHED BY SOURCE` since
  2.28.0.
- Cross-type partition-key comparisons received correctness and crash fixes in
  2.26.0.
- Since 2.28.0, updates that would unsafely change unique columns on compressed
  chunks are rejected.
