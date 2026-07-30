# Data streams, mappings, and ingest

Use this reference when defining templates, handling rejected documents,
building ingest pipelines, or configuring LogsDB, TSDB, and telemetry data.

## Failure stores

### Enablement and querying

In 8.19.0, data streams can redirect documents rejected by ingest pipelines or
mapping conflicts into a failure store. Enable one for an existing stream:

```http
PUT _data_stream/logs-test-apache/_options
{
  "failure_store": {
    "enabled": true
  }
}
```

For new streams, set
`template.data_stream_options.failure_store.enabled` in a component or index
template. Query failure-store backing indices with `::failures`, including
`logs-*::failures` in search and
`FROM my_data_stream*::failures` in ES|QL.

### Lifecycle and visibility

In 9.1.0, data streams gain dedicated failure-store lifecycle configuration
and default retention for failure indices. The get data stream API also
reports each stream's index mode.

### Default enablement for logs and telemetry

In 9.2.0, new `logs-*-*`, OTel, and APM data streams enable failure stores by
default. An invalid document routed to a `logs-*-*` failure store returns
`201 Created` with `"failure_store": "used"` rather than `400 Bad Request`.
Existing streams still require manual enablement.

### Remediation and cross-cluster use

In 9.2.0, the new `recover_failure_document` ingest processor supports
remediation of failure-store documents.

In 9.4.0, failure-store indices can participate in cross-cluster search.

## Data streams and index modes

### Streams restrictions

In 9.2.0, streams can be enabled only when no conflicting indices exist. After
enablement, indexing is restricted to child streams.

### LogsDB and OTel modes

In 9.0.0, LogsDB can route on sort fields and configure index sorting through
index settings. The field limit for OTel metrics is 10,000.

### New stream types

In 9.4.0, data streams add the `logs.otel` and `logs.ecs` stream types.

## Ingest request and processor behavior

### Parse-error source control

In 9.0.0, create, index, update, and bulk REST APIs accept
`include_source_on_error` to control whether parsing-error responses include a
document's source. It defaults to `true`.

### Reroute metadata

In 9.0.0, the reroute processor can set `type`, and simulate ingest responses
include ignored fields.

### Append and conditional processors

In 9.2.0, the append processor adds `copy_from` and an option to ignore empty
values. Conditional processors can use the Fields API.

### Simulated mappings and asset timestamps

In 9.2.0, simulate ingest accepts `merge_type` and returns the effective
mapping. Index templates, component templates, and pipelines expose created
and modified dates.

### CEF parsing

In 9.3.0, the `cef` ingest processor parses Common Event Format into structured
fields such as device vendor, product, version, signature ID, name, severity,
and extensions.

### Analysis and structured-text input

In 9.4.0, the ICU transform analysis plugin accepts custom rulesets. The Grok
processor adds `validate_only` to skip field extraction, and text-structure
endpoints accept nested NDJSON records.

## Mapping features and guardrails

### Time-series and synthetic source

In 9.1.0, `nested` fields are supported in `time_series` indices. Synthetic
recovery source is enabled by default with synthetic source. Text and
`match_only_text` multi-fields are no longer stored by default for synthetic
source.

### Pattern text and ignored paths

In 9.2.0, mappings gain a technical-preview `pattern_text` field mapper. For
ignored dynamic array fields, `_ignored` stores the full field path.

### Nested-field limits and long names

In 9.3.0, the nested-field limit increases to 100.
`index.mapping.nested_parents.limit` can separately cap nested parents.
Mappings can opt to ignore a field whose indexed name exceeds the length
limit.

### Flattened fields and default behavior

In 9.4.0, `flattened` fields gain declared `properties`, passthrough mapped
subfields, and an option for accurate leaf arrays. LogsDB defaults
`index.mapping.use_doc_values_skipper` to `true`. `ignore_malformed` date
fields no longer silently ignore object or array values.

### Explicit WKT dimensions

In 9.1.0, WKT geometries can explicitly specify Z and M attributes.

## Metrics and time-series storage

### Native histogram fields

In 9.3.0, `exponential_histogram` stores OpenTelemetry exponential histograms
and supports ES|QL `PERCENTILES`, `AVG`, `MIN`, `MAX`, and `SUM`.
Elasticsearch also adds a dedicated T-Digest field type usable as a metric in
time-series data streams.

```http
PUT metrics
{"mappings":{"properties":{"latency":{"type":"exponential_histogram"}}}}
```

### Doc-values skippers

In 9.3.0, fields with `index: false` and `doc_values: true` can use a sparse
doc-values index when `index.mapping.use_doc_values_skipper` is enabled. The
setting defaults to `false` generally and `true` for TSDB. In TSDB, skippers
replace separate indexes for `@timestamp`, dimensions, and `_tsid` unless
explicitly disabled.

### Time-series index identity

In 9.4.0, time-series mode gains synthetic IDs that avoid indexing `_id`, use a
Bloom filter for ingest-time duplicate detection, and resolve ID-dependent
operations through timestamps and dimensions. New TSDB indices disable
sequence numbers. Synthetic-ID indices support nested documents and
`best_compression`.

### Direct OTLP metrics ingest

In 9.2.0, the technical-preview `/_otlp/v1/metrics` endpoint accepts OTLP
metrics directly.

## Indexing safeguards

In 9.1.0, Elasticsearch adds `IndexingPressureMonitor`, accounts for memory
consumed by document expansion, and introduces a maximum document-size limit.
Thread-pool telemetry adds utilization and queue-latency metrics.
