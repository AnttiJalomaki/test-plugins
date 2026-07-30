# Indexing and Query Correctness

## Index selection and value eligibility

### Legacy secondary index versus SAI

When a column is covered by both a legacy secondary index and Storage-Attached
Indexing, Cassandra prioritizes the legacy secondary index (since 5.0.4).
Query analysis should account for this deterministic choice instead of
assuming SAI wins whenever it is present.

### Empty values

Indexes reject empty values for non-literal types and for any other type that
does not allow empty values (since 5.0.4). Write and indexing clients should
distinguish an empty value from a valid literal value rather than assuming all
types can index an empty byte sequence.

## SAI lifecycle and status

### Restart queryability

An already-built SAI index does not have an availability gap between the node
being marked `UP` after restart and the index being marked queryable (since
5.0.5). A client should not observe this former transient mismatch as normal
post-restart behavior.

### Repair flush state

If repair flushes a partial partition or a row modification, SAI marks the
index non-empty (since 5.0.5). This keeps index state aligned with flushed
repair data.

### SSTable writer switches

Switching the current SSTable writer flushes the active SAI segment builder
(since 5.0.6). Segment state is not left pending across the writer boundary.

### Forced optimized status format

`IndexStatusManager` can be explicitly forced to use its optimized
index-status format (since 5.0.7). Use the forced mode when automatic format
selection is unsuitable rather than trying to induce the optimized format
indirectly.

## Intersection and filtering

### Repaired and non-indexed matches

SAI intersection queries preserve consistency when combining repaired index
matches with matches on multiple non-indexed columns (since 5.0.4). This case
should not produce the prior consistency violation.

### Composite map filters

A multi-column SAI query can include a non-indexed composite column containing
a map (since 5.0.5). That filter shape no longer causes the query to fail.

### Static columns

Single-node SAI queries involving static columns return correct results (since
5.0.5). Do not work around this case by assuming static-column predicates are
intrinsically unsafe for SAI.

### Numeric ranges

`RowFilter.isMutableIntersection()` evaluates numeric ranges on one column
correctly (since 5.0.5). Query paths that depend on mutable-intersection
classification receive the correct result for same-column numeric ranges.

### Unresolved static rows

Replica filtering protection does not apply its fetch limit while a static
row remains unresolved (since 5.0.4). The coordinator can continue fetching
the data needed to resolve that row rather than prematurely enforcing the
limit.

### Reconciliation-required deletions

`RowFilter` does not purge deletions when reconciliation is required (since
5.0.5). This prevents a deletion needed by replica reconciliation from
disappearing before results are combined.

## ANN execution

SAI approximate-nearest-neighbor queries execute with score-ordered iterators
(since 5.0.7). The corrected ordering both fixes result execution and improves
query speed; callers should consume the resulting score order rather than
compensating for the older iterator behavior.

## Clustering order and aggregates

### Descending UDT and vector keys

UDTs and vectors can be clustering keys with `DESC` ordering (since 5.0.3):

```cql
CREATE TYPE coordinates (x int, y int);
CREATE TABLE samples (
    sensor_id uuid,
    position frozen<coordinates>,
    embedding vector<float, 3>,
    PRIMARY KEY (sensor_id, position, embedding)
) WITH CLUSTERING ORDER BY (position DESC, embedding DESC);
```

Snapshot-generated schema CQL includes the UDT definitions required by UDTs
used as reverse clustering columns. See the storage reference for snapshot
compatibility.

### `min` and `max`

The built-in `min` and `max` functions return correct results for clustering
columns ordered descending (since 5.0.4). Do not reverse or otherwise adjust
these aggregate results in the client to compensate for the former behavior.

## BTI early-open reads

Range queries over early-open BTI SSTables return correct results (since
5.0.6). Data read while a BTI SSTable is not yet fully opened should not be
treated as inherently unreliable.

## Secondary-index compaction notifications

During compaction, secondary-index implementations are notified about rows in
fully expired SSTables (since 5.0.6). Custom secondary indexes can use those
notifications to process expired rows consistently instead of missing the
entire fully expired file.
