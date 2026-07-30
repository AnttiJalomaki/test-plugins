---
name: polars-knowledge-patch
description: Polars
version: 1.41.0
license: MIT
metadata:
  author: Nevaberry
---


# Polars Knowledge Patch

## When to load this skill

Load this skill when a Python, Rust, or SQL task uses Polars and any of the
following are involved:

- upgrading code that depends on constructor, dtype, null, schema, or renamed
  API behavior;
- reading or writing Parquet, Excel, CSV, Delta Lake, Iceberg, Arrow, or
  databases in eager, lazy, or streaming execution;
- serializing frames, expressions, plans, or Python objects;
- using Polars SQL or nested, decimal, temporal, Enum, or `Float16` data.

First inspect the project's declared Polars version. Apply an item only when
the installed version includes that behavior. Prefer the manifest, executable
tests, and observed schemas over assumptions.

## Working method

1. Identify eager, lazy, streaming, or SQL execution and inspect both schemas.
2. Make orientation, strictness, null policy, and partition behavior explicit.
3. Treat persisted plans and expressions as compatibility-sensitive.
4. Exercise boundary data and verify both values and nested dtypes.

## Reference index

| Reference | Topics |
| --- | --- |
| [migration-and-core-api.md](references/migration-and-core-api.md) | Constructors, renamed APIs, defaults, deprecations, exceptions |
| [expressions-and-data-types.md](references/expressions-and-data-types.md) | Nulls, nested values, temporal expressions, decimals, enums |
| [lazy-streaming-and-sql.md](references/lazy-streaming-and-sql.md) | Lazy schemas, optimizer controls, streaming, SQL behavior |
| [io-cloud-and-databases.md](references/io-cloud-and-databases.md) | Parquet, CSV, Excel, cloud credentials, Delta, Iceberg, databases |
| [serialization-runtime-and-arrow.md](references/serialization-runtime-and-arrow.md) | Serialization, Python support, Arrow interchange, exception fidelity |

## Breaking changes and deprecations

### Make construction explicit

`Series` construction is strict even when the dtype is inferred. Mixed values
that cannot satisfy one inferred dtype raise by default:

```python
s = pl.Series([1, 2, 3.5], strict=False)
```

For heterogeneous row data, declare row orientation:

```python
df = pl.DataFrame([[1, "a"], [2, "b"]], orient="row")
```

Two-dimensional NumPy input and `reshape` produce fixed-size `Array` values.
Convert with `.arr.to_list()` only when a downstream consumer requires
variable-length `List` values.

A datetime constructor with a zoned dtype converts values into the requested
zone. Check the resulting instant and wall-clock value instead of assuming
that the operation only changes metadata.

### Replace and index deliberately

`replace` preserves the existing dtype. Use `replace_strict` when a mapping may
change dtype:

```python
out = s.replace_strict(old, new, default=s)
```

Without `default`, every non-null input must be mapped. This strict form also
works with Enum data.

All `get` and `gather` variants raise on an out-of-bounds index by default.
Request nullable lookup explicitly:

```python
item = pl.col("items").list.get(1, null_on_oob=True)
```

Every positional argument to `pl.nth` is an index. Use
`pl.col("a").get(1)` to index a named expression.

### Update renamed fields and arguments

`DataFrame.pivot` uses `on` in place of `columns`; `on` is its first
positional argument. `index` and `values` may be inferred, and multi-value
output names are shorter.

The `rle` result fields are `len` and `value`. The length uses the unsigned
index dtype.

Call `set_sorted` once per independently sorted column:

```python
df = df.set_sorted("a").set_sorted("b")
```

The implicit output column from `scan_lines` and `read_lines` is `line`.
Calling `unnest()` without arguments targets every applicable column.

### Review changed defaults

`group_by_dynamic` uses a zero offset. Specify a negative interval when the
old leading-window layout is required:

```python
df.group_by_dynamic("ts", every="1d", offset="-1d")
```

`LazyFrame.map_batches` enables no optimizer transformations by default.
State any safe optimization behavior explicitly.

`Series.equals` ignores names unless `check_names=True`.

EWM operations preserve null positions. Append `.forward_fill()` only when
forward-filled output is intended. A null bound passed to `clip` leaves the
input value unchanged.

### Remove deprecated dependencies

- Avoid new uses of `StringCache`; it is deprecated.
- Stop relying on cache-related `scan_ipc` arguments.
- Do not use `rolling_corr(ddof=...)`; `ddof` is deprecated and ignored.
- Treat the dataframe interchange protocol integration as transitional.
- Replace separate lazy optimizer controls with `QueryOptFlags`.
- Update removed installation extras; use supported extras such as
  `calamine`, `async`, and `graph`.
- Access `time_unit` and `time_zone` on dtype instances, not dtype classes,
  and define public dtype aliases locally.

## High-use recipes

### Resolve lazy schemas once

Properties such as `LazyFrame.schema`, `.dtypes`, `.columns`, and `.width` can
trigger expensive resolution and emit `PerformanceWarning`. Collect once:

```python
schema = lf.collect_schema()
names = schema.names()
```

Use `LazyFrame.match_to_schema` to reconcile an expected schema before
execution:

```python
lf = lf.match_to_schema({"id": pl.Int64})
```

Strict casts validate nested inner values as well as the outer dtype.

### Handle Parquet paths and partitions

Directory inputs enable Hive partitioning by default. A file, glob, or list of
files does not, so request it when partition columns must be recovered:

```python
lf = pl.scan_parquet(paths, hive_partitioning=True)
```

Use `cast_options` for scan-time Parquet casts. Project only needed columns;
unprojected columns are not dtype-validated. Parquet sinks preserve field IDs,
and readers give `DuplicateError` for duplicate column names.

### Make nested null semantics visible

Nested operations now expose specific controls:

```python
has_null = pl.col("items").list.contains(None, nulls_equal=True)
any_value = pl.col("items").list.any(ignore_nulls=False)
```

List and Array values support missing-aware equality, membership controls,
Boolean reductions, and uniqueness. Test empty lists, null lists, and null
elements separately.

### Use stable streaming and sinks

The streaming engine and `sink_*` APIs are supported interfaces. Streaming
covers grouped as-of joins, interpolation, inferred-format `strptime`,
covariance and correlation, PyArrow dataset sources, common group
aggregations, and Parquet output.

Do not assume that streaming changes semantic edge cases: grouped as-of joins
preserve null rows, and a Delta sink does not require maintained order.

### Choose the right SQL entry point

`DataFrame.sql` and `LazyFrame.sql` see only their own frame. Use top-level SQL
for multiple frames:

```python
result = pl.sql(
    "SELECT * FROM left CROSS JOIN right",
    eager=True,
)
```

Polars SQL supports true division, bit operations, discrete quantiles,
aggregate `FILTER`, `STRING_AGG`, Unicode normalization, and multiline
`LIKE`/`ILIKE`. Invalid expressions and invalid `HAVING` placement raise
specific SQL errors.

### Read spreadsheets predictably

`read_excel` defaults to the Calamine engine. Choose `xlsx2csv` when
engine-specific options such as `skip_empty_lines` are required:

```python
df = pl.read_excel(path, engine="xlsx2csv")
```

Use `drop_empty_rows` deliberately. Excel reads can select a named table and
accept raw bytes; Excel writes accept file-like outputs.

### Preserve serialization boundaries

Frame, lazy-frame, and expression serialization defaults to binary. Pair
binary output with `BytesIO`, or request `format="json"`. Row-oriented
`write_json` output is data, not a serialized frame:

```python
buffer = io.BytesIO()
df.serialize(buffer)
buffer.seek(0)
restored = pl.DataFrame.deserialize(buffer)
```

Persisted DSL data must be compatible with its reader. Credential-provider
objects are intentionally excluded from serialized state.

## Verification checklist

- Confirm Polars and Python versions, constructor orientation, and inferred
  dtypes from project files and focused tests.
- Inspect lazy schemas and exercise null, empty, duplicate, and boundary cases.
- Verify time zones, truncation anchors, and fractional datetime precision.
- Distinguish Parquet directories, files, globs, and explicit lists.
- Test cloud credentials without assuming serialization, compare eager,
  streaming, and SQL dtypes, and deserialize in the target runtime.
