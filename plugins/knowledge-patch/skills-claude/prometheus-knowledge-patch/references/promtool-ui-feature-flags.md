# Promtool, UI, and Feature Flags

## Promtool configuration and rule checks

`promtool` adds the `too-long-scrape-interval` lint option for excessive scrape
intervals (since 3.2.0). Use `--ignore-unknown-fields` when a configuration
contains fields the installed promtool does not recognize (since 3.2.0).

Rule unit tests accept fuzzy float64 comparisons from 3.5.0:

```yaml
fuzzy_compare: true
```

Rule-unit-test definitions accept `start_timestamp` for an explicit test origin
(since 3.9.0). PromQL test `load` blocks accept `@st` on individual samples for
start timestamps (since 3.12.0).

Promtool understands feature-gated PromQL syntax when the matching flags are
selected (since 3.4.0). In particular, support for `promql-duration-expr` and
`promql-extended-range-selectors` syntax was added in 3.10.0.

## Promtool data and query commands

`promtool tsdb create-blocks-from openmetrics` reads OpenMetrics input from a
pipe (since 3.3.0).

Dump series labels only as JSON from 3.9.0:

```text
promtool tsdb dump --format seriesjson
```

Select a Remote Write 2.0 protobuf message for `promtool push metrics` with
`--protobuf_message` (since 3.8.0).

`promtool query instant` accepts `--header`, matching `query range` (since
3.12.0).

Debug diagnostics go to stderr rather than stdout (since 3.11.0). Pipelines can
therefore preserve stdout as command output, but pipelines that merged both
streams may need adjustment.

Relative paths inside a file passed through `--http.config.file` resolve from
that file's directory, not an additional parent directory (since 3.13.0).

## Web UI changes

Example console JavaScript and templates are no longer bundled (since 3.0.0);
provide custom console files if using that feature.

The target UI can show a discovered target's relabeling trace, including how it
was dropped (since 3.8.0). An alerting rule that has not evaluated appears in
the `unknown` state from 3.8.0.

The Status menu provides time-series deletion and tombstone cleanup from 3.12.0.
Prometheus 3.13.0 includes the `sanitize-html` fix for CVE-2026-44990; use it or
later for exposed UIs. Third-party license text is available at
`/assets/third-party-licenses.txt` from that release.

## Discovering active features

Use `/api/v1/features` to inspect supported server features (since 3.9.0).
Do not infer runtime activation from a release string alone; inspect startup
flags and configuration too.

## PromQL gates

### Experimental functions

Use the current spelling (`feature-flags`):

```text
--enable-feature=promql-experimental-functions
```

Names, syntax, and semantics under this gate are unstable. Functions introduced
under an earlier experimental-functions spelling include the range-extrema
timestamp helpers (since 3.5.0) and `first_over_time()` /
`ts_of_first_over_time()` (since 3.7.0).

### Duration expressions

`promql-duration-expr` enables `step()` plus duration-bound helpers (`3.6.0`).
The experimental `min()` and `max()` spellings became `min_of()` and `max_of()`
in 3.13.0. Promtool can parse this gated syntax from 3.10.0.

### Extended range selectors

`promql-extended-range-selectors` enables experimental `anchored` and `smoothed`
selectors (since 3.7.0). Their constraints are fixed (`feature-flags`):

- `anchored`: `resets`, `changes`, `rate`, `increase`, or `delta`.
- `smoothed`: `rate`, `increase`, or `delta`.
- No extended selector supports a subquery.
- A recording or alerting group using `smoothed` needs `query_offset` of at
  least one scrape interval.

### Type and unit labels

`type-and-unit-labels` exposes experimental metric type and unit metadata as
labels (since 3.5.0). OTLP metrics receive `__type__` and `__unit__` from 3.6.0,
and outgoing Remote Write 2 series carry them from 3.7.0.

These names are reserved (`feature-flags`): ingestion metadata overwrites user
values, metadata WAL values beat conflicting Remote Write 2 labels, and PromQL
drops them where it would drop `__name__`.

## Ingestion gates

### OTLP delta handling

`otlp-deltatocumulative` converts delta temporality while maintaining in-memory
per-series state (since 3.2.0). `otlp-native-delta-ingestion` stores raw deltas
and is mutually exclusive with conversion (`feature-flags`). Raw delta series
must be summed over an interval; counter-rate functions are not correct for
them.

### Created-timestamp zero ingestion

`created-timestamp-zero-ingestion` no longer emits extra `_created` series
(since 3.0.0). It can write OTLP starts as zero samples (since 3.7.0).

Unless `scrape_protocols` is explicit, the gate changes negotiation preference
to protobuf first, then OpenMetrics 1.0.0, OpenMetrics 0.0.1, and Prometheus text
0.0.4 (`feature-flags`).

### Start-timestamp storage and use

`st-storage` stores start timestamps in TSDB/Agent WAL and exports them through
Remote Write 2 (since 3.11.0). It requires XOR2 for float chunks and
`histograms-st-encoding` for histogram start timestamps; `SamplesV2` WAL data
requires 3.11 or later to replay (`feature-flags`).

`use-start-timestamps` changes rate and reset evaluation and enables
`start_timestamp()` (since 3.12.0 and `feature-flags`). It cannot be combined
with extended selectors.

`st-synthesis` derives missing starts for scraped cumulative series (since
3.12.0). It drops and uses the first sample as a subtraction baseline, does not
support remote write or OTLP, rejects relevant out-of-order samples, and clears
state after append failure (`feature-flags`).

## Storage and runtime gates

`metadata-wal-records` writes metadata for automatic metrics into the WAL (since
3.2.0).

`fast-startup` writes active-series state to `series_state.json` for restart
reuse (since 3.11.0).

`xor2-encoding` selects XOR2 block encoding (since 3.11.0). The runtime
`storage.tsdb.chunk_encoding.floats` setting can choose XOR2 independently from
3.13.0. ST-capable experimental encodings create downgrade and interoperability
boundaries.

`use-uncached-io` performs Linux direct-I/O chunk writes and bypasses page cache
(`feature-flags`); it has no effect as a portable non-Linux feature.

## Configuration and rule gates

`auto-reload-config` was experimental in 3.0.0, learned to watch referenced
rule and scrape files in 3.4.0, and is stable from 3.12.0.

`concurrent-rule-eval` evaluates dependency-free rules within a group in
parallel (`feature-flags`). Bound the added query load with
`--rules.max-concurrent-evals`, default 4.

The deprecated `extra-scrape-metrics` feature flag is replaced by the global or
job-local `extra_scrape_metrics: true` configuration (`feature-flags`).

## Search gate

`search-api` enables experimental metric-name and label search endpoints (since
3.13.0). `--web.search.max-limit` defaults to 10000, returns HTTP 400 above the
cap, clamps the normal response default of 100 to a smaller cap, and permits
unsafe unbounded requests when set to zero (`feature-flags`).
