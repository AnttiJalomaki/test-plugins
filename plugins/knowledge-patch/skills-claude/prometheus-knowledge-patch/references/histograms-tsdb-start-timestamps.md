# Histograms, TSDB, and Start Timestamps

## Native-histogram ingestion

### Activation and scraping

Native-histogram created timestamps and out-of-order samples can be ingested
(since 3.0.0). Once native histograms are stable, the `native-histograms`
feature name is a no-op, but scraping still requires the configuration introduced
before stabilization (`3.0-migration`):

```yaml
global:
  scrape_native_histograms: true
```

If ingestion is disabled, the scrape path skips native-histogram series (since
3.3.0).

To retain a classic histogram exposed alongside a native form, use
`always_scrape_classic_histograms`, replacing `scrape_classic_histograms`
(`3.0-migration`). It is available globally from 3.5.0. Conversion of classic
histograms to custom-bucket native histograms is global from 3.4.0 and can
coexist with created-timestamp zero ingestion from 3.8.0.

From 3.13.0, relabeling can override all three choices per target through
`__scrape_native_histograms__`, `__always_scrape_classic_histograms__`, and
`__convert_classic_histograms_to_nhcb__`.

### Out-of-order ingestion

`--enable-feature=ooo-native-histograms` is a no-op from 3.4.0. Out-of-order
native histograms are accepted whenever `out_of_order_time_window` is positive
and the native-histogram feature is active.

`prometheus_tsdb_sample_ooo_delta` records every accepted or rejected sample's
out-of-order distance in seconds (since 3.9.0).

### Schema and custom-bound validation

Unsupported native histogram schemas are rejected on append (since 3.7.0).
Scrape and remote write reduce schemas -9 through 52 to the supported maximum
resolution. WAL/WBL replay ignores invalid schemas, while exponential schemas
read from chunks or remote read are also reduced.

For custom bounds, a NaN threshold is rejected and `-Inf` is accepted as the
first bound (since 3.8.0). Federation supports custom-bucket native histograms,
but Remote Write 1.0 does not; attempts are prevented and logged (since 3.7.0).

### Query accounting and instrumentation

Histogram samples count toward PromQL sample limits (since 3.8.0). PromQL,
rules, service discovery, and scraping publish native histograms beside existing
summaries (since 3.9.0). Notification latency also exposes
`prometheus_notifications_latency_histogram_seconds`.

## TSDB lifecycle

### Downgrades and block consumers

The data format prepared in 2.55 means a 3.x data directory can only be opened
by 2.55 or later (`3.0-migration`). Upgrade through 2.55 before the major
upgrade. Downgrading lower requires discarding the 3.x persistent data.

XOR2, `histogramST`, and `floathistogramST` blocks are unreadable by older
releases (`feature-flags`). Their experimental formats can change, and
downstream block consumers may not support them. Treat activation as a downgrade
and interoperability boundary for existing blocks.

### Block loading and compaction

`/v1/status/tsdb/blocks` exposes loaded-block metadata (since 3.6.0).
`--storage.tsdb.block-reload-interval` controls block reload cadence (since
3.9.0). `--storage.tsdb.delay-compact-file.path` configures a Thanos-compatible
compaction-delay file (since 3.9.0).

`stale_series_compaction_threshold` enables experimental early compaction of
stale in-memory series and sets its threshold (since 3.10.0). From 3.13.0,
`CompactStaleHead` and `CompactSelectedSeries` retain label records through WAL
checkpoint/replay and preserve samples at chunk-range boundaries. Upgrade before
using early stale-series compaction because earlier eviction could cause replay
failure or sample loss.

### Retention

`storage.tsdb.retention.percentage` caps TSDB disk use as a percentage (since
3.11.0). Removing retention from the configuration falls back to CLI values,
and file-based `storage.tsdb.retention.time` no longer has the unit mismatch that
made retention one million times too long.

Percentage retention works with the new data path and preserves decimal
precision from 3.12.0.

### Replay and Head metrics

`prometheus_tsdb_wal_replay_unknown_refs_total` and
`prometheus_tsdb_wbl_replay_unknown_refs_total` count unknown series references
during replay (since 3.4.0). `prometheus_tsdb_head_stale_series` counts stale
Head series (since 3.6.0).

Metadata for automatic metrics can be written to the WAL through
`metadata-wal-records` (since 3.2.0).

### Fast startup and uncached writes

The experimental `fast-startup` feature records active-series state in
`series_state.json` in the WAL directory for reuse after restart (since 3.11.0).

The experimental `use-uncached-io` gate uses direct I/O for chunk writes on
Linux and bypasses the page cache (`feature-flags`). It is Linux-only.

## Float chunk encoding

`xor2-encoding` selects a float-sample block encoding optimized for scraped data
and capable of storing start timestamps (since 3.11.0):

```text
--enable-feature=xor2-encoding
```

`storage.tsdb.chunk_encoding.floats` chooses `xor` or `xor2` at runtime,
independently of the feature gate (since 3.13.0).

Upgrade for the 3.13.0 chunk-snapshot fix before relying on `EncXOR2`; the
earlier snapshot behavior could corrupt TSDB data on restart.

## Start-timestamp storage

### Storage gate and prerequisites

`st-storage` stores ingested start timestamps—formerly called Created
Timestamps—from scrapes or OTLP in TSDB and the Agent WAL, and exposes them over
Remote Write 2 (since 3.11.0):

```text
--enable-feature=st-storage
```

The gate does not choose an ST-capable encoding (`feature-flags`). Float chunks
must resolve to XOR2 or startup/config reload fails. Persisting native- and
float-histogram start timestamps also requires
`--enable-feature=histograms-st-encoding`. `SamplesV2` WAL records require 3.11
or later for replay.

### Ingestion and query use

With `use-start-timestamps`, `rate()`, `irate()`, and `increase()` use start
timestamps, while `resets()` detects start-timestamp resets (since 3.12.0). The
gate also enables `start_timestamp()` (`feature-flags`). It is incompatible with
anchored or smoothed extended selectors.

PromQL test `load` blocks accept `@st` on samples to define start timestamps
(since 3.12.0).

### Synthesis

`st-synthesis` creates missing start timestamps for scraped cumulative metrics
(since 3.12.0). Its transformation is significant (`feature-flags`):

- The first sample is dropped and becomes a reference value.
- That value is subtracted from subsequent samples, so stored raw values differ
  from scraped values.
- Remote write and OTLP ingestion are unsupported.
- Out-of-order samples for those metrics are rejected regardless of
  `out_of_order_time_window`.
- An append failure clears per-series state, making the next sample a newly
  dropped reference point.

Use synthesis when downstream Remote Write 2 or delta/OTLP systems require
start timestamps and the input transformation is acceptable.

## Created-timestamp zero ingestion

Created-timestamp zero ingestion no longer emits additional `_created` series
(since 3.0.0). OTLP start times can be written as zero samples under this gate
(since 3.7.0). It can run with classic-to-NHCB conversion (since 3.8.0), and it
changes the default scrape-protocol preference unless that preference is
configured explicitly (`feature-flags`).
