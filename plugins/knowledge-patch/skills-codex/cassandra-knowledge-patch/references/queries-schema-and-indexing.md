# Queries, schema, and indexing

## CQL schema and expression behavior

### Descending UDT and vector keys

Frozen UDTs and vectors can be clustering keys ordered with `DESC` (5.0.3):

```cql
CREATE TYPE coordinates (x int, y int);
CREATE TABLE samples (
    sensor_id uuid,
    position frozen<coordinates>,
    embedding vector<float, 3>,
    PRIMARY KEY (sensor_id, position, embedding)
) WITH CLUSTERING ORDER BY (position DESC, embedding DESC);
```

Snapshot-generated schema CQL also emits definitions for UDTs used as reverse
clustering columns (5.0.3), so restoring such schema no longer depends on
manually supplying the missing type definition.

### Descriptions, aggregates, and names

`DESCRIBE TABLE` includes the table's materialized views (5.0.4). Treat the
output as a more complete schema representation when diffing or recreating a
table.

Built-in `min` and `max` return correct values for descending clustering
columns (5.0.4). Remove workarounds that inverted or post-processed the older
incorrect result.

Table creation rejects names that would lead to filenames that are too long
(5.0.6). DDL generators should cap constructed identifiers and handle a
validation error before any filesystem path is created.

`BytesType` compatibility is restricted to scalar types (5.0.7). Schema
evolution and compatibility checks must not treat it as compatible with
non-scalar types.

## Read and filtering correctness

Reading a partition after one of its columns has been deleted no longer throws
`IndexOutOfBoundsException` (5.0.3).

Replica filtering protection does not apply its fetch limit while a static row
is unresolved (5.0.4). This prevents premature limiting during resolution.

`RowFilter.isMutableIntersection()` evaluates numeric ranges on one column
correctly (5.0.5). Do not preserve a planner workaround based on the former
misclassification.

When reconciliation is required, `RowFilter` keeps deletions instead of
purging them early (5.0.5). This prevents deletion loss while replicas' results
are reconciled.

## Index selection and validation

When both a legacy secondary index and SAI exist on one column, Cassandra
prioritizes the legacy secondary index (5.0.4). Query diagnostics and
performance tests should expect that selection order.

Indexes reject empty values for non-literal types and for any other type that
does not permit an empty value (5.0.4). Validate such writes instead of relying
on an index to retain them.

During compaction, secondary-index implementations are notified about rows in
fully expired SSTables (5.0.6). Custom index implementations should handle the
notification consistently with other expired-row cleanup.

## SAI query correctness

Intersection queries avoid consistency violations involving repaired index
matches and matches across multiple non-indexed columns (5.0.4).

Multi-column SAI queries accept a non-indexed composite column that contains a
map instead of failing during filter evaluation (5.0.5).

Single-node SAI queries involving static columns return correct results
(5.0.5).

Range queries against early-open BTI SSTables return correct results before
the files are fully opened (5.0.6).

SAI approximate-nearest-neighbor queries use score-ordered iterators (5.0.7).
The corrected execution both preserves score ordering and improves query
speed; test relevance and latency together.

## SAI lifecycle and observability

`nodetool tablestats` exposes selected SAI index state and query-performance
metrics through the normal table-statistics output (5.0.3):

```shell
nodetool tablestats
```

Already-built SAI indexes no longer lag behind the node's `UP` state after a
restart (5.0.5). Clients should not encounter the earlier window in which a
node was up but those indexes were not queryable.

When repair flushes a partial partition or row modification, SAI marks the
index as non-empty (5.0.5).

When the current SSTable writer switches, Cassandra flushes the active SAI
segment builder (5.0.6), so segment state is not stranded at the writer
boundary.
