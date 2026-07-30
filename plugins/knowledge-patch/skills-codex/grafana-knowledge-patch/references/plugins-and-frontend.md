# Plugins and Frontend Development

Use this reference when building, packaging, installing, or upgrading Grafana
plugins and when migrating frontend code across Grafana UI API changes.

## Installation and compatibility

- Grafana prevents installation of plugin versions whose Angular compatibility
  is unsupported and can disable version installation for selected plugin
  types (since 11.5.0).
- The frontend toolchain moves to Node 22, and `react-router-dom` is again a
  Grafana UI dependency for plugin development (since 11.5.0).
- `plugins update-all` no longer uninstalls before updating (since 11.6.0).
- `grafana cli plugins install` enforces the plugin's `grafanaDependency`
  during the `12.0-upgrade`. It offers no compatibility-bypass option; use the
  deliberate ZIP path when an incompatible install is truly required.
- `pluginsSriChecks` is generally available from 12.0.0.
- Grafana 12 removes Angular frontend support. Migrate Angular plugins and
  extensions to supported frontend APIs.
- Grafana 12 removes plugin dependency-version support and secrets-manager
  plugin support.
- Before the `13.0-upgrade` to React 19, update the current Grafana line to its
  latest patch, then update and validate every installed plugin.

## Runtime registration and context

- Apps can register data sources at runtime rather than depending solely on
  static installation (since 11.5.0).
- `usePluginContext` exposes its `PluginMeta` generic (since 12.1.0).
- Data-source queries include dashboard and panel title headers (since
  12.1.0).
- Data-source configuration components can be registered by plugin extensions
  (since 12.2.0).
- Plugins receive `PluginContext` even when Scenes is disabled (since 12.3.0).
  Azure SSO settings are exposed through the context.
- Frontend API clients are packaged, cover all endpoints, provide regular and
  lazy hooks, and automatically set required PATCH headers (since 12.3.0).
- New asynchronous APIs and hooks replace `datasourceSrv` (since 13.1.0).
  Migrate frontend data-source access to the async pattern.

## Extension APIs

- Extensions can share functions as well as components, and components
  returned from `usePluginComponents` expose registry metadata (since 11.6.0).
- Observable APIs expose registered components and links, while deprecated
  extension APIs are removed (since 12.0.0).
- Grafana stops path-validating UI link extensions in 12.3.0, changing which
  links can register. Validate target paths within the plugin.
- Link extensions add `openInNewTab` (since 12.4.0).

## Plugin process isolation

From 12.4.0, plugin processes no longer inherit host environment variables by
default. Pass required configuration explicitly. External AWS plugins continue
to receive AWS SDK credential-chain variables. `PLUGIN_UNIX_SOCKET_DIR` is
available for deployments with restricted temporary directories.

Community plugins and Enterprise community/PPT plugins can opt into
experimental sandboxing (since 12.4.0).

## Manifest changes

- `plugin.json` routes must include `routes[].path` (since 12.4.0).
- Every entry in the `plugin.json` `includes` array must declare `type` (since
  13.0.0).
- Plugins can share `react-dom/client` and `react-dom/server` (since 13.0.0).

## Grafana UI component migrations

### From 11.6.0

- `Select` is deprecated; migrate to `Combobox`.
- `MultiCombobox` and `UsersIndicator` are exported from `@grafana/ui`.
- `InlineField.error` accepts a `React.ReactNode`.
- `WeekStart` is typed as `WeekStart | undefined`, not an arbitrary string.

### From 12.0.0

- `Combobox` supports grouping.
- `Stack` and `Grid` expose `columnGap` and `rowGap`.

### From 12.3.0

- `Collapse.collapsible` is deprecated.
- `Slider.inputId` is required.
- Slider has a control for whether its input is shown.

### From 12.4.0

- Slider supports decimal values.
- A childless `ToolbarButton` must define `tooltip` or `aria-label`.
- `InteractiveTable` adds `disableSortRemove` and `sortDescFirst`.
- `InteractiveTable` resets pagination after data changes only when
  `autoResetPage` requests it.

### From 13.0.0

- The Gauge visualization is generally available, but the `Gauge` component
  is removed from `@grafana/ui`.
- `Combobox` uses `isItemDisabled`.
- Deprecated `Modal` props are removed.
- `SeriesIcon.noMargin` is removed; no-margin behavior is the default.
- The Graph graveyard API is deleted.

## Logs and observability extensions

- Expressions work with plugins declaring `backend: true` and `alerting:
  false` (since 12.0.0).
- Plugin-request metrics include the plugin version (since 12.2.0).
- Dashboard Logs panels accept a custom plugin log grammar (since 13.0.0).
  OpenTelemetry log formatting supports dot-separated label names.
- Plugin rule origin is propagated through `X-Rule-Origin` (since 13.1.0).

## Packaged and removed core integrations

- Metrics Drilldown's external-app implementation is generally available and
  legacy paths are removed; Traces Drilldown is preinstalled (since 12.0.0).
- The core Elasticsearch data source is removed in 13.0.0. Package a supported
  replacement explicitly.
- Core Prometheus removes Azure and SigV4 authentication, and the
  `grafana-prometheus` package is removed (since 13.1.0).
- Zipkin is removed from the core data-source plugin set (since 13.1.0).

## Deprecated boot and folder APIs

`GrafanaBootData.config.apps`, `GrafanaBootData.config.panels`, and
`getFolderByUID` are deprecated from 12.4.0. Library Elements deprecates
`folderFilter` in favor of `folderFilterUIDs`.

## Upgrade validation

For each plugin:

1. Check `grafanaDependency`, signature/SRI state, Angular usage, manifest
   routes, `includes` types, and any removed core dependency.
2. Run its frontend build on Node 22 and resolve React 19 and UI component
   changes.
3. Exercise extension registration, async data-source access, PATCH requests,
   links, plugin context, and disabled-Scenes behavior.
4. Run backend code without inherited host environment variables and validate
   sockets, AWS credential handling, TLS, and alert expressions.
5. Verify dashboards, logs grammar, metrics, and accessibility labels.
