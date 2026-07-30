---
name: prometheus-knowledge-patch
description: Prometheus
version: 3.13.0
license: MIT
metadata:
  author: Nevaberry
---


# Prometheus

Use this skill when upgrading, configuring, querying, operating, or integrating
Prometheus. Start with compatibility and security hazards, then open only the
topic references needed for the task. Treat the running binary, its effective
configuration, and observed API behavior as authoritative.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration, security, and operations](references/migration-security-operations.md) | 3.x migration, removed flags, security fixes, reload behavior, images, logging, Alertmanager, runtime operations |
| [Scraping and ingestion](references/scraping-and-ingestion.md) | Protocol negotiation, UTF-8, relabeling, scrape controls, classic/native histogram ingestion, exemplars |
| [PromQL, rules, and templates](references/promql-rules-and-templates.md) | Query semantics, duration expressions, histogram functions, experimental selectors, rule evaluation, templates |
| [Histograms, TSDB, and storage](references/histograms-tsdb-and-storage.md) | Native histogram behavior, WAL and block formats, retention, start timestamps, compaction, encoding boundaries |
| [OTLP ingestion](references/otlp-ingestion.md) | Translation, delta metrics, resource and scope metadata, target info, start times, limits and tracing |
| [Remote read, remote write, and authentication](references/remote-io-and-auth.md) | Remote protocols, queue behavior, metrics, AzureAD, OAuth2, SigV4, validation and redirects |
| [Service discovery](references/service-discovery.md) | Kubernetes, AWS, Azure, Consul, Hetzner, STACKIT and other discovery providers |
| [HTTP APIs, promtool, and observability](references/apis-promtool-and-observability.md) | API fields and limits, feature discovery, OpenAPI, query statistics, promtool commands, self-monitoring |

## Apply breaking changes first

### Remove promoted or replaced feature flags

The following are default behavior and must not remain in `--enable-feature`:

- `promql-at-modifier`
- `promql-negative-offset`
- `new-service-discovery-manager`
- `expand-external-labels`
- `no-default-scrape-port`

Replace the old `agent` and `remote-write-receiver` feature flags with `--agent`
and `--web.enable-remote-write-receiver`. Automatic `GOMEMLIMIT` and
`GOMAXPROCS` sizing is also on by default; opt out only with
`--no-auto-gomemlimit` or `--no-auto-gomaxprocs`.

Remove the deleted startup flags `storage.tsdb.allow-overlapping-blocks`,
`alertmanager.timeout`, and `storage.tsdb.retention`.

### Correct 3.x scrape assumptions

A target without an accepted, parseable `Content-Type` now fails scraping.
Correct the exporter or set `fallback_scrape_protocol`; do not rely on implicit
Prometheus text parsing.

Native histograms are stable from 3.9, making `native-histograms` a no-op, but
scraping them still requires:

```yaml
global:
  scrape_native_histograms: true
```

Use `always_scrape_classic_histograms`, not the removed
`scrape_classic_histograms`, when classic and native forms must both be kept.

Metric and label names accept UTF-8. To retain the former validation:

```yaml
global:
  metric_name_validation_scheme: legacy
```

Classic histogram `le` and summary `quantile` values are normalized as
float-like strings. Change matchers such as `{le="1"}` to `{le="1.0"}` and
expect transition-spanning queries to be awkward.

### Respect data-format boundaries

A v3 data directory can be read only by v2.55 or newer. Upgrade through v2.55
before v3; downgrading below that floor means abandoning the v3 persistent
data.

Experimental XOR2, `histogramST`, and `floathistogramST` blocks are unreadable
by older releases and may not work with downstream block consumers.
`st-storage` also writes `SamplesV2` WAL records that require 3.11 or newer.
Treat these as explicit downgrade and interoperability decisions.

### Pin security-sensitive patch levels

On the 3.11 line, deploy at least 3.11.3. Earlier patch levels expose AzureAD
client secrets through `/-/config`, under-enforce declared Snappy remote-read
length, and contain stored-XSS paths in the UIs.

Use 3.13.0 or later for the `sanitize-html` UI fix. It also strips credentials
and configured headers when an HTTP redirect changes host across scraping,
remote I/O, alerting, and discovery.

STACKIT service-discovery users need a fixed 3.12 release because affected
versions expose its credentials through `/-/config`.

Use 3.9.1 rather than 3.9.0 for Agent mode or scrape relabel `keep`/`drop`:
3.9.0 can crash the Agent and breaks those relabel actions.

## Configure high-impact features deliberately

### Start timestamps and storage encodings

`st-storage` does not select a compatible float block encoding. Float chunks
must resolve to XOR2 or startup and reload are rejected. Native- and
float-histogram start timestamps additionally require:

```text
--enable-feature=histograms-st-encoding
```

`st-synthesis` rewrites scraped cumulative series: it drops the first sample
and subtracts it from later values. It does not support remote write or OTLP,
and it changes out-of-order and append-failure behavior. Do not enable it when
stored values must equal scraped values.

`use-start-timestamps` changes `rate()`, `irate()`, `increase()`, and
`resets()`, enables `start_timestamp()`, and cannot be combined with `anchored`
or `smoothed` selectors.

### OTLP delta temporality

Choose one delta strategy:

- `otlp-deltatocumulative` keeps per-series cumulative state. Restart loses
  that state and presents a counter reset.
- `otlp-native-delta-ingestion` stores deltas as received and is mutually
  exclusive with cumulative conversion.

For native delta series, `rate()` and `increase()` are wrong. Sum deltas over a
range aligned to the collection interval:

```promql
sum_over_time(delta_metric[5m])
sum_over_time(delta_metric[5m]) / 5m
```

Use a distinguishing label if cumulative and delta data can coexist.

### Extended range selectors

Enable experimental range modifiers with
`--enable-feature=promql-extended-range-selectors`.

- `anchored` is limited to `resets`, `changes`, `rate`, `increase`, and
  `delta`.
- `smoothed` is limited to `rate`, `increase`, and `delta`.
- Neither form supports subqueries.
- Rule groups using `smoothed` need `query_offset` of at least one scrape
  interval because evaluation requires a sample after the interval.

The experimental-function flag is
`--enable-feature=promql-experimental-functions`. Function names and semantics
under this flag are unstable. Duration-expression `min()` and `max()` were
renamed to `min_of()` and `max_of()`.

### Concurrent rules

`--enable-feature=concurrent-rule-eval` runs dependency-free rules in a group
concurrently. Bound the additional query load with
`--rules.max-concurrent-evals`; its default is `4`. When dependency analysis is
uncertain, evaluation remains serialized.

### Search API exposure

When `search-api` is enabled, `--web.search.max-limit` caps each search
endpoint request and defaults to `10000`; requests above it return HTTP 400.
The ordinary response default is `100` and is clamped to a lower operator cap.
Do not set the cap to `0` on an untrusted endpoint because that makes requests
unbounded.

## Query and integration checkpoints

### Histogram semantics

- `idelta()` and `irate()` support native histograms.
- `rate()`, `increase()`, and `delta()` return gauge histograms.
- Subtraction, or multiplication/division by a negative factor, also produces
  a gauge histogram.
- Time, clamp, scalar, and sort functions ignore histogram samples where
  documented in the references.
- `histogram_fraction()` accepts classic buckets, but
  `histogram_fraction()` and `histogram_quantile()` emit no value when classic
  and native forms coexist at one timestamp.
- Sample-limit enforcement counts histogram samples.

Inspect annotations: histogram operations can emit warn-level diagnostics for
counter-reset conflicts.

### Remote write

Remote-write HTTP/2 is opt-in because
`http_config.enable_http2` defaults to `false`. Enable it explicitly only when
the endpoint needs the older behavior.

Remote Write 1.0 cannot transport custom-bucket native histograms. Prometheus
blocks those sends and logs a warning. Remote Write 2 uses start-timestamp
terminology and returns HTTP 400 for too-old histogram samples so senders do
not retry them indefinitely.

Validate `remote_write.queue_config` during configuration review; current
Prometheus rejects invalid fields at load time.

### Alertmanager and alert rules

Alertmanager v1 API configuration is gone. Require Alertmanager 0.16.0 or later
and use `api_version: v2`.

Each Alertmanager now has an independent send loop. Notification queue,
capacity, and dropped metrics carry an `alertmanager` label, so aggregate
queries explicitly when preserving the former single-series interpretation.
Alert relabel mutations are scoped to their own Alertmanager configuration.

An unevaluated alert rule has state `unknown`; API and UI consumers must handle
it. Increasing an alert's `FOR` period no longer resets it to pending.

## Verification workflow

1. Inspect the exact Prometheus image tag or binary version and its startup
   arguments.
2. Run configuration and rule checks with the same PromQL feature gates used
   by the server.
3. Check exporter `Content-Type`, scrape protocol negotiation, histogram
   controls, and UTF-8 validation assumptions.
4. Review TSDB/WAL feature flags before any downgrade or shared-block
   deployment.
5. Exercise remote-write authentication, redirect behavior, queue validation,
   and protocol version in staging.
6. Update dashboards for renamed metrics, normalized labels, and added
   dimensions.
7. Query `/api/v1/features` when capability detection is safer than
   version inference.
8. Read the relevant reference file before changing experimental PromQL,
   start-timestamp, OTLP delta, or storage-encoding behavior.
