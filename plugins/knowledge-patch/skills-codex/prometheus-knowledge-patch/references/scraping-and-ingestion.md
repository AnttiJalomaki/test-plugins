# Scraping and ingestion

Use this reference for scrape protocol negotiation, metric-name handling,
relabeling, scrape-time histogram conversion, and target-level overrides.

## Protocol negotiation and parsing

### Content-Type is strict

Starting with the `3.0-migration`, a scrape fails if a target omits an accepted
`Content-Type` or sends an unparsable or unknown value. Prometheus no longer
silently falls back to its text format. Correct the endpoint to advertise one
of the supported formats:

- protobuf-delimited
- Prometheus text 0.0.4 or 1.0.0
- OpenMetrics 0.0.1 or 1.0.0

If the endpoint cannot be fixed, configure `fallback_scrape_protocol` for that
target.

### Requested escaping and unit rules

Scrape configuration can choose the escaping scheme requested during content
negotiation from 3.4.0.

Scraping classic protobuf no longer requires the metric unit to be embedded in
the metric name from 3.9.0. Producers can supply unit metadata independently.

The OpenMetrics parser accepts quoted exemplar keys from 3.1.0.

### Zero-injection protocol preference

Unless `scrape_protocols` is explicit, enabling
`created-timestamp-zero-ingestion` changes the global preference order to the
following (`feature-flags`):

1. `PrometheusProto`
2. `OpenMetricsText1.0.0`
3. `OpenMetricsText0.0.1`
4. `PrometheusText0.0.4`

This means protobuf is negotiated first.

## Names, labels, and normalization

### UTF-8 metric and label names

The `3.0-migration` accepts UTF-8 metric and label names. Names rejected by
older versions can be ingested, and exposed names can change after an upgrade.
Preserve the old validator globally with:

```yaml
global:
  metric_name_validation_scheme: legacy
```

The same `metric_name_validation_scheme` can be set to `utf8` or `legacy` per
scrape job.

From 3.2.0, replace relabel actions accept UTF-8 in `targetLabel`. `$<chars>`
and `${<chars>}` are expanded. `LabelMap` applies the same expansion rules to
its `replacement` field.

### Histogram and summary label values

The `3.0-migration` normalizes classic histogram `le` and summary `quantile`
values into float-like strings regardless of scrape protocol. For example,
`"1"` is stored as `"1.0"`:

```promql
my_classic_hist_bucket{le="1.0"}
```

Update rules and dashboards that match integer strings. Queries spanning the
normalization transition can still produce unexpected results.

## Native and classic histogram controls

### Enabling ingestion

Created-Timestamp processing supports native histograms and the TSDB can ingest
out-of-order native-histogram samples from 3.0.0.

Native histograms are stable from 3.9.0, so `native-histograms` becomes a
no-op. Scraping them still requires the configuration option introduced in
3.8.0:

```yaml
global:
  scrape_native_histograms: true
```

When native-histogram ingestion is disabled, scraping skips native-histogram
series from 3.3.0.

### Retaining classic histograms

During the `3.0-migration`, the job-level `scrape_classic_histograms` setting
is renamed to `always_scrape_classic_histograms`:

```yaml
scrape_configs:
  - job_name: mixed-histograms
    scrape_native_histograms: true
    always_scrape_classic_histograms: true
```

`always_scrape_classic_histograms` is also accepted globally from 3.5.0:

```yaml
global:
  always_scrape_classic_histograms: true
```

Reloaded configuration correctly applies this setting from 3.1.0.

### Classic-to-custom-bucket conversion

`convert_classic_histograms_to_nhcb` can be set globally from 3.4.0 instead of
being repeated in every scrape job:

```yaml
global:
  convert_classic_histograms_to_nhcb: true
```

Reloaded configuration honors it from 3.1.0. Classic-histogram-to-NHCB
conversion can be combined with created-timestamp zero ingestion from 3.8.0.

### Out-of-order ingestion

`--enable-feature=ooo-native-histograms` is a no-op from 3.4.0. At that point,
out-of-order native histograms are enabled when `out_of_order_time_window` is
greater than zero and `--enable-feature=native-histograms` is present. From
3.9.0 the latter feature flag is itself a no-op because native histograms are
stable.

## Per-target and per-job controls

Target relabeling can set these special labels from 3.13.0:

- `__convert_classic_histograms_to_nhcb__`
- `__always_scrape_classic_histograms__`
- `__scrape_native_histograms__`

They override the corresponding scrape settings for one target:

```yaml
relabel_configs:
  - target_label: __scrape_native_histograms__
    replacement: "true"
```

Dropped targets returned by `/api/v1/targets` include their scrape pool name
from 3.3.0, making per-pool diagnosis possible even after relabeling drops a
target.

## Created timestamps and extra scrape metrics

With `created-timestamp-zero-ingestion` enabled, created timestamps no longer
produce separate `_created` time series from 3.0.0.

Scrape requests include a `traceparent` header from 3.6.0 so scrape work can
participate in propagated tracing.

`--enable-feature=extra-scrape-metrics` is deprecated. Configure its
replacement globally or per scrape configuration (`feature-flags`):

```yaml
global:
  extra_scrape_metrics: true
```

This stores:

- `scrape_timeout_seconds`
- `scrape_sample_limit`
- `scrape_body_size_bytes`

A sample limit of zero means unlimited. Body size is `-1` when a size-limit
failure occurred and `0` for other scrape failures.
