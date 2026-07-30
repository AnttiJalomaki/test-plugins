# OTLP ingestion

Use this reference when configuring the OTLP receiver, choosing translation or
temporality behavior, promoting metadata, or debugging target information,
histograms, start times, and request limits.

## Translation strategies

### UTF-8 and suffix behavior

From 3.0.0, `otlp.translation_strategy` can preserve UTF-8 without escaping
while retaining translated suffixes:

```yaml
otlp:
  translation_strategy: NoUTF8EscapingWithSuffixes
```

From 3.6.0, `UnderscoreEscapingWithoutSuffixes` escapes names with underscores
without adding translated suffixes:

```yaml
otlp:
  translation_strategy: UnderscoreEscapingWithoutSuffixes
```

The translation configuration can also preserve metric names and attributes in
their original OTLP forms from 3.4.0.

From 3.7.1, translating an OpenTelemetry attribute whose name begins with
exactly one underscore prefixes the Prometheus label with `key_`; multiple
consecutive leading underscores are preserved. This restores behavior changed
in 3.7.0.

### Reserved names

OTLP conversion filters the `__name__` attribute from 3.10.0 so it cannot
create a second label alongside the translated metric name.

With `--enable-feature=type-and-unit-labels`, OTLP metrics receive `__type__`
and `__unit__` labels from 3.6.0. These labels are reserved
(`feature-flags`): ingestion metadata overrides incoming user values, and
metadata WAL values override conflicting values on an existing Remote Write
2.0 series.

## Resource, scope, and metric metadata

### Identity and metadata

From 3.1.0, translation retains identifying attributes in `target_info`. The
receiver also translates metric metadata and accepts colons in non-standard
unit strings.

Prometheus emits `target_info` samples between the earliest and latest sample
for each OTLP resource from 3.6.0, keeping its resource metadata available
across the sample interval.

Duplicate `target_info` samples with the same series and timestamp are
de-duplicated from 3.8.0.

### Attribute promotion

The 3.5.0 `otlp` block supports broad resource-attribute promotion plus an
exclusion list:

```yaml
otlp:
  promote_all_resource_attributes: true
  ignore_resource_attributes:
    - service.instance.id
```

Set `otlp.promote_scope_metadata` from 3.6.0 to add OTLP scope metadata as
metric labels:

```yaml
otlp:
  promote_scope_metadata: true
```

OTLP receiver defaults apply even when the configuration has no explicit
`otlp:` block from 3.4.0.

## Delta temporality

### Delta-to-cumulative conversion

Enable `otlp-deltatocumulative` from 3.2.0 to convert delta-temporality metrics
to cumulative metrics instead of dropping them:

```text
--enable-feature=otlp-deltatocumulative
```

Conversion retains per-series state in memory. Restarting loses that state and
causes a counter reset; inactive state is periodically removed according to
`max_stale`.

### Native delta ingestion

Primitive ingestion of delta metrics without conversion appears in 3.4.0. The
full `otlp-native-delta-ingestion` contract is described by `feature-flags`:

- it stores raw delta values;
- it is mutually exclusive with `otlp-deltatocumulative`;
- it currently ignores `StartTimeUnixNano`;
- it records unknown metric metadata;
- identical-timestamp deltas are not summed;
- federation can collect them incorrectly when ingestion and scrape intervals
  differ;
- cumulative and delta data need an explicit distinguishing label if they can
  mix.

Counter functions such as `rate()` and `increase()` are incorrect for these
raw-delta series. Sum deltas directly over a range aligned to their collection
interval:

```promql
sum_over_time(delta_metric[5m])
sum_over_time(delta_metric[5m]) / 5m
```

## Histograms

An opt-in feature added in 3.4.0 translates OTLP explicit-bucket histograms
into native histograms with custom buckets.

Classic-histogram-to-NHCB conversion can run with created-timestamp zero
ingestion from 3.8.0.

Prometheus 3.11.0 fixes OTLP exemplar placement so exemplars are not mixed
between incorrect sections of a histogram.

## Start times

With `--enable-feature=created-timestamp-zero-ingestion`, the receiver writes
OTLP metric start times as created-time zero samples from 3.7.0.

The 3.11.0 `st-storage` feature stores OTLP start timestamps in the TSDB and
Agent WAL and exposes them through Remote Write 2. Its encoding and replay
requirements are in
[Histograms, TSDB, and storage](histograms-tsdb-and-storage.md).

`st-synthesis` is not implemented for OTLP (`feature-flags`); it applies only
to scraped cumulative metrics.

## Receiver correctness and limits

Prometheus 3.10.0 fixes a receiver path that could silently lose OTLP sum
metrics. OTLP-sum users should upgrade rather than rely on the affected
behavior.

The OTLP write endpoint limits the decompressed size of gzip-compressed request
bodies from 3.12.0.

Prometheus no longer fails to start when tracing is configured for insecure
OTLP over HTTP from 3.12.0.
