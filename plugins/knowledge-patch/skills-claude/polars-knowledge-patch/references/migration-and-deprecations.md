# Migration and deprecations

## Construction and schema inference

- During the **1.0-upgrade**, `pl.Series` applies strict construction to
  inferred as well as declared dtypes. Mixed incompatible values raise by
  default. `pl.Series([1, 2, 3.5], strict=False)` finds a common `Float64`
  dtype; with an explicit integer dtype, it casts the float instead of
  replacing it with a missing value.
- `DataFrame` orientation is inferred from data and schema dimensions rather
  than by inspecting value types, and inferred row orientation warns. Use
  `orient="row"` for heterogeneous row data.
- Constructing values with a zoned datetime dtype always converts them to the
  target zone. It can shift wall-clock values; a naive
  `2020-01-01 00:00` constructed as
  `pl.Datetime("us", "Europe/Amsterdam")` becomes `01:00 CET`.
- A two-dimensional NumPy array and `Series.reshape` produce fixed-size
  `Array`, not `List`. Convert with `.arr.to_list()` when downstream code
  requires the list dtype.
- Data type details such as `time_unit` and `time_zone` are instance
  attributes. `pl.Datetime.time_unit` raises; class-aware code can use
  `getattr(dtype, "time_unit", None)`.
- Public dtype aliases are no longer re-exported from `polars` or
  `polars.datatypes`. Define aliases locally, for example
  `PolarsDataType = pl.DataType | type[pl.DataType]`.
- Frame construction broadcasts byte scalars across rows (since 1.41.0)
  rather than treating them as non-broadcast inputs.

## API and default changes

- `replace` preserves the existing dtype. Its `default` and `return_dtype`
  arguments are deprecated. Use
  `s.replace_strict(old, new, default=s)` when replacement may change dtype;
  without `default`, unmapped non-missing inputs raise.
- `DataFrame.pivot` renamed `columns` to `on`, moved it to the first
  positional argument, and made `index` and `values` optional. With several
  value columns, generated names omit the redundant pivot-column name:
  `test_1_maths` rather than `test_1_subject_maths`.
- `Series.equals` ignores names by default. Pass `check_names=True` for
  name-sensitive equality.
- Every positional input to `pl.nth` is an index; the old `columns` behavior
  is gone. For a named expression use `pl.col("a").get(1)`, not
  `pl.nth(1, "a")`.
- The struct returned by `rle` uses `len` and `value`, not `lengths` and
  `values`. `len` uses the unsigned index dtype, `UInt32` by default, rather
  than `Int32`.
- `set_sorted` accepts one column per call because each call asserts that one
  column is individually sorted. Chain
  `df.set_sorted("a").set_sorted("b")` for independent annotations.
- `get` and `gather` variants raise for out-of-bounds indices by default. Pass
  `null_on_oob=True` to return a missing value.
- `group_by_dynamic` defaults `offset` to zero rather than negative `every`.
  Specify the prior leading window explicitly, such as
  `every="1d", offset="-1d"`.
- The `str.to_decimal` inference parameter is keyword-only (since 1.20.0):
  use `str.to_decimal(inference_length=100)`.
- `scan_lines` and `read_lines` name their implicit output column `line`, not
  `lines` (since 1.40.0). Select the new name or rename it explicitly.
- `unnest()` with no column arguments operates on every applicable column
  (since 1.40.0).
- `pl.int_ranges` rejects non-numeric inputs rather than accepting them
  (since 1.40.0).
- Series index assignment accepts every integer dtype (since 1.40.0).

## Exceptions and validation

- Cast and schema-dependent failures that formerly used broad `ComputeError`
  reporting may raise `InvalidOperationError` or `SchemaError`. Update
  exception handlers around these operations.
- Python exceptions crossing Polars execution preserve their original type and
  traceback (since 1.30.0), enabling precise handlers and diagnostics.
- Constructing from a PyArrow table with duplicate names raises
  `DuplicateError` (since 1.20.0).
- `read_csv` validates the length of a `schema_overrides` sequence and raises
  for an invalid length (since 1.20.0).
- Deserialization rejects a DSL representation incompatible with its reader
  (since 1.30.0). Persisted expressions and lazy plans require a compatible
  representation.

## Deprecations and platform support

- Python 3.9 became the minimum supported Python version in 1.10.0.
- Python 3.13 is officially supported since 1.20.0.
- Installation extras changed during the 1.0-upgrade. Replace definitions
  using `fastexcel`, `gevent`, `matplotlib`, or `async` as appropriate; the
  documented replacement for `polars[fastexcel,gevent,matplotlib]` is
  `polars[calamine,async,graph]`.
- Deprecated APIs expose PEP 702 metadata (since 1.30.0), allowing compatible
  static analyzers to flag their use.
- The dataframe interchange protocol integration is deprecated (since
  1.40.0); integrations based on it are transitional.
- Cache-related arguments to `scan_ipc` are deprecated (since 1.40.0).
- `rolling_corr` ignores and deprecates `ddof` (since 1.40.0); supplying it no
  longer changes results.
- `StringCache` is deprecated (since 1.41.0). Avoid it in new code and prepare
  existing uses for removal.
- `Float16` is stable rather than experimental (since 1.41.0).
