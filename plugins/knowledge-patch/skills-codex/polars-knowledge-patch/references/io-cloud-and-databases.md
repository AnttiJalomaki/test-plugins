# I/O, Cloud Storage, and Databases

## Parquet reads and schemas

- **Directories and Hive partitions (1.0-upgrade).** `read_parquet` and
  `scan_parquet` accept directories and enable Hive partitioning by default
  for them. A file, glob, or list of files leaves it disabled; pass
  `hive_partitioning=True` when those paths encode partition columns.
- **`Float16` decoding (since 1.10.0).** Parquet readers decode `Float16`,
  including its column statistics.
- **Projection-aware validation (since 1.10.0).** Parquet reads do not validate
  dtypes for unprojected columns, so an unused incompatible column does not
  block a projected read.
- **Explicit schemas (since 1.10.0).** A Parquet `schema` argument is retained
  in the intermediate representation during planning.
- **Scan cast controls (since 1.30.0).** `scan_parquet` accepts
  `cast_options` for scan-time casting behavior.
- **PyArrow column indices (since 1.41.0).** `read_parquet` supports
  index-based column selection when the PyArrow reader is selected.
- **Unannotated MAP values (since 1.41.0).** The Parquet reader accepts MAP
  columns without a `LogicalType` annotation.
- **Duplicate columns (since 1.41.0).** A Parquet file with duplicate column
  names raises `DuplicateError`, allowing specific invalid-schema handling.

## Parquet writes and metadata

- **Cloud partitioned writes (since 1.20.0).** Partitioned Parquet datasets can
  be written directly to cloud storage.
- **Custom metadata (since 1.30.0).** Parquet writers can attach field
  metadata, and Parquet I/O reads and writes custom file-level metadata.
- **Field IDs (since 1.40.0).** `sink_parquet` writes Parquet field IDs and
  preserves that schema information.

## CSV and line-oriented input

- **Schema override validation (since 1.20.0).** `read_csv` validates the
  length of `schema_overrides`; a sequence of invalid length raises.
- **Safe missing-column insertion (since 1.40.0).**
  `scan_csv(missing_columns="insert")` preserves data in columns that exist
  instead of overwriting them with nulls.
- **PyArrow CSV column indices (since 1.41.0).** `read_csv` handles
  index-based column selection through the PyArrow reader.

## Excel and ODS

- **Calamine default (1.0-upgrade).** `read_excel` uses Calamine for every
  Excel format by default. Calamine does not accept `engine_options`; request
  `engine="xlsx2csv"` for options such as `skip_empty_lines`.
- **Empty rows (since 1.10.0).** `read_ods` and `read_excel` accept
  `drop_empty_rows`.
- **Tables and memory I/O (since 1.20.0).** `read_excel` can load a named Excel
  Table with `table_name`. Every `read_excel` and `read_ods` engine accepts
  raw bytes, and `write_excel` accepts file-like outputs.

## Cloud credentials and transport

- **Credential providers (since 1.10.0).** AWS and GCP credential-provider
  utility classes are available. `scan_parquet` has an experimental
  `credential_provider` argument for supplying one.
- **Azure key opt-in (since 1.20.0).** Automatic use of an Azure storage
  account key is opt-in.
- **Provider serialization boundary (since 1.20.0).** Credential-provider
  objects are excluded from serialization; serialized plans and objects do
  not carry provider state.
- **Invalid certificates (since 1.20.0).** Cloud I/O honors the
  `allow_invalid_certificates` storage option rather than ignoring it.
- **Boto3 endpoints (since 1.30.0).** AWS configuration obtained through
  boto3 includes `endpoint_url`, so configured S3-compatible endpoints are
  honored.

## Delta Lake and Iceberg

- **Delta objects as sources (since 1.10.0).** `scan_delta` and `read_delta`
  accept a `DeltaTable` object directly.
- **Unordered Delta sinks (since 1.40.0).** `sink_delta` does not require
  `maintain_order=True`; writes can use default ordering.
- **Iceberg sink (since 1.40.0).** Polars provides an Iceberg sink DSL and
  callback for writing through the sink interface.

## Database integration

- **DuckDB Arrow via SQLAlchemy (since 1.10.0).** `read_database` consumes
  DuckDB Arrow output when the query is a SQLAlchemy `Selectable`.
- **ADBC append creation (since 1.41.0).** An ADBC write in append mode creates
  the destination table when it is missing.

## Deprecated I/O controls

- **IPC scan caching (since 1.40.0).** Cache-related `scan_ipc` arguments are
  deprecated and should not be relied on.
