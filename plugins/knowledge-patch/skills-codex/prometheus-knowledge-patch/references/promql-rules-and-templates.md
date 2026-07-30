# PromQL, rules, and templates

Use this reference for parser changes, query semantics, experimental functions,
histogram operations, rule evaluation, and template helpers.

## Language and parser changes

### Regular expressions and UTF-8

From 3.0.0, the `.` regular-expression metacharacter matches every character,
including newline. A selector such as `{label=~"a.b"}` can therefore match more
series after an upgrade.

PromQL `label_replace()` supports UTF-8 labels from 3.3.0. Rule names support
UTF-8 from 3.2.0, except that `{` and `}` are rejected by the common-mistake
checks.

`label_join()` no longer emits duplicate results from 3.3.0.

### Durations

PromQL durations and float literals are interchangeable without a feature flag
from 3.0.0, so this scalar expression is stable syntax:

```promql
time() - 1h
```

The parser accepts arithmetic in duration expressions from 3.4.0, including
computed range durations.

With `--enable-feature=promql-duration-expr`, 3.6.0 adds `step()` and initially
adds `min()` and `max()` for duration expressions:

```promql
rate(http_requests_total[max(5m, step())])
```

In 3.13.0 the experimental duration helpers are renamed to `min_of()` and
`max_of()` to avoid confusion with aggregation operators. Scalar functions
with the same names return the lesser or greater of two arguments:

```promql
rate(http_requests_total[max_of(5m, step())])
max_of(2, 5)
```

PromQL rejects `NaN`, infinite, and out-of-range duration expressions from
3.12.0 rather than silently constructing an invalid duration.

Range selectors preserve millisecond precision from 3.5.0. A selector such as
`[1001ms]` is no longer rounded to `[1s]`, which can change included boundary
samples.

The experimental `start()`, `end()`, and `range()` functions arrive in 3.12.0.
`range()` is valid in a duration expression:

```promql
foo[5m+range()]
```

`/parse_ast` includes duration expressions in its response from 3.12.0.

### Aggregation parameters and fill modifiers

Aggregation operators such as `quantile` and `topk` accept non-constant
parameter expressions from 3.5.0:

```promql
topk(scalar(desired_series_count), rate(http_requests_total[5m]))
```

Prometheus rejects `NaN` parameters passed to `topk()`, `bottomk()`,
`limitk()`, or `limit_ratio()` from 3.6.0.

Binary expressions gain `fill()`, `fill_left()`, and `fill_right()` in 3.10.0
to provide values for series missing from one or both sides:

```promql
left_metric + fill(0) right_metric
```

From 3.13.0, `fill_left()` and `fill_right()` retain expected samples in range
queries using `group_left` or `group_right`.

## Histogram query semantics

### Supported functions and result types

`idelta()` and `irate()` support native histograms from 3.3.0, with corrected
native-histogram counter-reset detection.

From 3.9.0, `rate()`, `increase()`, and `delta()` return gauge histograms for
histogram inputs. A histogram also becomes gauge-typed after subtraction or
after multiplication or division by a negative factor (3.7.0).

PromQL emits warn-level annotations for counter-reset conflicts in certain
histogram operations from 3.7.0.

### Functions that ignore histograms

Time-related functions and clamp functions omit histogram samples from mixed
float-and-histogram inputs from 3.1.0.

`scalar()`, `sort()`, and `sort_desc()` ignore native histogram samples from
3.3.0.

`sort()`, `sort_by_label()`, and `sort_by_label_desc()` produce a warning in a
range query from 3.12.0 because sorting has no effect there.

### Fractions, quantiles, and deviation

`histogram_fraction()` accepts classic bucket histograms as well as native
histograms from 3.4.0:

```promql
histogram_fraction(0, 0.2, rate(http_request_duration_seconds_bucket[5m]))
```

From 3.5.0, `histogram_fraction()` and `histogram_quantile()` return no value
when classic and native histograms coexist at the same timestamp.

`histogram_stddev()` and `histogram_stdvar()` use the arithmetic mean from
3.4.0, changing results produced by the previous mean.

The experimental variadic `histogram_quantiles` function computes multiple
quantiles in one call from 3.11.0.

### Arithmetic and trimming

Addition and subtraction reconcile native histograms with different custom
bucket boundaries from 3.8.0; identical NHCB bounds are no longer required.

PromQL adds `</` and `>/` in 3.11.0 to trim observations from native
histograms. The operations retain the appropriate histogram buckets.

`avg_over_time()` handles a range containing one native histogram correctly
from 3.10.0.

Sample-limit enforcement counts histogram samples from 3.8.0, so
histogram-heavy queries can now hit the configured query limit.

## Extended range selectors

### Feature gate and supported functions

Enable experimental `anchored` and `smoothed` range modifiers with
`--enable-feature=promql-extended-range-selectors` from 3.7.0.

The `feature-flags` contract restricts them:

- `anchored` is accepted only by `resets`, `changes`, `rate`, `increase`, and
  `delta`.
- `smoothed` is accepted only by `rate`, `increase`, and `delta`.
- Extended selectors do not support subqueries.

Rule groups using `smoothed` must set `query_offset` to at least one scrape
interval. The modifier needs a sample after the evaluation interval and can
otherwise underestimate results.

Experimental anchored and smoothed rate evaluation supports native histograms
from 3.13.0.

### Boundary and reset behavior

`resets()` and `changes()` return an empty result for an anchored selector when
every sample lies outside the requested range from 3.9.0.

Smoothed selectors interpolate correctly across counter resets from 3.10.0.
From 3.12.0, smoothed `rate()` and `increase()` return no result rather than
zero when all data is strictly after the range. Smoothed selectors also work
in binary operations using an `@` modifier.

`resets()` and `changes()` produce corrected results for histograms used with
anchored selectors from 3.13.0.

### Start-timestamp incompatibility

With `--enable-feature=use-start-timestamps`, `rate()`, `irate()`, and
`increase()` use start timestamps and `resets()` detects start-timestamp
resets from 3.12.0. The same flag enables `start_timestamp()`
(`feature-flags`). This mode cannot be combined with anchored or smoothed
extended selectors.

## Experimental functions

Enable experimental functions with the current spelling
`--enable-feature=promql-experimental-functions` (`feature-flags`). Their
names, syntax, and behavior are explicitly unstable.

Under the experimental-function gate:

- 3.5.0 provides `ts_of_min_over_time()`, `ts_of_max_over_time()`, and
  `ts_of_last_over_time()`:

  ```promql
  ts_of_last_over_time(up[5m])
  ```

- 3.7.0 provides `first_over_time()` and `ts_of_first_over_time()`:

  ```promql
  first_over_time(metric[5m])
  ts_of_first_over_time(metric[5m])
  ```

The `type-and-unit-labels` feature exposes type and unit metadata as labels
from 3.5.0:

```text
--enable-feature=type-and-unit-labels
```

Incoming user values for the reserved `__type__` and `__unit__` labels are
overridden by ingestion metadata. PromQL drops them in the same classes of
operation that drop `__name__`; metadata WAL values take precedence over
conflicting labels already carried by Remote Write 2.0 (`feature-flags`).

## `info()` and metric-name behavior

The `info()` function retains series without identifying labels and handles a
filter on a label present in both the input and `target_info` correctly from
3.10.0. Either fix can add results.

Negated `__name__` matchers in `info()` work correctly from 3.12.0.

`last_over_time()` and `first_over_time()` drop the metric name from 3.12.0
when applied to a subquery containing a name-dropping function such as
`abs()`.

## Rules and tests

Rule files accept YAML anchors and aliases from 3.3.0.

When rule dependency analysis is ambiguous, Prometheus uses conservative
serialized evaluation from 3.1.0. Rule parse failures are detected earlier
during startup from 3.5.0.

An alerting rule that has not yet been evaluated has the explicit `unknown`
state from 3.8.0. API and UI consumers must handle it alongside established
states.

Promtool rule-unit-test files support `fuzzy_compare: true` from 3.5.0 when
exact float64 equality is too strict:

```yaml
fuzzy_compare: true
```

They accept `start_timestamp` from 3.9.0 for a fixed test origin. PromQL test
`load` blocks accept `@st` sample annotations from 3.12.0 for explicit start
timestamps.

Promtool can enable PromQL feature gates from 3.4.0. By 3.10.0 it understands
both `promql-duration-expr` and `promql-extended-range-selectors`, allowing
offline checks to parse the same gated syntax as the server.

## Template helpers

Prometheus templates provide `toDuration()` and `now()` from 3.6.0.

Alert templates provide `urlQueryEscape` from 3.8.0 for dynamic URL query
values:

```text
{{ urlQueryEscape $labels.instance }}
```
