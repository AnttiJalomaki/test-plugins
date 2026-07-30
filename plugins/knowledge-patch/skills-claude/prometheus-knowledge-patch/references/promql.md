# PromQL

## Syntax and matching changes

### Regular expressions

The `.` metacharacter matches every character, including newlines (since 3.0.0).
A matcher such as `{label=~"a.b"}` can therefore select more series than before.

### UTF-8 label replacement

`label_replace()` supports UTF-8 label names (since 3.3.0). Scrape relabeling and
rule-name support have separate constraints in the configuration reference.

`label_join()` no longer produces duplicate results (since 3.3.0).

### Metric-name dropping

`last_over_time()` and `first_over_time()` drop the metric name when applied to
a subquery whose inner function, such as `abs()`, drops the name (since 3.12.0).

## Duration expressions

### Literals and arithmetic

Duration and float literals are interchangeable without a feature gate (since
3.0.0), so this is a scalar expression:

```promql
time() - 1h
```

Arithmetic is accepted in duration expressions (since 3.4.0), including values
used by range selectors. Range selectors keep millisecond precision rather than
rounding `[1001ms]` to `[1s]` (since 3.5.0), which can change boundary samples.

Invalid computed durations are rejected rather than producing an out-of-range
duration: `NaN`, infinity, and values outside the supported range all fail
(since 3.12.0).

### Query-resolution helpers

With `promql-duration-expr`, duration expressions support `step()` and bounded
duration helpers (`3.6.0`). The helpers originally appeared as experimental
`min()` and `max()`; use `min_of()` and `max_of()` from 3.13.0:

```promql
rate(http_requests_total[max_of(5m, step())])
max_of(2, 5)
```

The scalar forms of `min_of()` and `max_of()` return the smaller or larger of
two scalar arguments.

Experimental `start()`, `end()`, and `range()` expose query boundaries (since
3.12.0). `range()` is valid in a duration expression:

```promql
foo[5m+range()]
```

## Aggregations and missing-series defaults

Aggregation parameters such as those to `quantile` and `topk` may be dynamic
expressions (since 3.5.0):

```promql
topk(scalar(desired_series_count), rate(http_requests_total[5m]))
```

`topk()`, `bottomk()`, `limitk()`, and `limit_ratio()` reject `NaN` parameters
(since 3.6.0).

Binary expressions support `fill()`, `fill_left()`, and `fill_right()` defaults
for unmatched series (since 3.10.0):

```promql
left_metric + fill(0) right_metric
```

`fill_left()` and `fill_right()` retain expected samples in range queries using
`group_left` or `group_right` (since 3.13.0).

## Range-vector functions

### Timestamps and first samples

With `experimental-promql-functions`, `ts_of_min_over_time()`,
`ts_of_max_over_time()`, and `ts_of_last_over_time()` return timestamps for
range-vector values (since 3.5.0). Later feature-flag spelling is covered below.

`first_over_time()` and `ts_of_first_over_time()` are available through the
experimental-functions gate (since 3.7.0):

```promql
first_over_time(metric[5m])
ts_of_first_over_time(metric[5m])
```

### Range-query sort warnings

PromQL warns when `sort()`, `sort_by_label()`, or `sort_by_label_desc()` is used
in a range query, where sorting has no effect (since 3.12.0).

## Histogram semantics

### Supported functions and ignored samples

`idelta()` and `irate()` support native histograms, with corrected counter-reset
detection (since 3.3.0). `rate()`, `increase()`, and `delta()` produce gauge
histograms for histogram input (since 3.9.0).

Time-related functions and clamp functions ignore histogram samples (since
3.1.0). `scalar()`, `sort()`, and `sort_desc()` ignore native histogram samples
(since 3.3.0). A range containing exactly one native histogram is handled
correctly by `avg_over_time()` (since 3.10.0).

### Fractions, quantiles, and deviation

`histogram_fraction()` accepts classic bucket histograms as well as native
histograms (since 3.4.0):

```promql
histogram_fraction(0, 0.2, rate(http_request_duration_seconds_bucket[5m]))
```

When classic and native histograms coexist at one timestamp,
`histogram_fraction()` and `histogram_quantile()` return no value (since 3.5.0).

`histogram_stddev()` and `histogram_stdvar()` use the arithmetic mean (since
3.4.0), so results can differ from earlier calculations.

Experimental `histogram_quantiles` computes several quantiles in one variadic
call (since 3.11.0).

### Arithmetic, bounds, and typing

Addition and subtraction reconcile mismatched custom bucket boundaries (since
3.8.0). Native histograms become gauge histograms after subtraction, or after
multiplication or division by a negative value (since 3.7.0).

The `</` and `>/` operators trim observations from native histograms while
retaining the correct buckets (since 3.11.0).

PromQL emits warn-level annotations for counter-reset conflicts in applicable
histogram operations (since 3.7.0). Histogram samples count toward query sample
limits (since 3.8.0).

## Extended range selectors

### Gates and allowed functions

Enable experimental `anchored` and `smoothed` rate range selectors with
`--enable-feature=promql-extended-range-selectors` (since 3.7.0).

The constraints are strict (`feature-flags`):

- `anchored` is accepted by `resets`, `changes`, `rate`, `increase`, and
  `delta` only.
- `smoothed` is accepted by `rate`, `increase`, and `delta` only.
- Extended selectors do not support subqueries.
- Alerting and recording rule groups using `smoothed` need `query_offset` of at
  least one scrape interval, because evaluation needs a sample after the range.

### Boundary behavior and corrections

For an anchored selector, `resets()` and `changes()` return an empty result when
all samples are outside the requested range (since 3.9.0). Their native
histogram reset detection is corrected from 3.13.0.

Smoothed `rate()` and `increase()` return no result, rather than zero, when all
data lies strictly after the query range (since 3.12.0). Smoothed vector
selectors work in binary expressions with an `@` modifier. Counter-reset
interpolation is corrected from 3.10.0.

Experimental anchored and smoothed rate evaluation supports native histograms
(since 3.13.0).

## Start timestamps in PromQL

With `--enable-feature=use-start-timestamps`, `rate()`, `irate()`, and
`increase()` use stored start timestamps, and `resets()` detects start-timestamp
resets (since 3.12.0). This mode is incompatible with anchored and smoothed
selectors.

The same gate enables `start_timestamp()` (`feature-flags`). It exposes a
sample's start timestamp and likewise does not work with extended range
selectors.

PromQL test data can supply start timestamps; see the promtool reference.

## `info()` behavior

`info()` retains series without identifying labels and handles a filter whose
label occurs in both the input metric and `target_info` (since 3.10.0). These
fixes may add results. Negated `__name__` matchers work correctly (since 3.12.0).

## Experimental function gate

Use `--enable-feature=promql-experimental-functions` for experimental PromQL
functions (`feature-flags`). Their names, syntax, and semantics are unstable;
avoid treating experimental spellings as permanent API.
