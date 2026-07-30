# Data types and expressions

## Missing-value semantics

- During the **1.0-upgrade**, `ewm_mean`, `ewm_std`, and `ewm_var` preserve
  missing positions instead of forward-filling them. Append `.forward_fill()`
  to recreate the earlier output.
- A missing lower or upper bound passed to `clip` leaves the original value
  unchanged instead of producing a missing result.
- Equality and inequality-with-missing work for both `List` and `Array`
  values (since 1.10.0).
- Bitwise aggregations ignore missing values (since 1.10.0).
- `list.contains` and `arr.contains` accept `nulls_equal` (since 1.30.0).
  Set it to `True` when a missing search value should match a missing element:
  `pl.col("items").list.contains(None, nulls_equal=True)`.
- List and Array `any` and `all` accept `ignore_nulls` (since 1.40.0), making
  nested Boolean reduction semantics explicit.
- List and Array values support `is_unique` (since 1.40.0).
- `DataFrame.fill_null` handles columns whose dtype is `Null` (since 1.40.0).

## Temporal and rolling expressions

- Datetime parsing formats containing `%f` or `%.f` default to microsecond
  precision during the **1.0-upgrade**. Excess fractional digits are
  truncated unless another precision is specified.
- `rolling_*_by` windows can be expressed by index count as well as time
  (since 1.10.0).
- `dt.truncate` anchors buckets at the Unix epoch (since 1.30.0); weekly
  buckets remain anchored on Monday.

## Nested and categorical data

- Strict casts validate failures inside nested values as well as at the outer
  dtype (since 1.30.0). Invalid inner values no longer bypass `strict=True`.
- Fixed-size `Array` columns can be grouping keys (since 1.30.0), for example
  `df.group_by("array_col").len()`.
- Appending Enum data does not merge different category sets (since 1.30.0).
  Align definitions before appending.
- `replace_strict` works with Enum values (since 1.40.0).
- `list.slice` broadcasts scalar input across rows (since 1.41.0).

## Numeric, Boolean, and aggregate behavior

- Cumulative minimum and maximum support Boolean data (since 1.10.0), such as
  `pl.col("flag").cum_min()` and `.cum_max()`.
- `Series.to_dummies` produces unique dummy-column names (since 1.10.0).
- Covariance involving a constant returns zero rather than `NaN` (since
  1.40.0).
- Decimal sum aggregation widens precision (since 1.41.0); the result need not
  retain the input decimal precision.

## Selection, reshaping, and reproducibility

- `pivot` retains rows whose `on` key is missing (since 1.40.0).
- `sample()` observes the global random seed (since 1.40.0), so globally
  seeded sampling is reproducible.
- `DataFrame.__array__` and `Series.__array__` match NumPy's standard
  `__array__` signature (since 1.41.0), allowing NumPy to pass its normal
  conversion arguments.
