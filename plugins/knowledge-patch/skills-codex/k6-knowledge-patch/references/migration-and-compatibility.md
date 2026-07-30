# Migration and compatibility

## Major-version contract

Since 1.0.0, k6 follows Semantic Versioning: breaking changes are reserved for
major releases and receive advance deprecation warnings. Each major receives
critical fixes for at least two years, and the supported public API surface is
explicitly delineated for extensions and integrations.

## v2 migration checklist

These changes take effect in 2.0.0:

- The Go module path is `go.k6.io/k6/v2`. Update every k6 import in extensions
  and external Go packages, for example:

  ```go
  import "go.k6.io/k6/v2/js/modules"
  ```

- The `externally-controlled` executor and `k6 pause`, `resume`, `scale`, and
  `status` commands are removed without replacements. A script using that
  executor will not start; choose another executor such as `ramping-vus`,
  `constant-vus`, or `constant-arrival-rate`.
- The top-level `k6 login`, positional `k6 cloud script.js`, and
  `--upload-only` are removed. Use `k6 cloud login`, `k6 cloud run script.js`,
  and `k6 cloud upload script.js`. Supply InfluxDB credentials through
  `K6_INFLUXDB_*` rather than `k6 login influxdb`.
- `options.ext.loadimpact` is rejected. Move its fields to `options.cloud`.
- `K6_OTEL_SINGLE_COUNTER_FOR_RATE` is removed; the single-counter Rate shape
  can no longer be postponed.
- `K6_BINARY_PROVISIONING` and `K6_ENABLE_COMMUNITY_EXTENSIONS` are removed.
  Community extensions use the default build service.
- The old `{USER_CONFIG_DIR}/loadimpact/config.json` path is no longer read or
  migrated. Move the file to `{USER_CONFIG_DIR}/k6/config.json` or regenerate
  it with `k6 cloud login`.
- The HTTP API no longer listens on `localhost:6565` by default. Enable it
  explicitly with `--address` or `K6_ADDRESS`.
- Public k6 Go types no longer provide easyjson-generated `MarshalJSON` and
  `UnmarshalJSON` methods. Use standard `encoding/json`.
- A Cloud run aborted by the system, a limit, a script error, the user, or a
  timeout exits `97`, not `0`. Success remains `0`; threshold aborts remain
  `99`.
- The web dashboard is built into k6; use `k6 run --out=web-dashboard` instead
  of a separate xk6-dashboard extension.

## Configuration path transition

In 1.0.0-rc1, `k6 cloud login` and the then-deprecated `k6 login` began
creating `{USER_CONFIG_DIR}/k6/config.json` instead of
`{USER_CONFIG_DIR}/loadimpact/config.json`. Login migrated the file, while
`k6 run` checked the new path first, fell back to the old path, and warned.
That compatibility path ends in 2.0.0 as described above.

## Deprecation replacements

- In 1.0.0-rc1, global `crypto` became stable and
  `k6/experimental/webcrypto` was deprecated for planned removal in 1.1.0.
- In 1.3.0, `--no-summary` and `K6_NO_SUMMARY` were deprecated in favor of
  `--summary-mode=disabled` and `K6_SUMMARY_MODE=disabled`.
- In 1.3.0, `legacy` summary mode was deprecated for removal in 2.0.0. Use
  `compact` or `full`.
- In 1.3.0, First Input Delay began its removal path. Replace
  `browser_web_vital_fid` with Interaction to Next Paint,
  `browser_web_vital_inp`.
- In 1.4.0, the stable output name became `opentelemetry`;
  `experimental-opentelemetry` remained an alias but was deprecated.
  `K6_OTEL_EXPORTER_TYPE` was replaced by `K6_OTEL_EXPORTER_PROTOCOL`.
- In 1.5.0, `k6/experimental/redis` was deprecated. Use the official Redis
  extension.
- In 1.6.0, WebSockets became stable at `k6/websockets`;
  `k6/experimental/websockets` was deprecated with the same API.

## Go build requirements

The minimum Go version changes with the k6 version:

| k6 batch | Build requirement |
| --- | --- |
| 1.0.0-rc1 | Go 1.23 or newer; release builds used Go 1.24 |
| 1.4.0 | Go 1.24 or newer |
| 1.7.0 | Go 1.25 or newer; the default toolchain was Go 1.26 |

Use the requirement associated with the k6 source tree being built.

## Containers and release tags

- In 1.0.0-rc1, release candidates also updated Docker `latest`. Pin an
  explicit image tag when prereleases are unacceptable.
- In 1.1.0, the image switched from a named `k6` user to numeric UID `12345`,
  avoiding a required `runAsUser` override in Kubernetes manifests.
- In 2.0.0, prereleases and maintenance releases on older major lines stopped
  updating Docker `:latest` and the GitHub latest-release marker. Floating
  `:vN` tags such as `grafana/k6:v1` track a selected major line.
