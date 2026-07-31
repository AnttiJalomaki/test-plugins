# Outputs and Observability

## OpenTelemetry output

### Use the stable output name (since 1.4.0)

Select the stable OpenTelemetry output as `opentelemetry`. The old
`experimental-opentelemetry` name remains accepted but is deprecated.
`K6_OTEL_EXPORTER_TYPE` is also deprecated; use
`K6_OTEL_EXPORTER_PROTOCOL`.

```sh
k6 run --out opentelemetry script.js
```

### Configure HTTP Basic Auth (since 2.1.0)

The OpenTelemetry HTTP exporter accepts Basic Auth credentials through
`K6_OTEL_HTTP_EXPORTER_USERNAME` and `K6_OTEL_HTTP_EXPORTER_PASSWORD`, or the
`username` and `password` output-config keys.

```sh
K6_OTEL_HTTP_EXPORTER_USERNAME=user \
K6_OTEL_HTTP_EXPORTER_PASSWORD=secret \
k6 run --out opentelemetry script.js
```

## TLS for metric outputs

### Account for the TLS 1.3 default (since 1.2.0)

The experimental OpenTelemetry and Prometheus outputs changed their default to
TLS 1.3. Because these outputs were experimental, this was a breaking change
in a minor release.

### Configure Prometheus remote-write minimum TLS (since 1.6.0)

The experimental Prometheus remote-write output accepts
`K6_PROMETHEUS_RW_TLS_MIN_VERSION`. Its default remains TLS 1.3.

```sh
K6_PROMETHEUS_RW_TLS_MIN_VERSION=1.3 \
k6 run script.js -o experimental-prometheus-rw
```

## Rate metric representation

### Consume the labeled counter shape (since 1.3.0)

An exported Rate is represented by one counter with a label whose values are
`zero` and `nonzero`. Downstream queries and integrations must consume that
labeled shape.

### Remove the fallback switch in v2 (since 2.0.0)

`K6_OTEL_SINGLE_COUNTER_FOR_RATE` was removed. Delete it from environments and
deployment configuration; the single-counter migration cannot be postponed.

## Machine-readable summaries

### Opt in to the structured shape (since 1.5.0)

A structured summary representation is available to both `--summary-export`
and `handleSummary()`. Opt in with `--new-machine-readable-summary` or
`K6_NEW_MACHINE_READABLE_SUMMARY`. It was planned to become the default in v2.

```sh
k6 run script.js \
  --new-machine-readable-summary \
  --summary-export=summary.json
```

## Cloud output correctness

### Read Gauge extrema from the corrected fields (since 1.8.0)

Cloud output v2 reports Gauge `min` and `max` in their correct fields. Cloud
test-result queries no longer return the peak as the floor or the floor as the
peak.

## Native histograms

### Enable experimental Trend storage (since 2.1.0)

The `native-histograms` feature makes Trend metrics use experimental native
histograms. Enable it through feature flags and remember that enabled features
are included in metric tags and preserved in archives and Cloud workers.

```sh
k6 run --features native-histograms script.js
```

Inspect the feature and its lifecycle with `k6 features` or
`k6 features --json`.

## Web dashboard

### Use the built-in output (since 2.0.0)

The web dashboard is built into the k6 binary. A separate xk6-dashboard
extension is no longer required.

```sh
k6 run --out=web-dashboard script.js
```
