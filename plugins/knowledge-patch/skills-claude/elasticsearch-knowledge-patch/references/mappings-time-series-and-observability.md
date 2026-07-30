# Mappings, Time Series, and Observability

Use this reference for field mappings, synthetic source, TSDB and LogsDB
storage, OpenTelemetry ingestion, and operational diagnostics.

## Contents

- [Mapping behavior](#mapping-behavior-and-guardrails)
- [Synthetic source](#synthetic-source-and-recovery-source)
- [Doc-values skippers](#doc-values-skippers)
- [Time-series identity and metrics](#time-series-index-identity)
- [Telemetry ingestion](#opentelemetry-and-prometheus-ingestion)
- [LogsDB and diagnostics](#logsdb-behavior)

## Mapping behavior and guardrails

### New and expanded field types

- Technical-preview `pattern_text` mappings arrive in 9.2.0.
- `exponential_histogram` arrives in 9.3.0 for OpenTelemetry exponential
  histograms. ES|QL supports `PERCENTILES`, `AVG`, `MIN`, `MAX`, and `SUM`:

```http
PUT metrics
{"mappings":{"properties":{"latency":{"type":"exponential_histogram"}}}}
```

- A dedicated T-Digest field type arrives in 9.3.0 and can be a metric in
  time-series data streams.
- The histogram data type is released for ES|QL in 9.3.0.
- `date_range` fields gain ES|QL support in 9.4.0.
- `flattened` fields gain declared `properties`, passthrough mapped subfields,
  and an accurate-leaf-arrays option in 9.4.0.

### Limits and malformed values

- The nested-field limit increases to 100 in 9.3.0.
- `index.mapping.nested_parents.limit` independently caps nested parents as of
  9.3.0.
- Mappings can opt to ignore a field when its indexed name exceeds the length
  limit (9.3.0).
- For an ignored dynamic array field, `_ignored` stores the complete field path
  as of 9.2.0.
- In 9.4.0, an `ignore_malformed` date field no longer silently ignores an
  object or array value.
- Metadata field definitions reject `type`, `fields`, `copy_to`, and `boost`.
  `_source.mode` is now a no-op.

### Geometry and analysis

- WKT geometries can explicitly include Z and M attributes in 9.1.0.
- The ICU transform analysis plugin accepts custom rulesets in 9.4.0.
- Snowball stemmers and the Nori Korean dictionary changed. `german2` now
  aliases the `german` Snowball stemmer, and the `persian` analyzer stems by
  default. Regression-test analyzer output during upgrades.

## Synthetic source and recovery source

- Sparse vectors can be kept outside `_source` in 9.0.0.
- Synthetic-source indices gain a setting to skip recovery source in 9.0.0.
- Synthetic recovery source is enabled by default when synthetic source is
  enabled in 9.1.0.
- Text and `match_only_text` multi-fields are no longer stored by default for
  synthetic source in 9.1.0.
- Normalized `keyword` fields now use native synthetic source.
- New indices enable `exclude_source_vectors` by default. Explicit vector
  retrieval and reindex behavior are covered in the vector reference.

## Doc-values skippers

In 9.3.0, fields with `index: false` and `doc_values: true` can use a sparse
doc-values index when `index.mapping.use_doc_values_skipper` is enabled.

- The general default is `false`.
- The TSDB default is `true`.
- In TSDB, skippers replace separate indexes for `@timestamp`, dimensions, and
  `_tsid` unless explicitly disabled.
- LogsDB defaults `index.mapping.use_doc_values_skipper` to `true` in 9.4.0.

Review range-query and storage performance before changing the default.

## Time-series index identity

Time-series mode adds synthetic IDs in 9.4.0:

- `_id` is not indexed.
- A Bloom filter detects duplicate documents during ingestion.
- ID-dependent operations resolve documents from timestamps and dimensions.
- New TSDB indices disable sequence numbers.
- Synthetic-ID indices support nested documents and `best_compression`.

Nested fields have been supported in `time_series` indices since 9.1.0.
Applications that rely on explicit IDs, `_seq_no`, updates, deletes, or
optimistic concurrency must test their workflow against these TSDB semantics.

## Time-series metrics and downsampling

- Time-series aggregation functions accept sliding windows in 9.3.0.
- Window and bucket restrictions are relaxed in 9.4.0; see the ES|QL
  reference.
- Data-stream lifecycle and ILM can select downsampling methods in 9.3.0.
- The 9.4.0 default `aggregate` method stores a counter's first value and
  auxiliary reset documents so later rate calculations preserve resets.
  `last_value` retains its storage-oriented behavior.
- `aggregate_metric_double` native ES|QL aggregations use their corresponding
  subfields. Other aggregations derive an average from `sum` and
  `value_count` as of 9.4.0.

## OpenTelemetry and Prometheus ingestion

- The OTel metrics field limit is 10,000 in 9.0.0.
- A technical-preview `/_otlp/v1/metrics` endpoint accepts OTLP metrics
  directly in 9.2.0.
- The OTLP endpoint now maps histograms to `exponential_histogram` by default.
- New OTel data streams enable failure stores by default in 9.2.0.
- In 9.4.0 the default-enabled Prometheus plugin provides technical-preview
  remote write and query endpoints; see the ES|QL reference for paths and
  PromQL integration.

## LogsDB behavior

- LogsDB can route on sort fields and configure index sorting through index
  settings in 9.0.0.
- LogsDB is conditionally enabled for `logs-*-*` streams when enabling
  conditions are met.
- LogsDB and TSDB text fields omit norms.
- LogsDB defaults `index.mapping.use_doc_values_skipper` to `true` in 9.4.0.
- Shrunk LogsDB and TSDB indices have a merge bug in 9.1.0 and 9.1.1; use
  9.1.2 or the compatibility-reference workaround.

## Metrics and diagnostics

- Thread-pool telemetry includes utilization and queue-latency metrics as of
  9.1.0.
- Reindex metrics report seconds rather than milliseconds in 9.0.0.
- Node and index stats expose dense-vector off-heap usage in 9.1.0.
- `/_security/stats` exposes document-level-security cache usage, hits, misses,
  and timing information in 9.2.0.
- Cat APIs add a circuit-breakers endpoint in 9.3.0.
- Shard-capacity health thresholds are configurable in 9.3.0.
- Query logging covers `_search`, ES|QL, EQL, and SQL in 9.4.0.
- A search-task watchdog can log hot threads for slow search tasks in 9.4.0.
- ES|QL query logging is deprecated starting in 9.4.2; avoid making it a new
  monitoring dependency.
