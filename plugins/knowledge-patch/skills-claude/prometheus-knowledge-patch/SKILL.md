---
name: prometheus-knowledge-patch
description: Prometheus
version: 3.13.0
license: MIT
metadata:
  author: Nevaberry
---


# Prometheus Knowledge Patch

Use this skill when upgrading, configuring, operating, or querying Prometheus and
the task depends on recent PromQL, ingestion, storage, API, or deployment
behavior. Check the running binary and configuration before applying advice;
feature-gated behavior may differ even between servers on the same release.

## Reference index

| Reference | Topics |
|---|---|
| [histograms-tsdb-start-timestamps.md](references/histograms-tsdb-start-timestamps.md) | Native and classic histograms, TSDB, WAL/WBL, retention, encodings, start timestamps |
| [http-apis-observability.md](references/http-apis-observability.md) | HTTP and status APIs, query statistics, server metrics, profiling, web UI |
| [migration-security-deployment.md](references/migration-security-deployment.md) | Major migration, removed flags, security fixes, images, platform and log changes |
| [promql.md](references/promql.md) | PromQL syntax, functions, range selectors, histogram semantics, result behavior |
| [promtool-ui-feature-flags.md](references/promtool-ui-feature-flags.md) | Promtool commands and tests, UI diagnostics, feature gates and constraints |
| [remote-storage-otlp.md](references/remote-storage-otlp.md) | Remote read/write, Remote Write 2, OTLP translation and ingestion, authentication |
| [scraping-configuration-rules.md](references/scraping-configuration-rules.md) | Scraping, content negotiation, relabeling, reloads, rules, alerts, templates |
| [service-discovery.md](references/service-discovery.md) | Kubernetes, cloud and catalog discovery, labels, filters, metrics and build tags |

## Upgrade blockers first

### Remove promoted or replaced feature flags

Do not leave these former feature names in `--enable-feature` after the major
upgrade:

```text
promql-at-modifier
promql-negative-offset
new-service-discovery-manager
expand-external-labels
no-default-scrape-port
```

They are normal behavior. Replace `agent` with `--agent` and
`remote-write-receiver` with `--web.enable-remote-write-receiver`. Automatic
`GOMEMLIMIT` and `GOMAXPROCS` sizing is also normal; opt out with
`--no-auto-gomemlimit` or `--no-auto-gomaxprocs`.

### Remove rejected startup flags

Delete `storage.tsdb.allow-overlapping-blocks`, `alertmanager.timeout`, and
`storage.tsdb.retention`. The last name is not the current retention-time flag.

### Respect the storage downgrade boundary

A data directory written by the major release can be opened only by 2.55 or
newer on downgrade. Upgrade through 2.55 before moving to the major release;
downgrading farther means discarding that persistent data. Experimental XOR2,
histogram start-timestamp encodings, and related blocks impose additional
downgrade and downstream-consumer boundaries.

### Patch security-sensitive lines

On the 3.11 line, deploy at least 3.11.3 for the AzureAD secret-disclosure,
Snappy remote-read length-limit, and stored-XSS fixes. Upgrade STACKIT discovery
users to a fixed 3.12 release. Use 3.13.0 or later where the web UI is exposed,
and rely on the corrected cross-host redirect behavior so configured credentials
are not forwarded to a different host.

### Keep protocol peers compatible

Alertmanager v1 configuration is gone; use Alertmanager 0.16.0 or later and
`api_version: v2`. Third-party remote-read storage must return only series that
match requested selectors. Remote Write 1.0 cannot transport custom-bucket
native histograms.

## Scraping quick reference

### Make scrape protocols explicit when endpoints are imperfect

A missing, unparsable, or unknown `Content-Type` now fails a scrape. Correct the
producer or set `fallback_scrape_protocol`. Accepted families include
protobuf-delimited, Prometheus text 0.0.4 or 1.0.0, and OpenMetrics 0.0.1 or
1.0.0.

Enabling created-timestamp zero ingestion changes the default negotiation order
to prefer `PrometheusProto` unless `scrape_protocols` is explicitly configured.

### Enable native histogram scraping in configuration

The old `native-histograms` gate becomes a no-op once native histograms are
stable, but scraping them still requires:

```yaml
global:
  scrape_native_histograms: true
```

To retain a concurrently exposed classic histogram, use
`always_scrape_classic_histograms`, not `scrape_classic_histograms`. It may be
global or job-local. Per-target relabeling can override these choices with
`__scrape_native_histograms__`, `__always_scrape_classic_histograms__`, and
`__convert_classic_histograms_to_nhcb__`.

### Audit label and parser assumptions

Metric and label names accept UTF-8. Set `metric_name_validation_scheme: legacy`
globally or per job to retain old validation. Relabel targets, replacements,
rule names, and `label_replace()` have their own UTF-8 support details in the
references.

Classic histogram `le` and summary `quantile` label values are normalized as
float-like strings, for example `"1.0"`. Update exact-match rules and dashboards.
The regular-expression dot now matches newlines too.

## PromQL quick reference

### Treat histogram results deliberately

`idelta()` and `irate()` accept native histograms. `rate()`, `increase()`, and
`delta()` return gauge histograms for histogram input. Addition and subtraction
can reconcile mismatched custom bounds, while negative multiplication or
division and subtraction also produce gauge typing.

Several scalar, sort, clamp, and time functions ignore histogram samples. A
classic/native mix at one timestamp makes `histogram_fraction()` and
`histogram_quantile()` emit no value. Histogram samples now count toward query
sample limits.

### Use duration expressions with the active spelling

Duration literals work as scalars and duration arithmetic is accepted. The
duration-expression gate adds `step()` and bounded-duration helpers. Use
`min_of()` and `max_of()` where the earlier experimental names `min()` and
`max()` are no longer accepted:

```promql
rate(http_requests_total[max_of(5m, step())])
```

Reject `NaN`, infinite, and out-of-range computed durations in generated
queries. Millisecond range selectors retain their exact precision.

### Constrain extended range selectors

`anchored` is limited to `resets`, `changes`, `rate`, `increase`, and `delta`;
`smoothed` is limited to `rate`, `increase`, and `delta`. Neither supports
subqueries. Rules using `smoothed` need a `query_offset` of at least one scrape
interval because evaluation needs a later sample.

Start-timestamp-aware rates cannot be combined with extended selectors. Consult
the PromQL reference for boundary behavior, native-histogram support, and fixes
to reset handling.

## OTLP and remote storage quick reference

### Choose exactly one delta strategy

`otlp-deltatocumulative` keeps per-series cumulative state in memory; restart
causes a counter reset and stale state is cleared according to `max_stale`.
`otlp-native-delta-ingestion` stores raw deltas and is mutually exclusive with
that conversion. Query raw deltas with `sum_over_time`, not `rate()` or
`increase()`, and align the range with collection cadence.

### Preserve or translate identity intentionally

OTLP can preserve UTF-8 names, use underscore escaping without suffixes, leave
names and attributes untranslated, promote scope metadata, or promote resource
attributes with exclusions. Reserved `__type__` and `__unit__` labels override
user values when their feature is enabled and are dropped by the same operations
that drop `__name__`.

### Validate remote-write assumptions

Remote-write HTTP/2 defaults off so parallel queues can use multiple sockets;
set `http_config.enable_http2: true` to retain the prior behavior. Configuration
loading now validates queue fields. Too-old Remote Write 2 histogram samples
return HTTP 400, and start-timestamp terminology replaces created timestamp in
the 2.0-rc.4 schema.

## Storage and start timestamps quick reference

### Satisfy encoding prerequisites

`st-storage` does not select a compatible block encoding. Float chunks must use
XOR2 or startup and reload fail; persisting histogram start timestamps also
requires `histograms-st-encoding`. Its `SamplesV2` WAL records require 3.11 or
later to replay.

Use `storage.tsdb.chunk_encoding.floats` to choose `xor` or `xor2` at runtime.
Upgrade for the XOR2 chunk-snapshot restart fix and the early-stale-series
checkpoint/replay fixes before relying on those paths.

### Understand synthesis before enabling it

`st-synthesis` drops the first scraped cumulative sample and subtracts it from
later values. It does not support remote write or OTLP, rejects those metrics'
out-of-order samples, and resets its per-series reference after append failure.
Stored values therefore differ from scraped values.

## Operational checks

1. Query `/api/v1/features` instead of inferring enabled behavior from the
   binary version.
2. Validate configuration and rule files before reload; automatic reload is
   stable and watches referenced rule and scrape files.
3. Check `/api/v1/status/self_metrics`, TSDB block metadata, WAL/WBL replay
   counters, stale-series metrics, and out-of-order-distance metrics during a
   rollout.
4. Keep API clients tolerant of pagination tokens, added fields and labels,
   unknown alert state, statistics caps, and sample/read accounting changes.
5. Use the topic references for exact flags, configuration keys, metrics, and
   edge cases before changing production behavior.
