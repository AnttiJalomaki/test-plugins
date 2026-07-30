# Migration and Core API

## Construction and inference

- **Strict `Series` input (1.0-upgrade).** `pl.Series` applies strict
  construction to inferred and declared dtypes. Incompatible mixed input
  raises by default. With `strict=False`, `[1, 2, 3.5]` finds `Float64`; with
  an explicit integer dtype, the float is cast rather than replaced by null.
- **Row orientation (1.0-upgrade).** `DataFrame` infers orientation from data
  and schema dimensions rather than value types, and warns when it infers
  rows. Use `orient="row"` for heterogeneous row records.
- **Time-zone construction (1.0-upgrade).** Constructing into a zoned datetime
  dtype always converts values to that zone rather than replacing time-zone
  metadata. A naive midnight constructed as Amsterdam time can become 01:00
  CET, so check for wall-clock shifts.
- **Fixed-size arrays (1.0-upgrade).** `Series.reshape` and construction from
  two-dimensional NumPy arrays produce fixed-size `Array`, not `List`. Use
  `.arr.to_list()` when variable-length lists are required.
- **Byte scalar broadcasting (since 1.41.0).** The `DataFrame` constructor
  broadcasts byte scalars across rows instead of treating them as
  non-broadcast inputs.

## Replacement, selection, and indexing

- **`replace` versus `replace_strict` (1.0-upgrade).** `replace` preserves the
  input dtype, and its `default` and `return_dtype` parameters are deprecated.
  Use `s.replace_strict(old, new, default=s)` when the mapping may change
  dtype. Without `default`, any unmapped non-null input raises.
- **Positional `nth` (1.0-upgrade).** `pl.nth` no longer has `columns`
  behavior; every positional input is an index. Use `pl.col("a").get(1)` for
  a row from a named expression.
- **Out-of-bounds lookup (1.0-upgrade).** All `get` and `gather` variants raise
  by default. Pass `null_on_oob=True` to return null.
- **Integer Series indices (since 1.40.0).** Series index assignment accepts
  every integer dtype.
- **Unique dummy names (since 1.10.0).** `Series.to_dummies` avoids duplicate
  output column names.

## Frame reshaping and ordering

- **Pivot API (1.0-upgrade).** `DataFrame.pivot` renamed `columns` to `on`,
  made it the first positional argument, and made `index` and `values`
  optional with remaining columns inferred. With multiple value columns,
  output names omit the redundant pivot-column name.
- **Null pivot keys (since 1.40.0).** `pivot` retains rows whose `on` value is
  null.
- **Run-length fields (1.0-upgrade).** `rle` returns struct fields `len` and
  `value`, replacing `lengths` and `values`. `len` uses the unsigned index
  dtype, `UInt32` by default, rather than `Int32`.
- **Sorted annotations (1.0-upgrade).** `set_sorted` accepts one column per
  call because it promises that column is independently sorted. Chain calls
  for several sorted columns.
- **All-column unnest (since 1.40.0).** `unnest()` with no column arguments
  operates on every applicable column.

## Window and range defaults

- **Dynamic-window offset (1.0-upgrade).** `group_by_dynamic` defaults
  `offset` to zero rather than negative `every`. Specify `offset="-1d"` with
  `every="1d"` when the former leading window is required.
- **Integer-range validation (since 1.40.0).** `pl.int_ranges` raises for
  non-numeric input instead of accepting it.
- **Ignored correlation degrees of freedom (since 1.40.0).** `rolling_corr`
  ignores and deprecates `ddof`; changing it no longer changes results.

## Equality, names, and implicit columns

- **Series name equality (1.0-upgrade).** `Series.equals` ignores names by
  default. Pass `check_names=True` for name-sensitive equality.
- **Line reader output name (since 1.40.0).** `scan_lines` and `read_lines`
  name their implicit column `line`, not `lines`. Update selectors or rename
  explicitly.

## Dtypes, aliases, and deprecations

- **Instance-only dtype attributes (1.0-upgrade).** Attributes such as
  `time_unit` and `time_zone` exist on dtype instances, so
  `pl.Datetime.time_unit` raises. Class-aware code can use
  `getattr(dtype, "time_unit", None)`.
- **Removed dtype alias exports (1.0-upgrade).** Public type aliases are no
  longer re-exported from `polars` or `polars.datatypes`. Define local public
  aliases, such as `pl.DataType | type[pl.DataType]`.
- **Keyword-only decimal inference (since 1.20.0).** Pass
  `str.to_decimal(inference_length=100)`; its parameter is keyword-only.
- **PEP 702 markers (since 1.30.0).** Deprecated Polars APIs expose PEP 702
  deprecation information to compatible static-analysis tools.
- **Dataframe interchange (since 1.40.0).** Polars integration with the
  dataframe interchange protocol is deprecated; treat dependent integrations
  as transitional.
- **`StringCache` (since 1.41.0).** `StringCache` is deprecated. Avoid it in
  new code and prepare existing uses for removal.

## Exceptions and diagnostics

- **Specific operation errors (1.0-upgrade).** Many failures previously
  surfaced as `ComputeError` now raise `InvalidOperationError` or
  `SchemaError`. Update broad handlers around casts and schema-dependent
  operations when the distinction matters.
- **Lazy schema properties (1.0-upgrade).** Accessing `LazyFrame.schema`,
  `.dtypes`, `.columns`, or `.width` emits `PerformanceWarning` because schema
  resolution may be expensive. Call `collect_schema()` and inspect the
  returned `Schema`.

## Installation extras

- **Renamed extras (1.0-upgrade).** Optional dependency definitions changed.
  Installations requesting `fastexcel`, `gevent`, `matplotlib`, or `async`
  should review their extras. The documented replacement for
  `polars[fastexcel,gevent,matplotlib]` is
  `polars[calamine,async,graph]`.
