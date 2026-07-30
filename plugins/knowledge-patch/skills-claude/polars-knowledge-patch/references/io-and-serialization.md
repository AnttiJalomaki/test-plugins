# I/O and serialization

## Parquet reads and scans

- During the **1.0-upgrade**, `read_parquet` and `scan_parquet` accept
  directories. Hive partitioning is enabled by default for a directory and
  disabled by default for a file, glob, or list. Pass
  `hive_partitioning=True` when those other inputs should recover partition
  columns.
- Parquet readers decode `Float16` and load its column statistics (since
  1.10.0).
- Dtype validation is projection-aware (since 1.10.0): an invalid or
  unsupported dtype in an unprojected column does not block a projected read.
- An explicit Parquet `schema` survives planning and reaches the intermediate
  representation (since 1.10.0).
- `scan_parquet` accepts `cast_options` to control scan-time casting (since
  1.30.0).
- Writers can attach field metadata, and Parquet I/O reads and writes custom
  file-level metadata (since 1.30.0).
- `sink_parquet` writes Parquet field IDs (since 1.40.0).
- Column indexing works through the selected PyArrow reader for
  `read_parquet` and `read_csv` (since 1.41.0).
- The Parquet reader accepts MAP columns without a `LogicalType` annotation
  (since 1.41.0).
- A Parquet file with duplicate column names raises `DuplicateError` (since
  1.41.0).

## Arrow and dataframe interchange

- Arrow decimal arrays convert to Polars `Decimal`, not `Float64`, during the
  **1.0-upgrade**. Decimal support needs no activation and
  `Config.activate_decimals` was removed.
- Constructing a `Series` from a chunked Arrow struct consumes every chunk
  (since 1.10.0).
- A PyArrow table with duplicate column names raises `DuplicateError` (since
  1.20.0).
- Enum export produces an ordered Arrow dictionary (since 1.41.0), preserving
  the Enum ordering marker for consumers.
- The dataframe interchange protocol integration is deprecated (since
  1.40.0).

## Serialization, pickle, JSON, and UDFs

- `LazyFrame`, `DataFrame`, and `Expr` serialization defaults to binary bytes
  during the **1.0-upgrade**. Use `BytesIO` or pass `format="json"`.
- `DataFrame.write_json` emits row-oriented JSON and no longer has
  `row_oriented` or `pretty`. Read serialized frames with
  `DataFrame.deserialize`, not `pl.read_json`.
- UDF deserialization checks the Python version (since 1.10.0), making
  incompatible cross-version UDFs fail explicitly.
- General Polars pickle payloads can be loaded across Python versions (since
  1.20.0).
- Credential-provider objects are excluded from serialization (since 1.20.0);
  do not assume a serialized plan carries provider state.
- Empty JSON frame construction preserves its schema (since 1.30.0).
- Incompatible serialized DSL representations are rejected (since 1.30.0);
  persisted expressions and plans must match the reader.

## Excel and ODS

- `read_excel` defaults to the Calamine engine for every Excel format during
  the **1.0-upgrade**. Calamine does not accept `engine_options`; request
  `engine="xlsx2csv"` for options such as `skip_empty_lines`.
- `read_ods` and `read_excel` accept `drop_empty_rows` (since 1.10.0).
- `read_excel` can load a named Excel Table with `table_name`, and every
  Excel/ODS reader engine accepts raw bytes (since 1.20.0).
- `write_excel` accepts file-like output objects (since 1.20.0).

## Cloud storage

- AWS and GCP credential-provider utilities are available, and
  `scan_parquet` has an experimental `credential_provider` argument (since
  1.10.0).
- Automatic use of an Azure storage-account key is opt-in (since 1.20.0).
- Partitioned Parquet datasets can be written directly to cloud storage (since
  1.20.0).
- The `allow_invalid_certificates` storage option is honored by cloud I/O
  (since 1.20.0).
- When Polars gets AWS configuration from boto3, it loads `endpoint_url`
  (since 1.30.0), so configured S3-compatible endpoints are honored.

## Databases and table formats

- `scan_delta` and `read_delta` accept an existing `DeltaTable` object (since
  1.10.0).
- `read_database` consumes DuckDB Arrow output when a SQLAlchemy `Selectable`
  supplies the query (since 1.10.0).
- Database reads infer Polars `Int128` from databases that expose the type
  (since 1.30.0).
- Stable `sink_*` APIs are available since 1.30.0.
- `sink_delta` does not require `maintain_order=True` (since 1.40.0), allowing
  default ordering behavior.
- An Iceberg sink DSL and callback integrate Iceberg writes with the sink
  interface (since 1.40.0).
- ADBC append mode creates the destination table when it is absent (since
  1.41.0).

## CSV safety

- `read_csv` validates the length of `schema_overrides` (since 1.20.0) and
  raises for an invalid override sequence.
- `scan_csv(missing_columns="insert")` preserves data in columns that exist
  (since 1.40.0) instead of overwriting them with missing values.
