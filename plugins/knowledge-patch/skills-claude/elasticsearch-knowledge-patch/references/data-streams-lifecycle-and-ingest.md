# Data Streams, Lifecycle, Reindexing, and Ingest

Use this reference when building index templates, data-stream failure
handling, ILM or data-stream lifecycle policies, migration tooling, transforms,
or ingest pipelines.

## Contents

- [Failure stores](#failure-stores)
- [Streams and stream types](#streams-and-stream-types)
- [ILM and data-stream lifecycle](#ilm-and-data-stream-lifecycle)
- [Migration and reindexing](#data-stream-migration-and-reindexing)
- [Indexing and ingest](#indexing-apis-and-safeguards)
- [Templates and transforms](#templates-pipelines-and-transforms)

## Failure stores

### Enable and query a failure store

Data-stream failure stores became available in 8.19.0. They can capture
documents rejected by an ingest pipeline or mapping conflict.

Enable a store on an existing stream:

```http
PUT _data_stream/logs-test-apache/_options
{
  "failure_store": {
    "enabled": true
  }
}
```

For a new stream, set
`template.data_stream_options.failure_store.enabled` in a component or index
template.

Query failure-store backing indices through the `::failures` selector:

```http
GET logs-*::failures/_search
```

```esql
FROM my_data_stream*::failures
```

### Defaults and lifecycle

- Data streams gain a dedicated failure-store lifecycle plus default failure
  retention in 9.1.0.
- New `logs-*-*`, OTel, and APM streams enable failure stores by default in
  9.2.0. Existing streams still need explicit enablement.
- An invalid document routed to a `logs-*-*` failure store in 9.2.0 returns
  HTTP 201 with `"failure_store": "used"` rather than HTTP 400.
- The Get Data Stream API reports each stream's index mode in 9.1.0.
- Failure-store indices can participate in cross-cluster search in 9.4.0.

### Recover failed documents

The 9.2.0 `recover_failure_document` ingest processor supports remediation of
failure-store documents. Preserve failure metadata while correcting the
document and route successful output back to the intended stream.

## Streams and stream types

- Streams can be enabled only when no conflicting indices exist (9.2.0).
  After streams are enabled, indexing is restricted to child streams.
- Data streams add `logs.otel` and `logs.ecs` stream types in 9.4.0.
- The older `logs` stream type is deprecated in 9.4.0.
- LogsDB can route on sort fields and take index-sort configuration through
  settings as of 9.0.0.
- LogsDB becomes conditionally enabled by default for `logs-*-*` data streams.

## ILM and data-stream lifecycle

### Per-index controls

Set `index.lifecycle.skip` to keep ILM from processing one index (9.1.0):

```http
PUT my-index/_settings
{
  "index.lifecycle.skip": true
}
```

The remove-block API removes a named index block:

```http
DELETE /my-index/_block/write
```

### Searchable snapshots and followers

- ILM `searchable_snapshot` accepts `replicate_for` in 9.0.0.
- Before downsampling, ILM injects an unfollow action when necessary
  (9.1.0).
- A follower waits for the leader's time-series end time before unfollowing
  (9.1.0).

### Downsampling

- Data-stream lifecycle and ILM can select among downsampling methods in
  9.3.0; the Downsample API also adds another sampling method.
- Force merge moved out of the downsample request into the ILM action in
  9.3.0, where it can be disabled.
- Starting in 9.4.0, ILM downsampling defaults to leaving the downsampled
  index unmerged. Add a force-merge action or set
  `force_merge_index: true` on the downsample action when a merge is required.
- The 9.4.0 default `aggregate` method stores a counter's first value and
  auxiliary documents for resets, preserving resets during later rate
  calculations. `last_value` keeps its storage-oriented behavior.
- In 9.1.0 and 9.1.1, an optimized merge path can fail after shrinking TSDB or
  LogsDB indices. Upgrade to 9.1.2 or use the temporary workaround documented
  in the compatibility reference.

### Rollover and lifecycle responses

- `max_size` is a deprecated ILM rollover condition as of 9.3.0.
- ILM explain responses include `age_in_millis` in 9.2.0.
- The read-only lifecycle action sets `indexing_complete` to `true` in 9.2.0.

## Data-stream migration and reindexing

### Migration controls

Elasticsearch 9.0.0 adds REST and action support for data-stream migration
reindexing:

- Create an index from a source index.
- Query or cancel migration reindex tasks.
- Throttle with `requests_per_second`.
- `_create_from` removes index blocks by default; control this with
  `remove_index_block`.
- Closed source indices are ignored.
- Deprecated destination settings are filtered.

### Remote reindex

- Remote reindex accepts a convenience API-key parameter in 9.3.0.
- Remote reindex gains a blocklist setting in 9.4.0.
- Reindex always includes vectors despite transparent vector exclusion from
  `_source` (9.2.0).
- Reindexing metrics report seconds instead of milliseconds as of 9.0.0.

## Indexing APIs and safeguards

- Create, index, update, and bulk APIs accept `include_source_on_error` in
  9.0.0. It controls source inclusion in document parse-error responses and
  defaults to `true`.
- An `IndexingPressureMonitor`, memory accounting for document expansion, and
  a maximum document-size limit arrive in 9.1.0.
- The `replica_unassigned_buffer_time` default increases from three seconds to
  five seconds in 9.0.0.

## Ingest processors and simulation

### Pipeline processors

- The reroute processor can set `type` in 9.0.0.
- The append processor gains `copy_from` and an option to ignore empty values
  in 9.2.0.
- Conditional processors can use the Fields API in 9.2.0.
- The `cef` processor in 9.3.0 parses Common Event Format into device vendor,
  product, version, signature ID, name, severity, and extension fields.
- The Grok processor adds `validate_only` in 9.4.0 to validate patterns without
  extracting fields.
- The `user_agent` processor no longer accepts `ecs`, and the GeoIP processor's
  ignored fallback option is removed.

### Simulate and structure discovery

- Simulate ingest responses include ignored fields as of 9.0.0.
- The simulate API accepts `merge_type` and returns the effective mapping in
  9.2.0.
- Invalid processors in simulate requests now produce HTTP 400.
- Text-structure endpoints accept nested NDJSON records in 9.4.0.

## Templates, pipelines, and transforms

- Index templates, component templates, and ingest pipelines expose created
  and modified timestamps in 9.2.0.
- Transforms gain upgrade mode, automatic migration of
  `max_page_search_size`, and `extended_stats` in 9.0.0.
- If `delete_dest_index=true` and a transform destination is an alias,
  deleting the transform deletes the alias's write index (9.0.0).
- Transforms add a preview-index request in 9.3.0.
- Stop-datafeed accepts `close_job` in 9.3.0.

## Mustache scripts

Use `mustache.max_output_size_bytes` to cap Mustache script-result length
(9.0.0), especially for templates fed by user-controlled or unexpectedly
large data.
