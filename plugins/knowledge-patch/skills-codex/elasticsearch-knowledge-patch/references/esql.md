# ES|QL

Use this reference for query construction, feature gating, response handling,
and time-series analytics. Features marked technical preview should not be
treated as stable application contracts.

## Sources, views, joins, and branching

### `LOOKUP JOIN` foundations

In 8.18.0, the technical-preview `LOOKUP JOIN` command combines an ES|QL result
with matching records from a lookup index. Use it to enrich results with
reference data or correlate events across indices.

In 9.1.0 it accepts aliases and mixed numeric join fields. ES|QL also adds a
technical-preview `COMPLETION` command, slow logging, and list/get query APIs.

In 9.2.0, `LOOKUP JOIN` accepts multiple join fields and, in technical preview,
expression predicates. Input may include remote indices, and a remote `ENRICH`
can follow the join:

```esql
FROM index1
| LOOKUP JOIN lookup_index ON field1, field2
```

In 9.3.0, full-text functions and Lucene-pushable conditions can operate on
lookup-index fields after `LOOKUP JOIN`.

### Join restrictions

In 9.1.0, ES|QL rejects search functions on non-standard index modes, and a
remote `ENRICH` cannot follow a `LOOKUP JOIN`. The latter restriction is
relaxed in 9.2.0 as described above.

### `FORK`

In 9.1.0, the technical-preview `FORK` command sends every input row through
multiple branches, merges the results, and adds an `_fork` discriminator:

```esql
FROM test
| FORK
  ( WHERE content:"fox" )
  ( WHERE content:"dog" )
| SORT _fork
```

Cross-cluster `FORK` support is released in 9.3.0. In 9.4.0, `FORK` and
subquery branches no longer receive implicit limits.

### Views

In 9.4.0, views are virtual indices whose fields come from reusable ES|QL
pipelines. A `FROM` clause can mix indices, views, and wildcards; each named
view runs its own pipeline. View CRUD is authorized as index actions, deletion
can target multiple views, and views cannot be queried when document- or
field-level security applies.

### External data sources

In 9.4.0, external sources add Azure and Google Cloud Storage plugins,
multi-endpoint Arrow Flight, and ORC alongside Parquet, CSV/TSV, and NDJSON.
Compressed input supports GZIP, Zstandard, BZIP2, LZ4, Snappy, and Brotli.
Azure, GCS, and S3 sources can use anonymous `auth=none`; CSV adds bracketed
multivalue parsing and configurable error policies.

### PromQL source and Prometheus endpoints

In 9.4.0, the technical-preview `PROMQL` source command runs PromQL and pipes
its result into the remainder of an ES|QL query:

```esql
PROMQL index=k8s-downsampled start="2026-02-17T08:00:00Z" end="2026-02-17T09:00:00Z" step=30m avg_bytes=(avg(rate(network.total_bytes_in[30m])))
| SORT avg_bytes DESC, step
```

The default-enabled Prometheus plugin adds technical-preview remote write at
`POST /_prometheus/api/v1/write` and instant-query, range-query, series, and
label endpoints below `/_prometheus/api/v1/`.

## Full text, expressions, and row processing

### Scoring and query functions

In 9.0.0, ES|QL can expose `METADATA _score` and score disjunctions of
full-text functions. Full-text and non-full-text conditions can participate in
the same disjunction, and scoring is no longer snapshot-only.

The same release adds a technical-preview `KQL` function, a term query, hash
functions, and options for `MATCH` and `QSTR`. Named identifier and pattern
parameters also leave snapshot status.

In 9.1.0, ES|QL adds technical-preview `MATCH_PHRASE`, list-form `LIKE`,
arbitrary intervals for `DATE_TRUNC`, `ROUND_TO`, and `::date` inline casts.
Full-text functions can be used in `STATS`, and `LIMIT` accepts parameters.

### Sampling and change detection

In 9.1.0, ES|QL adds a `SAMPLE` aggregation function, random sampling, and the
`change_point` processing command.

### General language additions

In 9.3.0, ES|QL adds `MV_INTERSECTION`, `GROUP BY ALL`, `network_direction`,
multiple patterns for `GROK`, parameterized `LIKE` and `RLIKE`, and an
`outputField` option for `TOP`. The histogram data type is released.

In 9.4.0, ES|QL adds `JSON_EXTRACT`, `MV_UNION`, `MV_DIFFERENCE`,
`MV_INTERSECTS`, `USER_AGENT`, `REGISTERED_DOMAIN`, `URI_PART`, and a sparkline
aggregation. Spatial additions are `ST_DIMENSION`, `ST_GEOMETRYTYPE`,
`ST_ISEMPTY`, `ST_BUFFER`, `ST_SIMPLIFY`, and
`ST_SIMPLIFYPRESERVETOPOLOGY`. `date_range` fields and timezone-aware date
formatting, conversion, and arithmetic are supported.

### Grouping and row behavior

In 9.4.0, technical-preview `LIMIT BY` limits rows per group and accepts
evaluatable grouping functions such as `BUCKET`. `SET approximate` enables
approximate analytical queries. `ROW` supports intra-row field references,
`MV_EXPAND` is generally available, and technical-preview
`unmapped_fields="load"` can load partially mapped fields.

## Dates, analytics, and spatial data

### `date_nanos`

In 9.0.0, `date_nanos` works with `IN`, date extraction, formatting and
difference functions, bucketing, comparison with millisecond dates, and
implicit casting.

### Statistical and spatial functions

In 9.0.0, ES|QL adds `STD_DEV`, spatial extent aggregation, `ST_ENVELOPE`,
`ST_XMIN`, `ST_XMAX`, `ST_YMIN`, and `ST_YMAX`, plus some statistics over
`aggregate_metric_double`.

In 9.2.0, it adds `ABSENT` and `ABSENT_OVER_TIME`, list-form `RLIKE`, and
`MIN`/`MAX` for unsigned longs. Geohash, geotile, and geohex grid types are
supported, including in `ST_INTERSECTS` and `ST_DISJOINT`.

### `aggregate_metric_double` semantics

In 9.4.0, non-native ES|QL aggregations such as `STD_DEV` can consume
`aggregate_metric_double` using the average derived from `sum` and
`value_count`. Native `min`, `max`, `sum`, `avg`, and `count` continue to use
their corresponding subfields.

## Time-series analytics

### Sliding windows and date controls

In 9.3.0, time-series aggregations accept an optional window as their second
argument. The window must be a multiple of the `TBUCKET` or `BUCKET` interval
and otherwise defaults to that interval. ES|QL also adds `TRANGE`; timezone
support for `DATE_TRUNC`, `BUCKET`, `TBUCKET`, and `DATE_DIFF`; and locale and
timezone arguments for `DATE_PARSE`.

```esql
TS metrics
| WHERE TRANGE(1h)
| STATS avg(rate(requests, 10m)) BY TBUCKET(1m), host
```

In 9.4.0, windows may be smaller than their bucket and need not be an exact
multiple. Target-count `TBUCKET` can omit explicit bounds when the request
supplies a timestamp range:

```esql
TS metrics | STATS AVG(RATE(requests, 15m)) BY TBUCKET(10m), host
```

The 9.4.0 default `aggregate` downsampling method stores the first counter
value and auxiliary documents for detected resets, preserving resets in later
rate calculations. `last_value` retains its storage-oriented behavior.

### Metric and series discovery

In 9.4.0, `METRICS_INFO` after a `TS` source returns one row per metric with
its data stream, unit, metric and field types, and dimension fields. `TS_INFO`
returns one row per metric-and-series pair and a `dimensions` JSON object with
the series labels:

```esql
TS my_data_stream
| TS_INFO
| SORT metric_name, dimensions
```

## Vectors and inference inside ES|QL

### Dense vectors and KNN

In 9.2.0, ES|QL adds the `dense_vector` field type and a KNN function,
`v_hamming` for Hamming distance, and `v_magnitude` for vector magnitude.

In 9.3.0, it adds vector-similarity functions; KNN accepts `k` and
`visit_percentage`.

In 9.4.0, dense vectors support equality, inequality, `COALESCE`, arithmetic,
`SUM`, `COUNT`, `PRESENT`, and `ABSENT`. Dense-vector functions are generally
available.

### Text embedding, reranking, and snippets

In 9.3.0, technical-preview `CHUNK` accepts optional `chunking_settings`, while
technical-preview `TOP_SNIPPETS` returns the best snippets for a field.
`TEXT_EMBEDDING` and `SCORE` are enabled in release builds; the inference
command supports cross-cluster search; and `COMPLETION` and `RERANK` have usage
limits.

In 9.4.0, `TEXT_EMBEDDING` and `RERANK` are generally available. ES|QL adds an
MMR diversification command.

## Request, schema, and result behavior

### Schema and text output

In 9.0.0, `CATEGORIZE` accepts nulls and multiple groupings, and initial
unmapped-field support is available. `CASE`, `GREATEST`, and `LEAST` perform
implicit numeric casting. `RENAME` processes sequentially like `EVAL`, and text
formats drop null columns.

### Partial and asynchronous results

In 9.0.0, EQL supports partial shard results. Async ES|QL can return partial
results on demand, async get supports formatting, and in-progress
cross-cluster responses include CCS metadata.

### Cross-cluster status

ES|QL cross-cluster querying is generally available in 8.19.0 rather than a
technical-preview feature.

### Request parameters and metadata

In 9.2.0, ES|QL adds a `SET` instruction, multivalued query parameters, and
`include_execution_metadata`; `_tsid` is available as metadata. Profiling is
rejected for text response formats.

### `INLINE STATS`

In 9.2.0, `INLINE STATS` is available in release builds as a technical preview,
including filters and cross-cluster search support.
