# Remote Storage and OTLP

## Remote read

TSDB-compatible storage must return only results matching every requested
selector (`3.0-migration`). Third-party `remote_read` implementations that
return extra series invoke undefined behavior even though Prometheus does not
explicitly enforce the contract.

Histograms received through remote read are validated (since 3.9.0); invalid
histogram data is no longer accepted silently. On the 3.11 line, use at least
3.11.3 to enforce declared-length limits on Snappy requests (`3.11.0`).

## Remote write transport and queues

### HTTP/2 and DNS

Remote-write clients default `http_config.enable_http2` to `false`
(`3.0-migration`) so parallel queues can open multiple sockets. Set it to
`true` explicitly to keep the older HTTP/2 behavior.

Clients can opt into a DNS resolver that selects a random returned IP (since
3.1.0), avoiding a fixed choice for multi-address endpoints.

### Queue validation and timing

Configuration loading validates `remote_write.queue_config` fields (since
3.12.0), rejecting invalid values before a panic or silent runtime
misconfiguration.

`prometheus_remote_storage_sent_batch_duration_seconds` is measured after the
request is sent, not before it (since 3.11.0).

### Metric migrations

Replace deprecated remote-write metrics from 3.7.0:

- `prometheus_remote_storage_samples_in_total` with
  `prometheus_wal_watcher_records_read_total{type="samples"}` plus
  `prometheus_remote_storage_samples_dropped_total`.
- `prometheus_remote_storage_histograms_in_total` with
  `prometheus_wal_watcher_records_read_total{type=~".*histogram_samples"}` plus
  `prometheus_remote_storage_histograms_dropped_total`.
- `prometheus_remote_storage_exemplars_in_total` with
  `prometheus_wal_watcher_records_read_total{type="exemplars"}` plus
  `prometheus_remote_storage_exemplars_dropped_total`.
- `prometheus_remote_storage_highest_timestamp_in_seconds` with
  `prometheus_remote_storage_queue_highest_timestamp_seconds`; the replacement
  accounts for relabeling and is more accurate.

## Remote Write 2

### Start-timestamp schema

The receiver follows the 2.0-rc.4 specification, which renames created
timestamp to start timestamp (since 3.8.0). Update senders and receiver
integrations to the revised terminology and schema.

With `st-storage`, ingested scrape and OTLP start timestamps can be exposed over
Remote Write 2 (`3.11.0`). Encoding and WAL replay prerequisites are in the
storage reference.

### Rejections and protocol limits

Too-old Remote Write 2 samples in histogram paths return HTTP 400 rather than
500 (since 3.11.0), preventing endless retries of non-retryable data.

Federation transports custom-bucket native histograms, but Remote Write 1.0
cannot. Prometheus blocks that send and logs a warning (since 3.7.0).

### Metadata labels

With `type-and-unit-labels`, outgoing Remote Write 2 series contain type and
unit labels (since 3.7.0). If a WAL metadata record conflicts with `__type__` or
`__unit__` already carried by the series, the reserved metadata value wins
(`feature-flags`).

### Promtool sender

`promtool push metrics --protobuf_message ...` can select and send a Remote
Write 2.0 message (since 3.8.0).

## Azure authentication for remote write

An empty `client_id` under managed identity selects system-assigned identity
(since 3.5.0):

```yaml
remote_write:
  - url: https://example.invalid/api/v1/write
    azuread:
      managed_identity:
        client_id: ""
```

Azure Workload Identity is supported from 3.7.0. AzureAD can request a custom
scope from 3.9.0. From 3.13.0, remote write can use a certificate to ingest into
an Azure Monitor Workspace.

Use at least 3.11.3 on the 3.11 line to prevent `client_secret` disclosure
through `/-/config` (`3.11.0`).

## OTLP translation

### Name strategies

`NoUTF8EscapingWithSuffixes` preserves UTF-8 while retaining suffix translation
(since 3.0.0).

From 3.4.0, translation can leave metric names and attributes in their original
OTLP forms. `UnderscoreEscapingWithoutSuffixes` escapes names with underscores
but adds no translated suffixes (since 3.6.0):

```yaml
otlp:
  translation_strategy: UnderscoreEscapingWithoutSuffixes
```

From 3.7.1 (`3.7.0`), translating an attribute name that begins with one
underscore prefixes the Prometheus label with `key_`; multiple consecutive
leading underscores are preserved. This restores behavior changed in 3.7.0.

The translated metric name owns `__name__`; conversion filters an incoming
OTLP `__name__` attribute so it cannot create a duplicate label (since 3.10.0).

### Resource, scope, identity, and metadata

Translation preserves identifying resource attributes in `target_info`; the
receiver converts metric metadata and accepts colons in non-standard unit
strings (since 3.1.0).

Enable scope labels from 3.6.0:

```yaml
otlp:
  promote_scope_metadata: true
```

Promote all resource attributes with selected exclusions from 3.5.0:

```yaml
otlp:
  promote_all_resource_attributes: true
  ignore_resource_attributes:
    - service.instance.id
```

Prometheus generates `target_info` samples from the earliest through latest
sample for each OTLP resource (since 3.6.0), keeping metadata present throughout
the resource interval. Duplicate samples with the same series and timestamp are
removed (since 3.8.0).

OTLP receiver defaults apply even when no `otlp:` block appears (since 3.4.0).

### Type and unit labels

With `type-and-unit-labels`, OTLP metrics receive `__type__` and `__unit__`
(since 3.6.0). Incoming user values for those names are overwritten by ingestion
metadata; PromQL drops them in the same operations that drop `__name__`
(`feature-flags`).

## OTLP temporality

### Delta-to-cumulative conversion

Enable `otlp-deltatocumulative` to convert delta metrics rather than dropping
them (since 3.2.0):

```text
--enable-feature=otlp-deltatocumulative
```

Conversion keeps per-series state in memory. Restart loses that state and
causes a counter reset; inactive state is periodically removed according to
`max_stale`.

### Raw delta ingestion

Primitive ingestion of OTLP deltas without cumulative conversion appeared in
3.4.0. The `otlp-native-delta-ingestion` gate stores raw delta values and is
mutually exclusive with `otlp-deltatocumulative` (`feature-flags`). It currently
ignores `StartTimeUnixNano` and records unknown metric metadata.

Counter functions are wrong for raw deltas. Sum them directly:

```promql
sum_over_time(delta_metric[5m])
sum_over_time(delta_metric[5m]) / 5m
```

Align the range to collection cadence. Same-timestamp deltas are not summed;
federation can miscollect them when scrape and ingestion cadences differ; mixed
delta and cumulative data needs an explicit distinguishing label.

### Sum correctness

A path that could silently lose OTLP sum metrics is fixed in 3.10.0. Upgrade
systems ingesting sums rather than depending on the affected behavior.

## OTLP histograms, exemplars, and start times

An opt-in gate can translate explicit-bucket OTLP histograms into custom-bucket
native histograms (since 3.4.0).

OTLP metric start times can become created-time zero samples when
`created-timestamp-zero-ingestion` is enabled (since 3.7.0). With `st-storage`,
they can instead be stored and transported as start timestamps (`3.11.0`).

OTLP exemplars are placed into the correct histogram components from 3.11.0.

## OTLP endpoint and tracing safety

The OTLP write endpoint limits decompressed size for gzip request bodies (since
3.12.0). Prometheus also starts successfully when tracing uses insecure OTLP
over HTTP (since 3.12.0).
