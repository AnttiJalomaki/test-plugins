# Lazy Execution, Streaming, and SQL

## Lazy planning and schema control

- **Frame-local SQL (1.0-upgrade).** `DataFrame.sql` and `LazyFrame.sql`
  resolve only their own frame, not other global frames. Use top-level
  `pl.sql("SELECT * FROM df1 CROSS JOIN df2", eager=True)` for multi-frame
  queries.
- **Lazy schema reconciliation (since 1.30.0).**
  `LazyFrame.match_to_schema` reconciles a lazy frame with an expected schema
  before execution, for example `lf.match_to_schema({"id": pl.Int64})`.
- **Central optimizer flags (since 1.30.0).** Lazy optimization controls live
  in `QueryOptFlags`. Pass that object instead of relying on separate
  optimization switches.
- **Conservative batches (since 1.40.0).** `LazyFrame.map_batches` defaults to
  no optimizations. Enable only transformations that are safe for the batch
  function.

## Streaming engine and sinks

- **Aggregation and Parquet execution (since 1.20.0).** The streaming engine
  executes first/last and additional `group_by` aggregations and can write
  through the Parquet sink.
- **Stable sink interface (since 1.30.0).** `sink_*` APIs are stable rather
  than experimental.
- **Broader operator support (since 1.40.0).** Streaming supports grouped
  as-of joins, native interpolation, `strptime(format=None)`, covariance and
  correlation, and PyArrow dataset sources.
- **Stable engine status (since 1.41.0).** The streaming engine is a stable,
  supported execution engine rather than an experimental one.

## SQL arithmetic and scalar functions

- **Bit operations (since 1.10.0).** SQL supports `bit_count` and bitwise
  `&`, `|`, and `xor`.
- **Discrete quantiles (since 1.10.0).** SQL supports `QUANTILE_DISC`, backed
  by discrete quantile interpolation.
- **Unicode normalization (since 1.20.0).** SQL supports the `NORMALIZE`
  function.
- **True division (since 1.41.0).** SQL `/` uses true-division semantics.
  Integer operands can produce fractions; `SELECT 1 / 2` yields `0.5`.

## SQL aggregation and pattern matching

- **Multiline patterns (since 1.20.0).** SQL `LIKE` and `ILIKE` match across
  line breaks, so `%` spans newline characters.
- **Literal counts (since 1.40.0).** `COUNT(<literal>)` returns the correct
  result; queries that count literals may produce different output.
- **Filtered and string aggregates (since 1.41.0).** SQL supports aggregate
  `FILTER` clauses and `STRING_AGG`, such as
  `SUM(value) FILTER (WHERE keep)` and `STRING_AGG(name, ',')`.

## SQL errors

- **Invalid `HAVING` placement (since 1.10.0).** `HAVING` outside a
  `GROUP BY` query raises `SQLSyntaxError`.
- **Invalid expression rejection (since 1.40.0).** `sql_expr` rejects invalid
  input instead of letting it pass into later planning.
