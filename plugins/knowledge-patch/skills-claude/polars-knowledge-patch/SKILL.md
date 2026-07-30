---
name: polars-knowledge-patch
description: Polars
version: 1.41.0
license: MIT
metadata:
  author: Nevaberry
---


# Polars compatibility guidance

Use this skill when upgrading or maintaining Python, Rust, or SQL code that
uses Polars. It highlights behavior changes that can silently alter schemas,
null handling, query results, serialization, or lazy execution.

## How to use this skill

1. Identify the Polars version pinned by the project manifest.
2. Inspect the migration notes before changing code that constructs frames,
   depends on inferred dtypes, or catches Polars exceptions.
3. Consult the task-specific reference for I/O, lazy execution, expressions,
   or SQL behavior.
4. Preserve explicit options when the application depends on historical
   defaults.
5. Run schema assertions and result-level tests after upgrading.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration and deprecations](references/migration-and-deprecations.md) | Constructor, API, default, runtime, and deprecation changes |
| [Data types and expressions](references/data-types-and-expressions.md) | Dtypes, nulls, nested values, temporal operations, aggregation, and sampling |
| [I/O and serialization](references/io-and-serialization.md) | Parquet, Arrow, CSV, Excel, JSON, cloud, database, Delta, and Iceberg |
| [Lazy execution and joins](references/lazy-execution-and-joins.md) | Schemas, streaming, optimization, sinks, joins, windows, and grouping |
| [SQL](references/sql.md) | SQL context, operators, functions, validation, and aggregate semantics |

## Breaking changes first

### Make row construction explicit

The `DataFrame` constructor infers orientation from data and schema dimensions
and warns when it infers rows. Mark heterogeneous row records explicitly:

```python
df = pl.DataFrame([[1, "a"], [2, "b"]], orient="row")
```

Byte scalars now broadcast across frame rows. Test constructors that previously
treated bytes as a non-broadcast input.

### Expect strict `Series` inference

Mixed incompatible values now raise even when the dtype is inferred. Use
`strict=False` only when common-type coercion is intended:

```python
s = pl.Series([1, 2, 3.5], strict=False)
```

With an explicit integer dtype, non-strict construction casts the float rather
than replacing it with a missing value. Strict nested casts also validate inner
list and array values.

### Audit timezone construction

Constructing a datetime column with a zoned dtype converts values into that
zone. It does not merely replace timezone metadata. Naive wall-clock values can
therefore shift.

### Distinguish `replace` from `replace_strict`

`replace` preserves the current dtype. Its `default` and `return_dtype`
parameters are deprecated. Use `replace_strict` for a mapping that may change
dtype:

```python
out = s.replace_strict(old, new, default=s)
```

Without `default`, every non-missing input must be mapped.

### Update pivot calls and output expectations

`DataFrame.pivot` uses `on` in place of `columns`; `on` is the first positional
argument, while `index` and `values` may be inferred. Multiple-value output
names omit the redundant pivot-column name. Missing pivot keys are retained.

### Handle out-of-bounds access deliberately

`get` and `gather` operations raise for an out-of-bounds index by default. Opt
into a missing result where required:

```python
value = s.list.get(1, null_on_oob=True)
```

### Separate serialization from JSON

Frame and expression serialization defaults to binary bytes. Use `BytesIO`, or
request `format="json"` explicitly. `DataFrame.write_json` produces
row-oriented JSON; serialized frames must be restored with
`DataFrame.deserialize`, not `pl.read_json`.

### Collect lazy schemas explicitly

Accessing `LazyFrame.schema`, `.dtypes`, `.columns`, or `.width` can perform
expensive schema resolution and emits `PerformanceWarning`. Resolve once:

```python
schema = lf.collect_schema()
```

### Recheck Parquet discovery

Parquet readers accept directories and enable Hive partitioning for directory
inputs. A file, glob, or file list does not enable it by default. Pass
`hive_partitioning=True` when those paths encode partition columns.

### Update SQL arithmetic assumptions

SQL `/` uses true division, so integer operands can produce fractions:

```sql
SELECT 1 / 2
```

The result is `0.5`. Revisit queries that relied on integer quotient behavior.

## Deprecation checklist

- Avoid new dependencies on `StringCache`; it is deprecated.
- Treat the dataframe interchange integration as transitional.
- Stop relying on cache-related arguments to `scan_ipc`.
- Remove `ddof` from `rolling_corr`; it is deprecated and ignored.
- Use `QueryOptFlags` instead of separate lazy optimization controls.
- Static tooling can detect deprecated Polars APIs through PEP 702 metadata.
- Update removed decimal activation and dtype-alias usage.
- Update old `pivot(columns=...)`, `replace(default=...)`, and
  `replace(return_dtype=...)` calls.

## High-value feature guide

### Streaming and sinks

The streaming engine is stable. It covers additional group-by aggregations,
first/last aggregation, grouped as-of joins, interpolation, covariance,
correlation, inferred-format `strptime`, PyArrow dataset sources, and Parquet
writes. Stable sink APIs include Parquet, Delta, and Iceberg paths.

`LazyFrame.map_batches` now starts with optimizations disabled. Enable only the
transformations that are valid for the callback.

### Schema reconciliation

Use `LazyFrame.match_to_schema` to reconcile a lazy result with an expected
schema before execution:

```python
lf = lf.match_to_schema({"id": pl.Int64})
```

This is especially useful at ingestion boundaries where columns or dtypes may
vary.

### Global aggregation interfaces

Window expressions may call `.over()` without partition keys, and
`group_by()` may be called without key expressions. Both forms apply their
aggregation interface to one global group.

```python
df.with_columns(total=pl.col("value").sum().over())
df.group_by().agg(pl.col("value").sum())
```

### Nested-value controls

List and Array operations offer explicit missing-value behavior:

- `contains(..., nulls_equal=True)` can match a missing search value.
- `any(ignore_nulls=...)` and `all(ignore_nulls=...)` select reduction
  semantics.
- `is_unique` works on nested values.
- `list.slice` broadcasts scalar input across rows.

Fixed-size `Array` columns can also be grouping keys.

### Excel and in-memory data

Excel reads default to the Calamine engine. Select `engine="xlsx2csv"` when
engine options are required. Excel and ODS readers accept raw bytes, Excel can
select a named table, and Excel writes accept file-like outputs.

### Cloud and database paths

Credential providers can supply cloud credentials to Parquet scans. Provider
objects are excluded from serialization, Azure account-key discovery is
opt-in, and boto3 endpoint URLs are honored. Partitioned Parquet datasets can
be written directly to cloud storage.

Database reads recognize `Int128`, DuckDB Arrow results can flow through a
SQLAlchemy selectable, and ADBC append mode creates a missing destination
table.

### SQL additions

SQL supports bit operations, discrete quantiles, Unicode normalization,
aggregate `FILTER`, and `STRING_AGG`. `LIKE` and `ILIKE` can span line breaks.
See the SQL reference for validation and exception changes.

## Verification targets

After an upgrade, prioritize tests that:

- assert constructor orientation and inferred dtypes;
- compare schemas before and after Parquet, Arrow, JSON, and Excel round trips;
- exercise missing values inside nested dtypes and grouped as-of joins;
- validate serialized-plan compatibility and restoration APIs;
- compare lazy and streaming results for the same query;
- assert SQL division, literal counts, pattern matching, and aggregates;
- verify exact exception classes when callers recover from Polars errors;
- check deterministic sampling after setting the global random seed.
