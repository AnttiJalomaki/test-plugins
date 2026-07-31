# Migration and Compatibility

## Apply the support contract

k6 follows Semantic Versioning beginning with 1.0.0: breaking changes are
reserved for major releases and receive prior deprecation warnings. Each major
version receives critical fixes for at least two years, and the supported
public API surface is explicitly delineated for extensions and integrations.
Keep integrations on that public surface rather than relying on internal Go
packages or accidental behavior.

## Complete the v2 migration

### Update Go module imports and JSON handling

k6 v2 uses the Go module path `go.k6.io/k6/v2` (2.0.0). Extensions and external
Go packages must update every import:

```go
import "go.k6.io/k6/v2/js/modules"
```

Public k6 Go types no longer provide easyjson-generated `MarshalJSON` and
`UnmarshalJSON` methods. Use standard `encoding/json` marshaling instead.

### Replace live test control

The `externally-controlled` executor and the `k6 pause`, `resume`, `scale`, and
`status` commands were removed without replacements in 2.0.0. Tests using that
executor do not start. Choose an executor such as `ramping-vus`, `constant-vus`,
or `constant-arrival-rate` and redesign any external live-control workflow.

### Migrate Cloud CLI and script options

The following interfaces were removed in 2.0.0:

- top-level `k6 login`
- positional `k6 cloud script.js`
- `--upload-only`
- `k6 login influxdb`
- `options.ext.loadimpact`

Use `k6 cloud login`, `k6 cloud run`, and `k6 cloud upload`. Provide InfluxDB
credentials with `K6_INFLUXDB_*`, and move Cloud fields to `options.cloud`.
See [Cloud and secrets](cloud-and-secrets.md) for commands and project
resolution.

### Remove obsolete environment switches

Delete these v2-removed switches from deployment configuration (2.0.0):

- `K6_OTEL_SINGLE_COUNTER_FOR_RATE`; the single-counter Rate shape is final.
- `K6_BINARY_PROVISIONING`; automatic extension resolution is the normal path.
- `K6_ENABLE_COMMUNITY_EXTENSIONS`; community extensions use the default build
  service.

`K6_AUTO_EXTENSION_RESOLUTION` remains only as an explicit disable switch.

### Move the configuration file

k6 v2 no longer reads or migrates
`{USER_CONFIG_DIR}/loadimpact/config.json` (2.0.0). Move it to
`{USER_CONFIG_DIR}/k6/config.json`, or create a current file with `k6 cloud
login`.

### Update CI exit-code handling

A Cloud run aborted by the system, limits, a script error, the user, or timeout
returns `97` in v2 (2.0.0). Success remains `0`; threshold aborts remain `99`.
Treat `97` as a failing CI result.

### Opt into the HTTP API

The k6 HTTP API no longer listens on `localhost:6565` by default in v2
(2.0.0). Use `--address` or `K6_ADDRESS` when a controller needs it:

```sh
k6 run --address=localhost:6565 script.js
```

### Use built-in and provisioned components

The web dashboard is built into k6 v2 (2.0.0):

```sh
k6 run --out=web-dashboard script.js
```

Remove the separate xk6-dashboard extension. Archives now preserve pre-manifest
`k6/x/` constraints in `metadata.json`, and provisioned `k6 x` subcommands
receive `K6_PROVISION_HOST_VERSION`. See
[Extensions and dependencies](extensions-and-dependencies.md) for the full
resolution workflow.

## Track build toolchain requirements

The source-build floor changes with the k6 line:

- Building 1.4.0 requires Go 1.24 or newer.
- Building 1.7.0 requires Go 1.25 or newer; its default Go toolchain is 1.26.

Use the requirement for the k6 line being compiled, not a floor copied from an
older build image.

## Run containers with the numeric user

The container image selects numeric UID `12345` rather than a named `k6` user
(since 1.1.0). Kubernetes manifests generally do not need to set `runAsUser`
solely to translate the old user name. Align writable volume ownership with UID
`12345`.

## Pin Docker release lines intentionally

In v2, prereleases and maintenance releases from older majors no longer update
Docker `:latest` or the GitHub latest-release marker (2.0.0). Floating major
tags such as `grafana/k6:v1` track a selected major line. Use a major tag when
accepting updates within one major, or an exact tag for fully reproducible
runs.

## Audit deprecations before they become removals

- Summary `--no-summary` and `K6_NO_SUMMARY` are deprecated; use
  `--summary-mode=disabled` (1.3.0).
- Summary mode `legacy` is deprecated; move to `compact` or `full` (1.3.0).
- `experimental-opentelemetry` and `K6_OTEL_EXPORTER_TYPE` are deprecated; use
  `opentelemetry` and `K6_OTEL_EXPORTER_PROTOCOL` (1.4.0).
- `k6/experimental/redis` is deprecated in favor of the official Redis
  extension (1.5.0).
- `k6/experimental/websockets` is deprecated; use `k6/websockets` (1.6.0).
- Replace First Input Delay integrations with Interaction to Next Paint;
  `browser_web_vital_fid` was planned for v2 removal (1.3.0).

## Post-upgrade verification

After changing versions:

1. Run `k6 deps --json` on both source and archives.
2. Exercise Cloud cancellation and threshold-abort paths and assert their
   distinct exit codes.
3. Verify Rate consumers handle the labeled single-counter representation.
4. Confirm controllers explicitly enable the HTTP API.
5. Run browser redirect tests and remove duplicate-metric workarounds.
6. Check container volume permissions against UID `12345`.
