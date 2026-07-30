# Migrations and Compatibility

## Release and support expectations

As of `1.0.0`, k6 follows Semantic Versioning. Breaking changes are confined
to major releases and receive deprecation warnings in advance. Each major
version receives critical fixes for at least two years. The supported public
API surface for extensions and integrations is explicitly defined; do not
assume internal Go packages are stable extension APIs.

## Migrating applications and automation to v2

### Go module and JSON contracts

In `2.0.0`, the Go module path changed to `go.k6.io/k6/v2`. Update every k6
import in extensions and external Go packages:

```go
import "go.k6.io/k6/v2/js/modules"
```

Public k6 Go types also stopped exposing easyjson-generated `MarshalJSON` and
`UnmarshalJSON` methods in `2.0.0`. Use standard `encoding/json` marshaling
instead.

### Removed live-control surface

The `externally-controlled` executor was removed in `2.0.0`, along with
`k6 pause`, `k6 resume`, `k6 scale`, and `k6 status`. There are no direct
replacements. Choose an executor such as `ramping-vus`,
`constant-vus`, or `constant-arrival-rate`; scripts that retain the removed
executor do not start.

### Cloud configuration and commands

The following old interfaces were removed in `2.0.0`:

- top-level `k6 login`;
- positional `k6 cloud script.js`;
- `k6 cloud --upload-only`;
- `options.ext.loadimpact`;
- login-managed InfluxDB credentials.

Use explicit commands and the Cloud namespace:

```sh
k6 cloud login
k6 cloud run script.js
k6 cloud upload script.js
```

```javascript
export const options = {
  cloud: { projectID: 12345, name: 'My Test' },
};
```

The `k6 login influxdb` flow is gone; supply InfluxDB credentials with
`K6_INFLUXDB_*` environment variables.

### Exit status and automation

A Cloud run aborted by the system, a limit, a script error, the user, or a
timeout exits with status `97` as of `2.0.0`. Successful runs remain `0`, and
threshold aborts remain `99`. CI must classify `97` as failure.

### Configuration and environment cleanup

The old `{USER_CONFIG_DIR}/loadimpact/config.json` path is not read or migrated
in `2.0.0`. Move the file to `{USER_CONFIG_DIR}/k6/config.json`, or regenerate
it with `k6 cloud login`.

`K6_BINARY_PROVISIONING` and `K6_ENABLE_COMMUNITY_EXTENSIONS` were removed in
`2.0.0`; community extensions resolve through the default build service.
`K6_AUTO_EXTENSION_RESOLUTION` is needed only when explicitly disabling
resolution. `K6_OTEL_SINGLE_COUNTER_FOR_RATE` was also removed, so downstream
systems must accept the single labeled Rate counter.

The HTTP API stopped listening on `localhost:6565` by default in `2.0.0`.
Enable it explicitly:

```sh
k6 run --address=localhost:6565 script.js
```

## Deprecation map

### Script modules and metrics

- Global Web Crypto became stable in `1.0.0-rc1`; use global `crypto` instead
  of deprecated `k6/experimental/webcrypto`, which was scheduled for removal
  in `1.1.0`.
- First Input Delay was scheduled to warn in the `1.4.x` line and disappear
  in `2.0.0`. Replace `browser_web_vital_fid` thresholds and integrations with
  Interaction to Next Paint, such as `browser_web_vital_inp: ['p(95)<200']`
  (`1.3.0` guidance).
- `k6/experimental/redis` was deprecated in `1.5.0`; migrate to the official
  Redis extension.
- WebSockets stabilized as `k6/websockets` in `1.6.0`;
  `k6/experimental/websockets` is deprecated.

### Summary and output names

- `--no-summary` and `K6_NO_SUMMARY` were deprecated in `1.3.0`; select
  `disabled` using `--summary-mode` or `K6_SUMMARY_MODE`.
- The `legacy` summary mode was deprecated in `1.3.0`; use `compact` or
  `full`.
- OpenTelemetry stabilized under `opentelemetry` in `1.4.0`. The
  `experimental-opentelemetry` alias remains deprecated.
- `K6_OTEL_EXPORTER_TYPE` was deprecated in `1.4.0`; use
  `K6_OTEL_EXPORTER_PROTOCOL`.

## Go build requirements

The source-build floor advanced several times:

| k6 release | Requirement |
| --- | --- |
| `1.0.0-rc1` | Go 1.23 or newer; release builds used Go 1.24 |
| `1.4.0` | Go 1.24 or newer |
| `1.7.0` | Go 1.25 or newer; default toolchain Go 1.26 |

Use the requirement for the k6 revision being built, not the oldest row.

## Container identity and image tags

The container switched to numeric UID `12345` in `1.1.0`, avoiding a need to
set `runAsUser` solely because the former named `k6` user could not be
resolved.

Release candidates updated Docker `latest` beginning with `1.0.0-rc1`, so
pinning an explicit tag was necessary to exclude prereleases. In `2.0.0`,
prereleases and maintenance releases from older majors stopped updating both
Docker `:latest` and the GitHub latest-release marker. Floating tags such as
`grafana/k6:v1` track a selected major line; the general tag form is `:vN`.
