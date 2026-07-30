# Storage and data path

## Cache, hints, and mutation lifetime

SSTable deletion has an option that preserves the key cache instead of always
invalidating it (5.0.3). Use preservation only when the deletion caller can
accept retained entries; the default invalidation assumption is no longer the
only available behavior.

Hint expiry calculates TTL from the request start time rather than the timeout
time (5.0.3). Capacity and expiry analysis should use the corrected origin.

Schema mismatch no longer categorically blocks hint delivery (5.0.3). Do not
diagnose delivered hints during a mismatch as inherently invalid.

Changing a column to a new TTL while keeping the same expiration time is
deterministic (5.0.5), preventing replica divergence and repair mismatches for
that update pattern.

## Mutation and deletion serialization

Rows containing multiple collections retain their complex deletions during
mutation serialization (5.0.4). This closes a data-correctness path where a
collection deletion could disappear.

## Commitlog and recovery

Commitlog buffers are flushed safely when Direct I/O mode is enabled (5.0.5).

After a CRC error, `CommitLogSegmentReader` skips sync blocks correctly
(5.0.5), allowing recovery to continue at the appropriate block boundary.

## SSTable formats and readers

Memory-mapped trie indexes larger than 2 GiB are read correctly (5.0.5). Do
not impose an application-side 2 GiB limit as a workaround for the former
reader failure.

A runtime exception from `FileUtils.write` while writing a `TOCComponent`
remains a runtime exception instead of being reclassified as `FSError`
(5.0.6). Error handling should distinguish this failure from filesystem error
classification.

Legacy SSTables using the old Bloom-filter format automatically disable
zero-copy streaming and fall back to a compatible path (5.0.7). Mixed-format
streaming need not force zero-copy for files that cannot support it.

## Compaction and corruption

Unified Compaction validates its minimum and target size settings (5.0.5).
Invalid size combinations fail validation and should be corrected rather than
expected to run.

When compaction reads a corrupted SSTable, Cassandra marks it suspected and
releases the associated buffer-pool resources (5.0.5). Incident handling can
use the suspected status without expecting a buffer leak from that path.

The configured `MAX_PARALLEL_TRANSFERS` limit is honored (5.0.5). Streaming
capacity planning should use the configured cap rather than compensate for
earlier over-parallelization.

## Snapshot compatibility

SSTable path validation accepts snapshot names that were unnecessarily
rejected before (5.0.5).

`SnapshotLoader` accepts schemas from before Cassandra 2.1 whose directory
names lack a table ID (5.0.8). Historical snapshot recovery can feed those
directories directly to the compatibility path.
