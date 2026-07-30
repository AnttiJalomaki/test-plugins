# SQL

## Query context and validation

- During the **1.0-upgrade**, `DataFrame.sql` and `LazyFrame.sql` see only
  their own frame and cannot resolve global frames. Use top-level SQL for
  multi-frame queries:

  ```python
  pl.sql("SELECT * FROM df1 CROSS JOIN df2", eager=True)
  ```

- `HAVING` outside a `GROUP BY` query raises `SQLSyntaxError` (since 1.10.0).
- `sql_expr` rejects invalid input rather than admitting it to a plan (since
  1.40.0).

## Operators and scalar functions

- SQL supports `bit_count` and bitwise `&`, `|`, and `xor` (since 1.10.0).
- String expressions provide `str.normalize`, and SQL provides `NORMALIZE`
  (since 1.20.0):

  ```python
  df.select(pl.col("text").str.normalize())
  ```

- `LIKE` and `ILIKE` match across line breaks (since 1.20.0), so `%` can span
  newline characters.
- SQL `/` uses true-division semantics (since 1.41.0). Integer operands can
  produce fractions; `SELECT 1 / 2` evaluates to `0.5`.

## Aggregates

- `QUANTILE_DISC` is available through discrete quantile interpolation (since
  1.10.0).
- `COUNT(<literal>)` returns the correct literal-expression count (since
  1.40.0), which may change historical query output.
- Aggregate `FILTER` clauses and `STRING_AGG` are supported (since 1.41.0):

  ```sql
  SELECT
    SUM(value) FILTER (WHERE keep),
    STRING_AGG(name, ',')
  FROM frame
  ```
