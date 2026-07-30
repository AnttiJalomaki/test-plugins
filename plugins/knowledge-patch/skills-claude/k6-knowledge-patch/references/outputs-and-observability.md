# Outputs and Observability

## End-of-test summary contracts

The redesigned human summary in `1.0.0-rc1` defaults to `compact`; `full`
includes detailed metrics and per-group and per-scenario results, while
`legacy` reproduced the older display. That release did not change the
`handleSummary` input or `--summary-export` data structure.

Threshold values became visible in the end-of-test output in `1.0.0-rc2` even
when they are absent from `summaryTrendStats`.

`1.5.0` introduced an opt-in structured summary shared by
`--summary-export` and `handleSummary()`:

```sh
k6 run script.js --new-machine-readable-summary \
  --summary-export=summary.json
```

Enable it with `--new-machine-readable-summary` or
`K6_NEW_MACHINE_READABLE_SUMMARY`. It was announced as the intended v2
default, but consumers should inspect the actual selected format rather than
assuming the transition occurred.

## OpenTelemetry output

The OpenTelemetry output stabilized as `opentelemetry` in `1.4.0`:

```sh
k6 run --out opentelemetry script.js
```

The old `experimental-opentelemetry` alias still works but is deprecated.
`K6_OTEL_EXPORTER_TYPE` is also deprecated; use
`K6_OTEL_EXPORTER_PROTOCOL`.

The HTTP exporter gained Basic Auth in `2.1.0`. Configure credentials with
`K6_OTEL_HTTP_EXPORTER_USERNAME` and `K6_OTEL_HTTP_EXPORTER_PASSWORD`, or with
the `username` and `password` output-config keys:

```sh
K6_OTEL_HTTP_EXPORTER_USERNAME=user \
K6_OTEL_HTTP_EXPORTER_PASSWORD=secret \
k6 run --out opentelemetry script.js
```

## TLS behavior for metric outputs

The experimental OpenTelemetry and Prometheus outputs began defaulting to TLS
1.3 in `1.2.0`. This was a minor-release breaking change because the outputs
were experimental.

The experimental Prometheus remote-write output gained
`K6_PROMETHEUS_RW_TLS_MIN_VERSION` in `1.6.0`; its default remains TLS 1.3:

```sh
K6_PROMETHEUS_RW_TLS_MIN_VERSION=1.3 \
k6 run script.js -o experimental-prometheus-rw
```

## Rate metric representation

Exported Rate metrics changed in `1.3.0` to one counter with a label whose
values are `zero` and `nonzero`. Update queries and downstream consumers for
the labeled shape.

`K6_OTEL_SINGLE_COUNTER_FOR_RATE`, which had allowed postponing this migration,
was removed in `2.0.0`. Delete the variable from deployment configuration.

## Cloud metric corrections

Cloud output v2 corrected Gauge extrema in `1.8.0`: `min` and `max` now occupy
their proper fields. Queries no longer report the peak as the floor or the
floor as the peak.
