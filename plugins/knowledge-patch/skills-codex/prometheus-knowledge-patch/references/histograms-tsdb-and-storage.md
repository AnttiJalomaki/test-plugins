# Histograms, TSDB, and storage

Use this reference before enabling experimental encodings, changing retention,
downgrading, sharing blocks with other systems, or diagnosing native histogram
storage behavior.

## Upgrade and downgrade boundaries

### v3 TSDB floor

The TSDB format prepared in v2.55 means a v3 data directory can be read only by
v2.55 or newer (`3.0-migration`). Upgrade through v2.55 before v3 as a safety
step. Downgrading below v2.55 requires abandoning the v3 persistent data.

### Experimental start-timestamp encodings

The `feature-flags` guidance treats XOR2, `histogramST`, and
`floathistogramST` blocks as a compatibility boundary:

- older Prometheus releases cannot read these block encodings;
- their experimental formats may change between releases;
- downstream block consumers may not support them;
- `SamplesV2` WAL records written by start-timestamp storage require
  Prometheus 3.11 or later for replay.

Do not enable them until downgrade and interoperability plans account for
already-written blocks and WAL records.

## Start timestamps

### Storage

The 3.11.0 `st-storage` feature stores ingested start timestamps—formerly
called Created Timestamps—from scrapes or OTLP in the TSDB and Agent WAL and
exposes them through Remote Write 2:

```text
--enable-feature=st-storage
```

The feature does not choose a start-timestamp-capable block encoding by itself
(`feature-flags`). Float chunks must resolve to XOR2 or startup/configuration
reload is rejected. Persisting native- and float-histogram start timestamps
also requires:

```text
--enable-feature=histograms-st-encoding
```

The `use-start-timestamps` query feature is covered in
[PromQL, rules, and templates](promql-rules-and-templates.md).

### Synthesis

The 3.12.0 `st-synthesis` feature synthesizes missing start timestamps for
scraped cumulative metrics.

Its exact input-rewrite behavior comes from `feature-flags`:

- it drops the first sample;
- it subtracts that first value from later samples, so stored raw values differ
  from scraped values;
- it is not implemented for remote write or OTLP;
- it rejects out-of-order samples for those metrics regardless of
  `out_of_order_time_window`;
- after an append failure it clears synthesis state, causing the next sample to
  become a newly dropped reference point.

Enable it only when these transformed-series semantics are acceptable.

## Float and histogram encodings

### XOR2

The 3.11.0 `xor2-encoding` feature chooses an alternate float-sample chunk
encoding for TSDB blocks. XOR2 is optimized for scraped data and can encode
start timestamps:

```text
--enable-feature=xor2-encoding
```

From 3.13.0, `storage.tsdb.chunk_encoding.floats` selects `xor` or `xor2` at
runtime, independently of the feature flag.

Prometheus 3.13.0 fixes chunk-snapshot encoding for `EncXOR2`. Earlier behavior
could corrupt the TSDB on restart when XOR2-encoded series were present.
Deployments using XOR2 should include this restart-safety fix.

### Native histogram schema validation

From 3.7.0, unsupported native histogram schemas are rejected on append. For
scrape and remote write, schemas from -9 through 52 are reduced in resolution
to fit the supported maximum. Invalid schemas are ignored during WAL/WBL
replay, and exponential schemas loaded from chunks or remote read are also
reduced in resolution.

Custom-bucket native histograms reject a NaN threshold from 3.8.0, while
`-Inf` is accepted as the first custom-bound value.

Native histograms with custom buckets can pass through federation from 3.7.0.
Remote Write 1.0 cannot carry them, so Prometheus prevents that send and logs a
warning.

## WAL, WBL, and Head observability

Metadata for automatic metrics is written into the WAL when
`metadata-wal-records` is enabled from 3.2.0.

Use these counters from 3.4.0 to detect unknown series references during
replay:

- `prometheus_tsdb_wal_replay_unknown_refs_total`
- `prometheus_tsdb_wbl_replay_unknown_refs_total`

Use `prometheus_tsdb_head_stale_series` from 3.6.0 for the number of stale
series in the TSDB Head block.

Use `prometheus_tsdb_sample_ooo_delta` from 3.9.0 for the out-of-order distance
in seconds of every sample, whether accepted or rejected.

PromQL, rules, discovery, and scrape instrumentation expose native histograms
alongside existing summaries from 3.9.0. Notification latency also adds
`prometheus_notifications_latency_histogram_seconds`.

## Block loading and compaction

### Block status and reload

`/v1/status/tsdb/blocks` lists loaded TSDB block metadata from 3.6.0.

Use `--storage.tsdb.delay-compact-file.path` from 3.9.0 to configure the
compaction-delay file path for improved Thanos interoperability.

Use `--storage.tsdb.block-reload-interval` from 3.9.0 to control how often TSDB
blocks are reloaded.

The TSDB status endpoint returns no more than 10,000 sets of statistics from
3.9.0. Clients must not interpret the response as an unbounded inventory.

### Stale-series compaction

The 3.10.0 `stale_series_compaction_threshold` configuration key enables
experimental early compaction of stale series from memory and selects its
threshold.

Prometheus 3.13.0 corrects `CompactStaleHead` and `CompactSelectedSeries` so
label records survive WAL checkpoint/replay and samples at chunk-range
boundaries are preserved. Earlier eviction paths can cause replay failures or
sample loss; early-compaction users should upgrade.

### Fast startup

The 3.11.0 experimental `fast-startup` feature writes active-series state to
`series_state.json` in the WAL directory and reuses it on restart:

```text
--enable-feature=fast-startup
```

## Retention

Prometheus 3.11.0 adds `storage.tsdb.retention.percentage`, which caps TSDB
storage at a percentage of disk capacity.

The same release also corrects configuration fallback:

- removing retention from the configuration file falls back to CLI values;
- file-based `storage.tsdb.retention.time` no longer has a unit mismatch that
  made retention one million times too long.

From 3.12.0, percentage retention works with the new data path and preserves
decimal precision.

## Native histogram correctness

The TSDB can ingest out-of-order native-histogram samples from 3.0.0. Scrape
enablement and `out_of_order_time_window` details are in
[Scraping and ingestion](scraping-and-ingestion.md).

Prometheus validates histograms returned by remote read from 3.9.0 rather than
silently accepting malformed data.

Classic-histogram-to-NHCB conversion can coexist with created-timestamp zero
ingestion from 3.8.0.

Concurrent Agent appends for one label set no longer create duplicate
in-memory series or duplicate WAL records from 3.12.0.

## I/O behavior

The experimental `--enable-feature=use-uncached-io` performs direct chunk
writes on Linux, bypassing the page cache (`feature-flags`). It is Linux-only.
