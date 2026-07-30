# Serialization, Runtime, and Arrow

## Frame and expression serialization

- **Binary serialization default (1.0-upgrade).** `LazyFrame`, `DataFrame`,
  and `Expr` serialize to binary bytes by default. Use `BytesIO` for default
  `serialize` and `deserialize`, or pass `format="json"`.
- **JSON is data, not serialized state (1.0-upgrade).**
  `DataFrame.write_json` emits only row-oriented JSON and no longer accepts
  `row_oriented` or `pretty`. Restore a serialized frame with
  `DataFrame.deserialize`, not `pl.read_json`.
- **Reader-compatible DSL (since 1.30.0).** Deserialization rejects a DSL
  representation incompatible with its reader. Persist expressions and lazy
  plans only across compatible representations.
- **Empty JSON schema (since 1.30.0).** Constructing a frame from empty JSON
  preserves its schema instead of dropping schema information.

## Python serialization and runtime support

- **UDF runtime check (since 1.10.0).** Deserializing a UDF checks its Python
  version, surfacing cross-version incompatibility instead of silently
  accepting it.
- **Minimum Python (since 1.10.0).** Python 3.9 is the oldest supported
  runtime.
- **Cross-version pickle (since 1.20.0).** Polars pickle payloads can be loaded
  across Python versions rather than only by the version that created them.
- **Python 3.13 (since 1.20.0).** Python 3.13 is an officially supported
  runtime.
- **Exception fidelity (since 1.30.0).** Python exceptions crossing Polars
  execution preserve their original type and traceback, allowing specific
  handlers and diagnostics.
- **NumPy conversion signatures (since 1.41.0).** `DataFrame.__array__` and
  `Series.__array__` match NumPy's signature and accept standard conversion
  arguments from NumPy callers.

## Arrow construction and schema integrity

- **Chunked struct construction (since 1.10.0).** Constructing a `Series` from
  a chunked Arrow struct consumes every chunk rather than omitting later
  chunks.
- **Duplicate table fields (since 1.20.0).** Constructing from a PyArrow table
  with duplicate column names raises `DuplicateError`, distinguishing the
  invalid schema from other conversion failures.
- **Ordered Enum dictionaries (since 1.41.0).** Exporting Enum data to Arrow
  produces an ordered dictionary and preserves the Enum ordering marker for
  Arrow consumers.
