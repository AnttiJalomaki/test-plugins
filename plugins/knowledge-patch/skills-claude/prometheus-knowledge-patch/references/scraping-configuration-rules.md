# Scraping, Configuration, and Rules

## Scrape protocol negotiation

### Strict Content-Type handling

A target that omits an accepted `Content-Type`, or sends an unparsable or
unknown one, fails scraping (`3.0-migration`). Fix the endpoint to advertise
protobuf-delimited, Prometheus text 0.0.4 or 1.0.0, or OpenMetrics 0.0.1 or
1.0.0. When the endpoint cannot be changed, configure
`fallback_scrape_protocol` for that target.

Classic protobuf scraping no longer requires a metric's unit to be included in
its metric name; a producer may provide the unit separately (since 3.9.0).

The OpenMetrics text parser accepts quoted exemplar keys (since 3.1.0).

### Protocol preference and escaping

Scrape configuration can select the escaping scheme requested during content
negotiation (since 3.4.0).

Unless `scrape_protocols` is explicit, enabling
`created-timestamp-zero-ingestion` changes the global default preference to
`PrometheusProto`, `OpenMetricsText1.0.0`, `OpenMetricsText0.0.1`, then
`PrometheusText0.0.4` (`feature-flags`).

Scrape requests include a `traceparent` HTTP header (since 3.6.0), allowing
scrapes to participate in propagated trace context.

## Names, labels, and relabeling

### UTF-8 names

Metric and label names accept UTF-8 (`3.0-migration`). To preserve the previous
validation, set this globally or on a scrape job:

```yaml
global:
  metric_name_validation_scheme: legacy
```

Valid values are `utf8` and `legacy`. Exposed names may change after upgrading
if the producer previously relied on normalization.

Replace relabel actions accept UTF-8 in `targetLabel` (since 3.2.0). `$<chars>`
and `${<chars>}` expand, and `LabelMap` applies the same behavior to
`replacement`. External labels expand `$var` and `${var}`; undefined values
become empty and `$$` escapes a dollar (`3.0-migration`). Scrape target labels
remain configured rather than receiving scheme-derived ports.

### Histogram and summary labels

Classic histogram `le` and summary `quantile` values normalize to float-like
strings across protocols (`3.0-migration`). For example, an exposed `"1"` is
stored as `"1.0"`:

```promql
my_classic_hist_bucket{le="1.0"}
```

Update exact label matchers in alerts and dashboards. Queries spanning the
transition can still return surprising results.

### Per-target histogram controls

Target relabeling can set these internal labels to override scrape settings for
one target (since 3.13.0):

- `__convert_classic_histograms_to_nhcb__`
- `__always_scrape_classic_histograms__`
- `__scrape_native_histograms__`

```yaml
relabel_configs:
  - target_label: __scrape_native_histograms__
    replacement: "true"
```

## Histogram scrape configuration

Use `scrape_native_histograms: true` globally or on a job to ingest native
histograms. The `native-histograms` gate becomes a no-op after stabilization,
but this setting remains required (`3.0-migration`).

When a target exposes a native histogram and its classic counterpart, use
`always_scrape_classic_histograms`, not the old
`scrape_classic_histograms` key. It works at job level (`3.0-migration`) and
globally from 3.5.0:

```yaml
global:
  always_scrape_classic_histograms: true
```

`convert_classic_histograms_to_nhcb` is global as well as job-local (since
3.4.0). Reloads honor it and `always_scrape_classic_histograms` rather than
silently ignoring either setting (since 3.1.0).

Created-timestamp processing supports native histograms, and TSDB accepts their
out-of-order samples (since 3.0.0). Created-timestamp zero ingestion no longer
creates extra `_created` series (since 3.0.0). Conversion from classic
histograms to NHCB can be combined with zero-timestamp ingestion (since 3.8.0).

When native-histogram ingestion is disabled, scraping skips those series (since
3.3.0).

## Extra scrape metrics

`--enable-feature=extra-scrape-metrics` is deprecated. Enable the replacement
globally or per scrape configuration (`feature-flags`):

```yaml
global:
  extra_scrape_metrics: true
```

It stores `scrape_timeout_seconds`, `scrape_sample_limit`, and
`scrape_body_size_bytes`. A zero sample limit means unlimited. Body size is `-1`
when the size limit caused failure and `0` for other scrape failures.

## Configuration reload

`auto-reload-config` first appeared as an experimental feature in 3.0.0. By
3.4.0 it watches referenced rule and scrape configuration files, not only the
main file. It is stable from 3.12.0.

## Rules and evaluation

### Parsing and rule files

Rule names may contain UTF-8 except `{` and `}`, which common-mistake checks
reject (since 3.2.0). Rule YAML may use anchors and aliases (since 3.3.0).
Parse failures are detected earlier during startup (since 3.5.0).

### Concurrency

When dependency analysis is uncertain, rules evaluate serially instead of being
run concurrently (since 3.1.0). With `concurrent-rule-eval`, dependency-free
rules inside one group can run concurrently (`feature-flags`). Limit this extra
query load with `--rules.max-concurrent-evals`; its default is 4.

### Alert state and relabeling

Alert relabeling participates in the decision to drop an alert (since 3.3.0).
Mutations in one `alertmanager_config.alert_relabel_configs` block do not flow
into later Alertmanager configuration blocks (since 3.7.0).

Increasing an alert's `FOR` period no longer resets its state incorrectly to
pending (since 3.11.0). Restoration also works when rule labels contain Go
template expressions.

An alerting rule that has never evaluated reports `unknown` (since 3.8.0).

### Notifications

Set the largest notification batch with
`--alertmanager.notification-batch-size` (since 3.4.0). Each Alertmanager uses
an independent send loop from 3.10.0.

## Templates

Templates provide `toDuration()` and `now()` (since 3.6.0). Use
`urlQueryEscape` when interpolating an alert value into a URL query string
(since 3.8.0):

```text
{{ urlQueryEscape $labels.instance }}
```
