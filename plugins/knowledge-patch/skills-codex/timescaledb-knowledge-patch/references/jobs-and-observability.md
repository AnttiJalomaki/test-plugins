# Jobs and observability

## Configure background jobs

Background jobs can have custom names since 2.20.0. Continuous-aggregate jobs
also show the aggregate name in the jobs informational view.

Per-job history configuration can separately cap retained successful and failed
executions since 2.23.0.

The `bgw_job` table moved to `_timescaledb_catalog` in 2.25.0, with a
compatibility `bgw_job` alias. Background worker jobs also accept a `work_mem`
configuration.

Background jobs no longer use advisory locks and support graceful cancellation
since 2.26.0.

## Observe long-running and compressed work

Index creation reports progress since 2.27.0.

`EXPLAIN` reports batch-pruning and false-positive statistics for composite
bloom indexes since 2.26.0. Multi-column predicates can be pushed into
compressed scans for both `SELECT` and `UPSERT`.

`timescaledb.enable_compression_ratio_warnings` defaults on since 2.20.0 and
warns about poor compression ratios.

`timescaledb.stats_max_chunks` controls the per-database capacity of the
in-memory compressed-chunk statistics cache since 2.28.0. It defaults to
`1024`; use `0` to disable it:

```sql
SET timescaledb.stats_max_chunks = 0;
```

## Account for maintenance work

`VACUUM FULL` recompresses affected chunks since 2.25.0, so plan for
recompression cost. Ordinary `VACUUM` and `ANALYZE` accept continuous aggregate
names and redirect to their materialization hypertables since 2.28.0.

Concurrent chunk merging is available since 2.24.0. Nonblocking recompression
allows concurrent DML by default since 2.19.0.

## Use supported metadata

Primary-dimension information is available through the information schema since
2.20.0. Prefer TimescaleDB informational views over private catalog tables:
`_timescaledb_catalog.chunk_constraint` became only a temporary compatibility
view in 2.28.0.

The database owner can configure hypertables and policies since 2.28.0.
