# Breaking changes

Use these checks when upgrading clients, plugins, mappings, or cluster
configuration. The changes are grouped by affected workflow.

## Query and response behavior

### ES|QL partial results are the default

ES|QL responses may be partial by default. Callers must inspect `is_partial`.
Require complete results per request with `allow_partial_results=false`, or
cluster-wide with `esql.query.allow_partial_results: false`.

### EQL allows partial search results by default

EQL defaults `allow_partial_search_results` to `true`. Callers that require all
shards to succeed must opt out.

### `skip_unavailable` suppresses remote runtime errors

With `skip_unavailable: true`, every runtime error from a remote cluster,
including a missing index, is non-fatal; Elasticsearch reports the cluster as
skipped or partial instead.

### ES|QL index-pattern quoting is all-or-nothing

Parentheses are rejected in unquoted index patterns. Remote-cluster and index
components cannot be quoted separately. Quote the entire pattern or none of
it: `FROM "remote:index"` and `FROM remote:index` are valid, but
`FROM remote:"index"` is invalid.

### Date histograms reject booleans

The `date_histogram` aggregation no longer accepts the formerly supported
boolean form.

### `random_score` has a new implicit field

When no field is supplied, `random_score` uses `_seq_no`.

### Timeout and byte-size handling changed

Elasticsearch timeouts return HTTP 429 rather than a 5xx response. Byte-size
values are limited to two decimal places.

## Source, mapping, and analysis behavior

### New indices exclude vectors from `_source`

`exclude_source_vectors` is enabled by default for new indices. Use supported
vector retrieval controls when an application needs the vector values.

### LogsDB and TSDB text fields omit norms

Text fields no longer have norms enabled when the index mode is LogsDB or TSDB.

### Normalized keywords use native synthetic source

Normalized `keyword` fields use a native synthetic-source implementation.

### Mapping definitions are more restrictive

Metadata field definitions no longer support `type`, `fields`, `copy_to`, or
`boost`. The `_source` meta-field's `mode` attribute is a no-op.

### Analyzer output changed

Snowball stemmers and the Nori Korean dictionary were updated. `german2` is an
alias for the `german` Snowball stemmer, and the `persian` analyzer stems by
default.

## Ingest, logs, and telemetry

### Invalid ingest simulation returns HTTP 400

The simulate ingest API returns `400 Bad Request` when the request contains an
invalid processor.

### Ingest processor options were removed

The `user_agent` processor no longer accepts `ecs`. The GeoIP processor's
ignored fallback option was removed.

### LogsDB is conditionally enabled

LogsDB is enabled by default for data streams matching `logs-*-*` when its
enabling conditions are met.

### OTLP defaults to exponential histograms

The OTLP endpoint maps histograms as `exponential_histogram` fields by default.

## Lifecycle and cluster behavior

### ILM downsampling no longer force-merges by default

Starting in 9.4.0, the ILM downsampling action leaves the downsampled index
unmerged by default. Policies that need the old behavior must add a force-merge
action or set the downsample action's `force_merge_index` parameter to `true`.

### Allocation interfaces changed

The `cluster.routing.allocation.disk.watermark.enable_for_single_data_node`
setting was removed. `/_cluster/reroute` responses no longer include cluster
state.

### Fleet search is local-only

`_fleet/_fleet_search` and `_fleet/_fleet_msearch` no longer support
cross-cluster operation.

## Security, credentials, and plugins

### Inference secrets cannot be overridden

In 9.3.8 and 9.4.4, Inference API requests can no longer override
`secret_parameters`.

### LDAP and Active Directory require complete bind credentials

Configuring a bind DN without a corresponding bind password prevents the node
from starting.

### Connector APIs require connector privileges

Connector APIs are restricted to `manage_connector` and `monitor_connector`.

### `discovery-ec2` uses AWS SDK v2

The plugin requires IMDSv2, ignores `discovery.ec2.protocol`, and no longer
supports `aws.secretKey` or
`com.amazonaws.sdk.ec2MetadataServiceEndpointOverride`. Put `http://` directly
in `discovery.ec2.endpoint` when needed. Configure both
`discovery.ec2.access_key` and `discovery.ec2.secret_key`, or neither.

### TLS defaults remove legacy protocols and ciphers

JDK 24 installations no longer support `TLS_RSA` ciphers, and TLSv1.1 is no
longer in the default protocol list.

## Removed APIs, settings, and platforms

### Highlighting and index APIs

Highlighting no longer accepts `force_source`. Alias APIs no longer accept
`local`. Frozen indices cannot be read, and the unfreeze REST endpoint was
removed.

### Settings and deprecation-log field

The `client.type`, `tracing.apm.*`, and
`xpack.searchable.snapshot.allocate_on_rolling_restart` settings were removed.
The deprecation-log keyword is `elasticsearch.deprecation` rather than
`deprecation.elasticsearch`.

### Platform and legacy APIs

Machine learning is disabled on macOS x86_64. The `data_frame_transforms`
roles, technical-preview `_knn_search` API, and `types` field in Watcher
searches were removed.
