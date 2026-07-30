# Custom Integration Development

## Config entries, discovery, and flows

### Custom-integration discovery and units

*Source batch: 2025.2.*

Developer-facing changes add energy-by-distance units and relocate the DHCP, SSDP, USB, and zeroconf `ServiceInfo` models, requiring custom integrations that use those discovery models to update their imports.

### Custom-integration flow results

*Source batch: 2025.8.*

The `result` attribute has been removed from the `FlowResult` typed dictionary; custom integrations must stop reading it.

## Entity models, services, and coordinators

### Advanced mode and developer interfaces

*Source batch: 2026.6.*

The profile-level Advanced mode toggle is removed and its formerly gated features are available to everyone; advanced-mode checks in data-entry flows are correspondingly deprecated. Developer changes also require a domain on `BrowseMediaSource`, deprecate combining config-entry listeners with reload methods, and add an entity-name formatting helper.

### Custom-integration API changes

*Source batch: 2025.1.*

Developer-facing changes include a `WaterHeaterEntityDescription` rename and migration toward Pydantic v2, plus independent horizontal swing for climate entities, a new vacuum state property, Kelvin as the preferred color-temperature unit, and an area device class for squared units.

### Custom-integration API changes

*Source batch: 2025.12.*

MQTT subscriptions gain a status callback, data update coordinators support Retry After, service-action translation descriptions support placeholders, and device-identification buttons are classified as diagnostic. Worker-thread serialization for `Store` data is now opt-in, and `CalculatedState.capability_attributes` is removed.

### Custom-integration API changes

*Source batch: 2025.3.*

Developer-facing changes introduce backup agents and config subentries, change `BackupAgent` APIs and config-entry state transitions, and add checks for config-flow unique IDs.

### Custom-integration development tooling

*Source batch: 2026.2.*

Home Assistant's custom-integration development workflow is replacing `pre-commit` with `prek`.

### Custom-integration device areas

*Source batch: 2025.9.*

`DeviceEntry.suggested_area` is deprecated and will be removed; custom integrations must stop relying on that attribute.

### Custom-integration interface changes

*Source batch: 2025.11.*

Target selectors no longer support the device filter, and service translations are no longer returned by WebSocket `get_services` or REST `/api/services`. `TemperatureConverter.convert_interval` is deprecated, while update coordinators now support retriggering.

### Custom-integration interfaces

*Source batch: 2026.3.*

Developer-facing changes deprecate Labs `async_listen`, change OAuth 2.0 helper error handling, allow custom integrations to provide brand images, and add reconfiguration support to the webhook helper. Deprecated light features have also been removed.

### Custom-integration interfaces

*Source batch: 2026.7.*

Developer-facing changes revise device-tracker entity models and frontend components, introduce new unit enumerators, and deprecate the `home_assistant_start` flag of `async_initialize_triggers`.

### Custom-integration service APIs

*Source batch: 2025.10.*

The `hass` argument in service helpers is deprecated, and platform entity services have a revised registration API; custom integrations using either interface need migration.

## Frontend and metadata interfaces

### Custom dashboard strategies

*Source batch: 2026.5.*

Custom integrations and frontend modules can now register dashboard strategies that generate complete dashboards dynamically.

### Custom-integration display metadata

*Source batch: 2025.6.*

Icon translations can now define ranges, and sensor device classes now provide default display precision.

## Deprecations and migration work

### Custom-integration deprecations

*Source batch: 2026.5.*

Developer-facing changes deprecate the legacy device-tracker platform API and entity IDs whose domains do not match their platforms, migrate App builds to Docker BuildKit, and standardize doorbell event types. Frontend components and context APIs also change in this release.
