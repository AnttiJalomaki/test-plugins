# Migration, security, and operations

Use this reference for upgrade audits, process flags, reload behavior,
Alertmanager integration, container choices, and security-sensitive patch
selection.

## Core 3.x migration

### Promoted and replaced features

For the `3.0-migration` batch, remove these entries from `--enable-feature`
because they became default behavior:

- `promql-at-modifier`
- `promql-negative-offset`
- `new-service-discovery-manager`
- `expand-external-labels`
- `no-default-scrape-port`

External labels now expand `$var` and `${var}`; an undefined variable becomes
empty and `$$` escapes a dollar. Scrape target labels remain as configured and
do not gain ports inferred from their URL scheme.

Replace the old `agent` and `remote-write-receiver` feature flags with `--agent`
and `--web.enable-remote-write-receiver`. Automatic `GOMEMLIMIT` and
`GOMAXPROCS` sizing is default behavior; disable it only with
`--no-auto-gomemlimit` or `--no-auto-gomaxprocs`.

### Removed interfaces

The `storage.tsdb.allow-overlapping-blocks`, `alertmanager.timeout`, and
`storage.tsdb.retention` command-line flags are rejected from 3.0.0. Remove
them rather than retaining dead compatibility arguments.

The bundled JavaScript and template examples for the console feature were also
removed in 3.0.0. Console deployments must provide their own files.

### Logging format

The `3.0-migration` changes Prometheus logs from `go-kit/log` to `log/slog`.
Parsers expecting `ts`, `caller`, or lowercase levels must accept fields such
as `time`, `source`, and `level=INFO`.

## Configuration reload and rule operation

### Automatic reload lifecycle

In 3.0.0, `--enable-feature=auto-reload-config` enables experimental automatic
configuration reload. From 3.4.0 it also reacts to referenced rule files and
scrape configuration files rather than only the main file. The capability is
stable in 3.12.0.

Reloads apply `always_scrape_classic_histograms` and
`convert_classic_histograms_to_nhcb` correctly from 3.1.0; earlier reload
behavior could silently ignore those settings.

### Rule scheduling and validation

When dependency analysis is uncertain, rule evaluation is serialized rather
than concurrent from 3.1.0. Rule parse errors are detected during startup
earlier in the lifecycle from 3.5.0.

The `concurrent-rule-eval` feature evaluates dependency-free rules in the same
group concurrently. The separate `--rules.max-concurrent-evals` limit defaults
to `4`; use it to bound the extra query load (`feature-flags`).

Increasing an alert's `FOR` duration no longer resets the alert to pending in
3.11.0. State restoration also works when rule labels contain Go template
expressions.

## Alertmanager and notifications

### API and relabeling

The `3.0-migration` removes Alertmanager v1 API configuration. Alertmanager
0.16.0 or later is required; change an explicit `api_version: v1` to
`api_version: v2`.

Alert relabeling participates in the decision to drop an alert from 3.3.0. From
3.7.0, mutations made by `alertmanager_config.alert_relabel_configs` are scoped
to that Alertmanager configuration and are not passed into subsequent
Alertmanager configuration blocks.

### Notification behavior and metrics

`prometheus_notifications_errors_total` increments by the number of affected
alerts, not once per failed batch, from 3.1.0. Reinterpret dashboards and alert
thresholds accordingly.

Set the maximum alert notification batch size with
`--alertmanager.notification-batch-size` from 3.4.0.

In 3.10.0, configured Alertmanagers get independent notification send loops.
The following metrics also gain an `alertmanager` label:

- `prometheus_notifications_dropped_total`
- `prometheus_notifications_queue_capacity`
- `prometheus_notifications_queue_length`

Queries that previously assumed one unlabeled aggregate must aggregate across
the new dimension deliberately.

## Images, builds, and process operation

### Container filesystem and variants

The image's `/prometheus` directory is writable from 3.3.0.

Prometheus 3.10.0 publishes `-busybox` and `-distroless` variants. The
unsuffixed image remains the busybox image. Distroless runs as UID/GID 65532
and declares no `VOLUME`; adjust an existing named volume or bind mount before
migration:

```text
docker run --rm -v prometheus-data:/prometheus alpine chown -R 65532:65532 /prometheus
docker run -v prometheus-data:/prometheus prom/prometheus:latest-distroless
```

Prometheus 3.13.0 also publishes images through GitHub Container Registry.
Release tarballs and images no longer include `npm_licenses.tar.bz2`;
third-party npm licenses are served by the binary at
`/assets/third-party-licenses.txt`.

### Platform and readiness

The `aix/ppc64` compilation target is supported from 3.12.0.

From 3.10.0, `/-/ready` again returns `X-Prometheus-Stopping` in the
`NotReady` shutdown state. Health-check clients can use the header to
distinguish shutdown.

Concurrent Agent appends for the same label set no longer create duplicate
in-memory series or WAL records in 3.12.0.

### Time-series maintenance

The 3.12.0 Status menu includes a UI for deleting time series and cleaning
tombstones. Apply the same operational care as for the underlying destructive
admin endpoints.

## Patch-level security and correctness

### Required 3.11 maintenance level

Deploy at least 3.11.3 on the 3.11.0 line:

- CVE-2026-42151 prevents an AzureAD remote-write OAuth `client_secret` from
  being exposed through `/-/config`.
- CVE-2026-42154 enforces the declared-length limit on Snappy remote-read
  requests.
- CVE-2026-40179 and GHSA-fw8g-cg8f-9j28 close stored-XSS paths involving
  metric or label values in both current and old UIs.

### Other security fixes

STACKIT service-discovery users must choose a fixed 3.12.0 release because
affected builds expose STACKIT credentials in plaintext through `/-/config`.

Prometheus 3.13.0 updates `sanitize-html` for CVE-2026-44990. Deploy it or later
where the UI is exposed.

In 3.13.0, HTTP clients strip authorization headers, basic and bearer
credentials, OAuth2 credentials, and configured headers when a redirect
changes host. This applies to scrapes, remote read and write, alerting, and
service discovery and closes CVE-2025-4673 and CVE-2023-45289. Integrations
must not depend on credentials crossing hosts.

### Regression avoidance

Prometheus 3.9.1 fixes an Agent-mode crash shortly after startup and restores
scrape relabel `keep` and `drop`, which are broken in 3.9.0. Use 3.9.1 rather
than 3.9.0 for either path.

## Tracing and operational output

Prometheus no longer fails startup when tracing uses insecure OTLP over HTTP
from 3.12.0.

Query-log entries include both `traceID` and `spanID` when tracing is enabled
from 3.11.0.

`promtool` debug output moves to stderr in 3.11.0, leaving stdout available for
the tool's primary output. Pipelines that merged or parsed both streams must be
adjusted.
