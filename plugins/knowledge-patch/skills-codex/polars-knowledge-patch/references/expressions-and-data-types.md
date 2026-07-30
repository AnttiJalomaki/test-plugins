# Expressions and Data Types

## Null and missing-value semantics

- **EWM and clip (1.0-upgrade).** `ewm_mean`, `ewm_std`, and `ewm_var`
  preserve null positions instead of forward-filling; add `.forward_fill()`
  only when that result is desired. A null lower or upper `clip` bound leaves
  the original value unchanged rather than returning null.
- **Missing-aware nested comparison (since 1.10.0).** Equality and
  inequality-with-missing operations work for both `List` and `Array`.
- **Nested membership (since 1.30.0).** `list.contains` and `arr.contains`
  accept `nulls_equal`; use
  `pl.col("items").list.contains(None, nulls_equal=True)` when null search
  values should match null elements.
- **Nested Boolean reduction (since 1.40.0).** List and Array `any` and `all`
  accept `ignore_nulls`, making null policy explicit.
- **Grouped as-of null rows (since 1.40.0).** Grouped as-of joins preserve
  null rows instead of dropping them.
- **Null-typed columns (since 1.40.0).** `DataFrame.fill_null` operates on
  columns whose dtype is `Null`.
- **Bitwise aggregation nulls (since 1.10.0).** Bitwise aggregations ignore
  null values.

## Nested and fixed-size data

- **Index-count rolling windows (since 1.10.0).** `rolling_*_by` operations
  can define a window by index count as well as by time.
- **Boolean cumulative extrema (since 1.10.0).** Boolean expressions support
  cumulative minimum and maximum, including
  `pl.col("flag").cum_min()` and `.cum_max()`.
- **Overlapping join names (since 1.10.0).** `join_where` resolves columns
  whose names partially overlap.
- **Array grouping keys (since 1.30.0).** Fixed-size `Array` columns can be
  grouping keys, for example `df.group_by("array_col").len()`.
- **Strict inner casts (since 1.30.0).** Strict casts enforce failures inside
  nested values as well as at the outer dtype; invalid inner values cannot
  bypass `strict=True`.
- **Nested uniqueness (since 1.40.0).** `List` and `Array` support
  `is_unique`.
- **Scalar list slicing (since 1.41.0).** `list.slice` broadcasts scalar
  input across rows of a list column.

## Grouping, windows, joins, and sampling

- **Exact as-of matching (since 1.20.0).** `join_asof` accepts
  `allow_exact_matches`. Set it to `False` to exclude equal join keys.
- **Global windows (since 1.30.0).** Window expressions may call `.over()`
  without `partition_by`, treating the full input as one window:
  `pl.col("value").sum().over()`.
- **Keyless grouping (since 1.40.0).** `group_by()` can be called without key
  expressions to use group-by aggregation for one global group.
- **Multiple sorted frames (since 1.40.0).** Top-level `pl.merge_sorted`
  merges multiple already-sorted frames.
- **Global sampling seed (since 1.40.0).** `sample()` respects the global
  random seed, so global seeding makes samples reproducible.

## Temporal and string expressions

- **Fractional datetime parsing (1.0-upgrade).** `str.to_datetime` formats
  containing `%f` or `%.f` default to microsecond, not nanosecond, precision.
  Excess fractional digits are truncated unless precision is specified
  another way.
- **Unicode normalization (since 1.20.0).** String expressions expose
  `str.normalize`; the equivalent SQL function is `NORMALIZE`.
- **Epoch-aligned truncation (since 1.30.0).** `dt.truncate` anchors buckets
  at the Unix epoch. Weekly buckets remain anchored on Monday.

## Decimal, floating-point, and integer types

- **Arrow decimal preservation (1.0-upgrade).** `pl.from_arrow` maps Arrow
  decimal arrays to Polars `Decimal`, not `Float64`. Decimal support needs no
  activation, and `Config.activate_decimals` was removed.
- **Database `Int128` (since 1.30.0).** Database reads infer Polars `Int128`
  from database types that expose that integer width.
- **Constant covariance (since 1.40.0).** Covariance involving a constant
  returns zero rather than `NaN`.
- **Stable `Float16` (since 1.41.0).** `Float16` is a stable dtype rather than
  an experimental one.
- **Wider decimal sums (since 1.41.0).** Decimal sum aggregation widens
  precision; the output need not retain the input decimal precision.

## Enum behavior

- **Appending Enum values (since 1.30.0).** Appending Enum data does not merge
  different category sets. Align categories before appending inputs whose
  definitions differ.
- **Strict Enum replacement (since 1.40.0).** `replace_strict` works with Enum
  data.
