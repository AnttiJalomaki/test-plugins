# Storage, SSTables, and Recovery

## SSTable lifecycle and formats

### Optional key-cache preservation

SSTable deletion can be requested without invalidating the key cache (since
5.0.3). Callers that need the previous invalidation behavior can retain it,
while callers that deliberately preserve cache entries can opt out.

### Large memory-mapped trie indexes

Memory-mapped trie indexes larger than 2 GiB can be read correctly (since
5.0.5). Index size crossing that boundary is no longer by itself a reason to
avoid the memory-mapped path.

### BTI early-open reads

Range queries against early-open BTI SSTables return correct results (since
5.0.6). The partial-open stage no longer carries the previous range-query
correctness hazard.

### TOC exception classification

A runtime exception from `FileUtils.write` while a `TOCComponent` is being
written remains a runtime exception (since 5.0.6). It is not reclassified as
an `FSError`; error handlers should preserve the distinction.

### Legacy streaming fallback

Legacy SSTables with the old Bloom-filter format automatically disable
zero-copy streaming and use a compatible fallback (since 5.0.7). This is a
per-file compatibility decision, not evidence that all streaming has lost
zero-copy capability.

## Snapshot compatibility

### Snapshot names

SSTable path validation accepts snapshot names that were unnecessarily
rejected before (since 5.0.5). A valid snapshot name should not need
application-side renaming solely to satisfy the old over-restrictive check.

### Reverse UDT clustering schema

Snapshot-generated schema CQL includes definitions for UDTs used as reverse
clustering columns (since 5.0.3). Restoring schema for a table with a
descending UDT clustering key therefore has the type definition it needs.

### Very old snapshot schemas

`SnapshotLoader` loads schemas created before Cassandra 2.1 even when their
directory names omit a table ID (since 5.0.8). Preserve those historical
directory names during recovery; the missing ID is a supported legacy layout.

## Compaction and index integration

### Corrupt SSTables

When compaction encounters a corrupt SSTable, it marks that SSTable as
suspected and releases the associated buffer-pool resources (since 5.0.5).
Diagnostics should look for the suspected marker, and resource accounting
should not assume the failed read retains those buffers.

### Fully expired SSTables

Compaction notifies secondary-index implementations about rows in fully
expired SSTables (since 5.0.6). Custom index implementations should handle
these expired-row notifications consistently even when the entire SSTable is
expired.

## Commitlog and Direct I/O

### Safe Direct I/O flushes

Commitlog data is flushed safely with Direct I/O enabled (since 5.0.5). The
Direct I/O path no longer has the earlier commitlog flush hazard.

### Recovery after CRC errors

`CommitLogSegmentReader` skips sync blocks correctly after encountering CRC
errors (since 5.0.5). Recovery can advance over the affected block rather than
misreading subsequent block boundaries.

### Tool initialization

Cassandra command-line tools skip their DirectIO check during initialization
(since 5.0.4). Tool startup should not be made conditional on a Direct I/O
capability check intended for the server path.

## Deletion and reconciliation semantics

### Complex collection deletions

Mutation serialization preserves complex deletions when a row contains
multiple collections (since 5.0.4). A serialized mutation must retain each
collection deletion rather than dropping one because other collections are
present.

### Reads after column deletion

Reading a partition after one of its columns has been deleted no longer fails
with `IndexOutOfBoundsException` (since 5.0.3). Treat such a failure as
unexpected on corrected nodes rather than as a required client retry case.

### Row-filter deletions

`RowFilter` retains deletions when result reconciliation requires them (since
5.0.5). Purging at that point could lose the deletion while replica results
are reconciled.

### Deterministic TTL replacement

Updating a column with a new TTL that produces the same expiration time is
deterministic (since 5.0.5). Replicas should agree on the update, avoiding
repair mismatches caused solely by equivalent expiration times.

## Capacity protection

A node can start for the first time with
`data_disk_usage_max_disk_size` already configured, even before the data
directory has been created (since 5.0.5). Initial provisioning can apply the
disk limit without a separate first-boot exception.

Full AutoRepair also observes disk-protection conditions (since 5.0.8).
Investigate those protections when scheduled full repair does not proceed.
