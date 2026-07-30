# Lazy execution and joins

## Schema planning and optimization

- During the **1.0-upgrade**, accessing `LazyFrame.schema`, `.dtypes`,
  `.columns`, or `.width` emits `PerformanceWarning` because lazy schema
  resolution may be expensive. Call `lf.collect_schema()` and inspect the
  returned `Schema`.
- `LazyFrame.match_to_schema` reconciles a lazy frame to an expected schema
  before execution (since 1.30.0), for example
  `lf.match_to_schema({"id": pl.Int64})`.
- Lazy optimization controls are centralized in `QueryOptFlags` (since
  1.30.0). Pass the flags object instead of separate controls.
- `LazyFrame.map_batches` defaults to no optimizations (since 1.40.0). Do not
  assume optimizer transformations are enabled for a callback.

## Streaming execution and sinks

- The newer streaming engine handles first/last and additional `group_by`
  aggregations and can write through the Parquet sink (since 1.20.0).
- Sink APIs are stable rather than experimental (since 1.30.0).
- Streaming support includes grouped as-of joins, native interpolation,
  `strptime` with `format=None`, covariance, correlation, and PyArrow dataset
  sources (since 1.40.0).
- The streaming engine is stable rather than experimental (since 1.41.0).
  Code that explicitly selects it can treat it as a supported engine.

## Joins and sorted data

- `join_where` correctly resolves partially overlapping column names (since
  1.10.0).
- `join_asof` accepts `allow_exact_matches` (since 1.20.0). Set
  `allow_exact_matches=False` to exclude equal keys:

  ```python
  left.join_asof(right, on="ts", allow_exact_matches=False)
  ```

- The top-level `pl.merge_sorted` merges multiple already-sorted frames (since
  1.40.0).
- Grouped as-of joins preserve missing rows rather than dropping them (since
  1.40.0).

## Windows and grouping

- `group_by_dynamic` uses a zero default offset after the **1.0-upgrade**,
  rather than negative `every`. Specify an explicit negative offset when the
  leading historical window is required.
- `rolling_*_by` operations can use index-count windows as well as time
  windows (since 1.10.0).
- Window expressions can call `.over()` without `partition_by` (since
  1.30.0), treating the full input as one window:

  ```python
  df.with_columns(total=pl.col("value").sum().over())
  ```

- Fixed-size `Array` values can be grouping keys (since 1.30.0).
- `group_by()` accepts no key expressions (since 1.40.0), exposing the
  group-by aggregation interface for one global group.

## Plan persistence

- UDF deserialization checks the Python version (since 1.10.0).
- Credential providers are not serialized with plans or objects (since
  1.20.0).
- A DSL representation incompatible with its reader is rejected (since
  1.30.0). Keep persisted expressions and lazy plans compatible with the
  process that restores them.
