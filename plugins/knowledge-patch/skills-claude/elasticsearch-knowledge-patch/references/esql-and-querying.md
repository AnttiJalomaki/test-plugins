# ES|QL and Distributed Querying

Use this reference when authoring ES|QL, EQL, cross-cluster, or cross-project
query workflows. Preserve technical-preview status in durable interfaces.

## Contents

- [Result and distributed-query controls](#result-completeness-and-response-controls)
- [Joins and branching](#joins-enrichment-branching-and-subqueries)
- [Full-text and vector querying](#full-text-search-scoring-and-query-syntax)
- [Time-series and aggregation](#time-series-sources-and-windowing)
- [Functions and pipeline authoring](#dates-numerics-multivalue-fields-and-nulls)
- [Views and PromQL](#views-promql-and-prometheus-apis)
- [External data](#external-data-sources)

## Result completeness and response controls

- ES|QL may return partial results by default. Inspect `is_partial`, use
  `allow_partial_results=false` when completeness is required, or disable
  partial results cluster-wide with `esql.query.allow_partial_results: false`.
- Async ES|QL gained opt-in partial results in 9.0.0; async get also supports
  response formatting, and in-progress cross-cluster responses carry CCS
  metadata.
- Async result retrieval adds `return_intermediate_results` in 9.4.0, while
  async task status exposes `keep_alive`.
- EQL supports partial shard results as of 9.0.0 and now defaults
  `allow_partial_search_results` to `true`.
- ES|QL requests can include execution metadata with
  `include_execution_metadata` (9.2.0). `_tsid` is available as a metadata
  field.
- Profiling is rejected when an ES|QL request asks for a text response format.
- Text output formats omit columns whose values are entirely null (9.0.0).
- `CATEGORIZE` accepts null inputs and multiple grouping expressions.
- Unmapped-field support began in 9.0.0; `unmapped_fields="load"` is a
  technical-preview option for loading partially mapped fields in 9.4.0.

## Index patterns, modes, and distributed execution

- Quote an entire remote index expression or none of it:
  `FROM "remote:index"` and `FROM remote:index` are valid, while
  `FROM remote:"index"` is invalid.
- When `skip_unavailable: true`, all remote runtime errors, including missing
  indices, are non-fatal; the cluster is marked skipped or partial.
- ES|QL cross-cluster query support is generally available in 8.19.0.
- Search functions are rejected on non-standard index modes (9.1.0).
- Cross-project search and `project_routing` support `_search`,
  `_async_search`, `_msearch`, EQL, field capabilities, SQL, and JDBC as of
  9.3.0. PIT creation and closure can span projects, and cross-project search
  defaults to minimizing round trips.
- In 9.4.0, project routing also covers templated searches, data streams,
  scrolls, and the SQL CLI. SQL CLI and JDBC clients support API-key
  authentication.
- Stateful cross-cluster operation disables `_delete_by_query` and
  `_update_by_query` (9.3.0).

## Joins, enrichment, branching, and subqueries

### `LOOKUP JOIN`

The technical-preview `LOOKUP JOIN` introduced in 8.18.0 enriches rows with
matching lookup-index records or correlates records across indices.

- Aliases and mixed numeric join-field types are supported in 9.1.0.
- Multiple equality join fields are supported in 9.2.0:

```esql
FROM index1
| LOOKUP JOIN lookup_index ON field1, field2
```

- Expression predicates are technical preview in 9.2.0.
- The input can include remote indices as of 9.2.0.
- In 9.1.0 a remote `ENRICH` could not follow `LOOKUP JOIN`; it can in 9.2.0.
- Full-text functions and Lucene-pushable conditions can query lookup-index
  fields after the join in 9.3.0.

### `FORK`

The technical-preview `FORK` command introduced in 9.1.0 sends every input row
through multiple branches, merges them, and adds an `_fork` discriminator:

```esql
FROM test
| FORK
  ( WHERE content:"fox" )
  ( WHERE content:"dog" )
| SORT _fork
```

Cross-cluster `FORK` is released in 9.3.0. In 9.4.0, `FORK` and subquery
branches no longer receive implicit limits.

### Inline aggregation and row limits

- `INLINE STATS` is available in release builds as a technical preview in
  9.2.0, including filters and cross-cluster search.
- Technical-preview `LIMIT BY` in 9.4.0 limits rows per group and accepts
  evaluatable grouping functions such as `BUCKET`.
- `LIMIT` accepts parameters as of 9.1.0.
- `ROW` supports references to fields created earlier in the same row as of
  9.4.0.

## Full-text search, scoring, and query syntax

- ES|QL can request `METADATA _score` and score disjunctions of full-text
  functions as of 9.0.0. Full-text and non-full-text predicates can occur in
  the same disjunction.
- Full-text functions can be used within `STATS` in 9.1.0.
- Technical-preview `MATCH_PHRASE` arrived in 9.1.0.
- The technical-preview `KQL` function, a term-query function, hash
  functions, and new `MATCH` and `QSTR` options arrived in 9.0.0.
- Named identifier and pattern parameters left snapshot-only status in 9.0.0.
- List-form `LIKE` arrived in 9.1.0; list-form `RLIKE` in 9.2.0.
- `LIKE` and `RLIKE` patterns can be parameterized in 9.3.0.
- Technical-preview `TOP_SNIPPETS` returns the best snippets for a field in
  9.3.0.
- The technical-preview `CHUNK` function accepts optional
  `chunking_settings` in 9.3.0.
- `TEXT_EMBEDDING` and `SCORE` are enabled in release builds in 9.3.0.
  `TEXT_EMBEDDING` becomes generally available in 9.4.0.
- The inference command supports cross-cluster search in 9.3.0.
- Technical-preview `COMPLETION` was added in 9.1.0; `COMPLETION` and `RERANK`
  have usage limits in 9.3.0. `RERANK` is generally available in 9.4.0.
- ES|QL adds an MMR diversification command in 9.4.0.

## Vector expressions and kNN

- ES|QL supports the `dense_vector` field type and a KNN function in 9.2.0.
- `v_hamming` measures dense-vector Hamming distance and `v_magnitude`
  computes magnitude (9.2.0).
- Vector-similarity functions and KNN options `k` and `visit_percentage`
  arrived in 9.3.0.
- Dense-vector functions are generally available in 9.4.0.
- Dense-vector values support equality, inequality, `COALESCE`, arithmetic,
  `SUM`, `COUNT`, `PRESENT`, and `ABSENT` in 9.4.0.

## Time-series sources and windowing

- Time-series aggregation functions accept an optional window as their second
  argument in 9.3.0. Initially the window had to be a multiple of the
  `TBUCKET` or `BUCKET` interval and defaulted to that interval:

```esql
TS metrics
| WHERE TRANGE(1h)
| STATS avg(rate(requests, 10m)) BY TBUCKET(1m), host
```

- In 9.4.0 a window may be smaller than its bucket and need not be an exact
  multiple:

```esql
TS metrics | STATS AVG(RATE(requests, 15m)) BY TBUCKET(10m), host
```

- Target-count `TBUCKET` can omit bounds when the request supplies a timestamp
  range (9.4.0).
- `TRANGE` arrived in 9.3.0.
- `DATE_TRUNC`, `BUCKET`, `TBUCKET`, and `DATE_DIFF` accept timezones in
  9.3.0; `DATE_PARSE` accepts locale and timezone.
- `DATE_TRUNC` accepts arbitrary intervals as of 9.1.0.
- After a `TS` source, `METRICS_INFO` returns one row per metric with its data
  stream, unit, metric type, field type, and dimensions (9.4.0).
- `TS_INFO` returns one row per metric-and-series pair, with a `dimensions`
  JSON object:

```esql
TS my_data_stream
| TS_INFO
| SORT metric_name, dimensions
```

## Aggregation, grouping, and approximation

- `STD_DEV` and statistics over `aggregate_metric_double` arrived in 9.0.0.
- For non-native aggregations such as `STD_DEV`, 9.4.0 derives an
  `aggregate_metric_double` average from `sum / value_count`. Native `min`,
  `max`, `sum`, `avg`, and `count` keep using their corresponding subfields.
- `SAMPLE`, random sampling, and the `change_point` processing command arrived
  in 9.1.0.
- `MIN` and `MAX` accept unsigned longs in 9.2.0.
- `GROUP BY ALL` arrived in 9.3.0.
- `SET approximate` enables approximate analytical queries in 9.4.0.
- A sparkline aggregation is available in 9.4.0.
- `TOP` accepts an `outputField` option in 9.3.0.
- Histogram data type support is released in 9.3.0.

## Dates, numerics, multivalue fields, and nulls

- `date_nanos` supports `IN`, extraction, formatting, difference, bucketing,
  comparison with millisecond dates, and implicit casts as of 9.0.0.
- `CASE`, `GREATEST`, and `LEAST` implicitly cast numeric arguments in 9.0.0.
- `ROUND_TO` and `::date` inline casts arrived in 9.1.0.
- `MV_INTERSECTION` arrived in 9.3.0.
- `MV_UNION`, `MV_DIFFERENCE`, and `MV_INTERSECTS` arrived in 9.4.0.
- `MV_EXPAND` is generally available in 9.4.0.
- `date_range` fields and timezone-aware date formatting, conversion, and
  arithmetic are supported in 9.4.0.
- `ABSENT` and `ABSENT_OVER_TIME` arrived in 9.2.0.
- `PRESENT` and `ABSENT` work with dense vectors in 9.4.0.

## String, network, JSON, and spatial functions

- `network_direction`, multiple `GROK` patterns, parameterized `LIKE` and
  `RLIKE`, and `TOP outputField` arrived in 9.3.0.
- `JSON_EXTRACT`, `USER_AGENT`, `REGISTERED_DOMAIN`, and `URI_PART` arrived in
  9.4.0.
- Spatial extent aggregation plus `ST_ENVELOPE`, `ST_XMIN`, `ST_XMAX`,
  `ST_YMIN`, and `ST_YMAX` arrived in 9.0.0.
- Geohash, geotile, and geohex grid types, including use with
  `ST_INTERSECTS` and `ST_DISJOINT`, arrived in 9.2.0.
- `ST_DIMENSION`, `ST_GEOMETRYTYPE`, `ST_ISEMPTY`, `ST_BUFFER`,
  `ST_SIMPLIFY`, and `ST_SIMPLIFYPRESERVETOPOLOGY` arrived in 9.4.0.

## Pipeline authoring and management

- `RENAME` processes assignments sequentially like `EVAL` as of 9.0.0.
- A `SET` instruction and multivalued query parameters arrived in 9.2.0.
- ES|QL slow logging plus list/get query APIs arrived in 9.1.0.
- Query logging covers `_search`, ES|QL, EQL, and SQL in 9.4.0, but the ES|QL
  query log itself is deprecated starting in 9.4.2.
- A search-task watchdog can log hot threads for slow searches in 9.4.0.

## Views, PromQL, and Prometheus APIs

### Views

Views in 9.4.0 are virtual indices whose fields come from reusable ES|QL
pipelines. `FROM` can mix views, indices, and wildcards; each view runs its own
pipeline. CRUD is authorized as index actions, delete can target multiple
views, and views cannot be queried when document- or field-level security is
active.

### PromQL and Prometheus

Technical-preview `PROMQL` is an ES|QL source command in 9.4.0:

```esql
PROMQL index=k8s-downsampled start="2026-02-17T08:00:00Z" end="2026-02-17T09:00:00Z" step=30m avg_bytes=(avg(rate(network.total_bytes_in[30m])))
| SORT avg_bytes DESC, step
```

The default-enabled Prometheus plugin provides technical-preview remote write
at `POST /_prometheus/api/v1/write`, plus instant-query, range-query, series,
and label endpoints under `/_prometheus/api/v1/`.

## External data sources

In 9.4.0, external sources include:

- Azure and Google Cloud Storage plugins and multi-endpoint Arrow Flight.
- ORC, Parquet, CSV/TSV, and NDJSON formats.
- GZIP, Zstandard, BZIP2, LZ4, Snappy, and Brotli compression.
- Anonymous `auth=none` for Azure, GCS, and S3.
- CSV bracketed multivalue parsing and configurable error policies.
