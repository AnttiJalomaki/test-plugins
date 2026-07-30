# HTTP APIs, promtool, and observability

Use this reference when maintaining API clients, command pipelines, rule tests,
feature detection, dashboards, or self-monitoring.

## HTTP API response changes

### Query and rules APIs

The `/query` and `/query_range` endpoints accept a `limit` parameter from
3.2.0:

```text
/query?limit=100
/query_range?limit=100
```

The rules API can paginate rule groups from 3.1.0. Its `groupNextToken` field is
present even when empty, so clients must accept it whether or not another page
exists.

### Status APIs

The `/status` response includes `Node` and `ServerTime` from 3.2.0.

The loaded-block metadata endpoint `/v1/status/tsdb/blocks` is available from
3.6.0.

From 3.9.0, the TSDB status endpoint returns at most 10,000 sets of statistics.
Clients must not assume the response is an unlimited inventory.

`/api/v1/status/self_metrics` returns the current JSON state of the server's
own metrics from 3.12.0.

### Feature and schema discovery

Use `/api/v1/features` from 3.9.0 to discover server capabilities instead of
inferring them from the version.

Prometheus serves an OpenAPI 3.2 HTTP API document at
`/api/v1/openapi.yaml` from 3.10.0.

The experimental search API added in 3.13.0 searches metric names, label names,
and label values.

With `search-api`, `--web.search.max-limit` limits each search request
(`feature-flags`). Its default is `10000`; a request above it gets HTTP 400.
The ordinary response default is `100` and is clamped to a lower operator cap.
A cap of `0` permits unbounded requests and is unsafe on untrusted endpoints.

## Query statistics

From 3.13.0, query statistics expose `samplesRead`. They also expose
`samplesReadPerStep` when `stats=all` and `promql-per-step-stats` are enabled.
These fields measure storage I/O.

Do not confuse them with `totalQueryableSamples`, which counts samples loaded
into the evaluator and can count one reused sample in several range-vector
windows. Use `prometheus_engine_query_samples_read_total` for the engine-wide
storage-read counter.

The same release corrects query accounting:

- range subqueries stop at the parent's last actual step when query end is not
  step-aligned, avoiding inflated `peakSamples`, `query.max-samples`, and
  storage reads;
- an `@`-modified range beneath an at-modifier-unsafe function counts
  `totalQueryableSamples` correctly after the first step.

## Debug and operational endpoints

Wall-time profiling is available at `/debug/pprof/fgprof` from 3.10.0.

The Status UI can delete time series and clean tombstones from 3.12.0.

Prometheus 3.13.0 serves embedded third-party npm licenses at
`/assets/third-party-licenses.txt`; images and tarballs no longer include the
former `npm_licenses.tar.bz2` archive.

## promtool configuration and linting

Promtool adds the `too-long-scrape-interval` lint from 3.2.0.

Use `--ignore-unknown-fields` from 3.2.0 when fields unknown to the installed
promtool should not fail validation.

Promtool accepts PromQL feature flags from 3.4.0 so offline validation can use
feature-gated syntax. From 3.10.0 it understands
`promql-duration-expr` and `promql-extended-range-selectors`.

Relative paths inside the file supplied to `--http.config.file` resolve from
that file's directory from 3.13.0, not from its parent directory. Adjust paths
that depended on the previous extra parent traversal.

## promtool data commands

`promtool tsdb create-blocks-from openmetrics` accepts OpenMetrics input from a
pipe from 3.3.0.

`promtool tsdb dump` supports a labels-only JSON format from 3.9.0:

```text
promtool tsdb dump --format seriesjson
```

`promtool push metrics` can send Remote Write 2.0 messages from 3.8.0 by
selecting `--protobuf_message`.

`promtool query instant` accepts `--header` from 3.12.0, matching the range
query command.

## Rule and PromQL tests

Rule unit tests accept fuzzy float64 comparison from 3.5.0:

```yaml
fuzzy_compare: true
```

They accept `start_timestamp` from 3.9.0 for time-sensitive tests. PromQL test
`load` data accepts `@st` annotations for individual sample start timestamps
from 3.12.0.

## Output streams

Promtool sends diagnostic/debug output to stderr from 3.11.0, preserving stdout
for the primary result. Pipelines that merged or parsed both streams need to
separate them.

## Self-monitoring metrics

### Rules and Go runtime

Prometheus exports these metrics from 3.1.0:

- `rule_group_last_rule_duration_sum_seconds`, the total duration of rule-group
  evaluation;
- `go_sync_mutex_wait_total_seconds_total`, Go mutex wait time.

### TSDB and histograms

Use `prometheus_tsdb_wal_replay_unknown_refs_total` and
`prometheus_tsdb_wbl_replay_unknown_refs_total` from 3.4.0 for unknown series
references during WAL/WBL replay.

Use `prometheus_tsdb_head_stale_series` from 3.6.0 for stale series in Head.

Use `prometheus_tsdb_sample_ooo_delta` from 3.9.0 for the out-of-order distance
of accepted and rejected samples.

PromQL, rule, service-discovery, and scrape instrumentation include native
histograms alongside summaries from 3.9.0. Notification latency adds
`prometheus_notifications_latency_histogram_seconds`.

### Service discovery

Most `prometheus_sd_refresh` metrics include a `config` label with the job name
from 3.9.0.

Use `prometheus_sd_last_update_timestamp_seconds` from 3.11.0 for the last
update delivered to discovery consumers.

Per-job `prometheus_sd_refresh*` and
`prometheus_sd_discovered_targets` series disappear when their scrape job is
removed from 3.12.0.

### Remote write and notifications

Remote-write metric replacements are listed in
[Remote read, remote write, and authentication](remote-io-and-auth.md).

From 3.10.0, notification dropped, queue capacity, and queue length metrics have
an `alertmanager` label. `prometheus_notifications_errors_total` counts
affected alerts rather than failed batches from 3.1.0.

## Mixin configuration

The Prometheus mixin's `cluster` label is configurable through `clusterLabel`
from 3.3.0.
