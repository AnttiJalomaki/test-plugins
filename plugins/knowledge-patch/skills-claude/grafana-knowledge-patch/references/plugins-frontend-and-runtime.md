# Plugins, frontend APIs, and runtime

Use this reference for plugin installation and manifests, frontend components
and extensions, Image Renderer, process isolation, server behavior, containers,
and derived deployment images.

## Plugin installation and compatibility

- In 11.5.0, Grafana blocks installation of a plugin version whose Angular
  version is unsupported. Administrators can also disable version installation
  for selected plugin types.
- In 11.6.0, `grafana cli plugins update-all` no longer performs a distinct
  uninstall step. Automation must not depend on that intermediate state.
- During the 12.0-upgrade, `grafana cli plugins install` enforces
  `grafanaDependency` compatibility. There is no compatibility bypass; use ZIP
  installation when deliberately installing an incompatible plugin.
- In 12.0.0, Angular frontend support is removed, as are plugin
  dependency-version support and secrets-manager plugins.
- During the 13.0-upgrade, Grafana moves to React 19. Update the current Grafana
  release line to its latest patch, update and validate every installed plugin,
  and then upgrade Grafana.

## Build and shared dependencies

- In 11.5.0, the frontend toolchain moves to Node 22.
  `react-router-dom` returns as a Grafana UI dependency for plugin development.
- In 13.0.0, plugins may share `react-dom/client` and `react-dom/server`.

## Extension APIs

- In 11.6.0, extensions can share functions as well as components. Components
  returned by `usePluginComponents` expose their registry metadata.
- In 12.0.0, observable APIs can access registered components and links, while
  deprecated extension APIs are removed.
- In 12.2.0, extensions can register data-source configuration components.
- In 12.3.0, plugin context is supplied even when Scenes is disabled. Azure SSO
  settings are exposed through context. UI link extensions no longer undergo
  Grafana path validation, changing which links can register.
- In 12.4.0, link extensions add `openInNewTab`.
- In 13.1.0, new asynchronous data-source APIs and hooks replace
  `datasourceSrv`.

## Plugin context and packaged clients

- In 12.1.0, `usePluginContext` exposes its `PluginMeta` generic.
- In 12.3.0, packaged frontend API clients cover all endpoints, expose regular
  and lazy hooks, and set required PATCH headers automatically.

## Grafana UI component changes

- In 11.6.0, `Select` is deprecated in favor of `Combobox`.
  `MultiCombobox` and `UsersIndicator` are exported from `@grafana/ui`.
  `InlineField.error` accepts `React.ReactNode`.
- In 12.0.0, `Combobox` supports grouping. `Stack` and `Grid` expose
  `columnGap` and `rowGap`.
- In 12.3.0, `Collapse.collapsible` is deprecated. `Slider.inputId` is required,
  and Slider can control whether its input is shown.
- In 12.4.0, Slider supports decimals. A `ToolbarButton` without children must
  provide `tooltip` or `aria-label`. `InteractiveTable` adds
  `disableSortRemove` and `sortDescFirst`; it resets pagination for data changes
  only when `autoResetPage` requests it.
- In 13.0.0, the Gauge visualization is generally available, but the `Gauge`
  component is removed from `@grafana/ui`. `Combobox` moves to
  `isItemDisabled`; deprecated `Modal` props and `SeriesIcon.noMargin` are
  removed, with no-margin behavior becoming default. The Graph graveyard API
  is deleted.

## Plugin manifests

- In 12.4.0, every plugin `routes[]` manifest entry must declare `path`.
- In 13.0.0, every `plugin.json` `includes[]` entry must declare `type`.

## Process environment and sandboxing

- In 12.4.0, plugin processes stop receiving host environment variables by
  default. Pass required values deliberately.
- External AWS plugins continue to receive AWS SDK credential-chain variables.
  Plugin processes receive `PLUGIN_UNIX_SOCKET_DIR` for installations that
  restrict temporary directories.
- In 12.4.0, community plugins, plus Enterprise community and PPT plugins, can
  use an experimental sandbox mode.

## Image Renderer

- In 11.6.0, plugin-mode Image Renderer supports SSL.
- From 12.3.5, Image Renderer supports custom CA certificates for private PKI.
- During the 13.0-upgrade, plugin-mode renderer support is removed; deploy the
  renderer as a separate service.
- `renderAuthJWT` is enabled by default for the 13.0-upgrade. Configure the same
  nonempty, non-`-` token on Grafana and renderer:

```ini
[rendering]
renderer_token = replace-with-a-shared-secret
```

- To temporarily restore database-backed opaque authentication, set
  `renderAuthJWT = false` under `[feature_toggles]`.

## Server and network runtime

- In 12.4.0, short URLs default to never expiring. Grafana HTTP telemetry uses
  native histograms by default, while classic histograms remain configurable.
  Grafana Live adds `client_queue_max_size`.
- In 13.0.0, `server.enable_gzip` defaults to `true`. Disable it explicitly if a
  proxy or another layer should own compression.
- In 13.0.0, the server can serve HTTPS and HTTP/2 over a Unix domain socket and
  can listen on TCP and Unix sockets simultaneously. Redis remote cache accepts
  `network=unix`.
- In 13.0.0, Grafana Live push is protected by RBAC.

## Commands

- During the 13.0-upgrade, `grafana-cli` and `grafana-server` are removed.
  Update services, images, CI, and scripts to use `grafana cli` and
  `grafana server`.
- In 12.4.0, `grafana cli admin flush-rbac-seed-assignment` flushes seeded RBAC
  assignments.

## Container and libc assumptions

- In 11.6.0, Docker builds use Grafana-supplied glibc 2.40 binaries.
- In 12.2.0, `grafana/grafana-oss` is deprecated; use `grafana/grafana`.
- From 12.3.8, Alpine images use Alpine 3.24.1.
- In 13.0.0, the Ubuntu base changes from 22.04 to 24.04.

Rebuild and test derived images, native plugin dependencies, package
installation, certificates, and runtime assumptions after each base change.

## Plugin-related metrics and request context

- In 12.1.0, data-source requests include dashboard and panel titles as
  headers.
- In 12.2.0, plugin request metrics include the plugin version.
- In 13.0.0, plugins can provide a custom Logs grammar.
