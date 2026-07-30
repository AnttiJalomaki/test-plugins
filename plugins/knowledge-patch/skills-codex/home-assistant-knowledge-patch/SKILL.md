---
name: home-assistant-knowledge-patch
description: Home Assistant
version: 2026.7
license: MIT
metadata:
  author: Nevaberry
---


# Home Assistant Knowledge Patch

Use this skill when changing, reviewing, or troubleshooting Home Assistant
configuration, automations, dashboards, integrations, installation, or custom
components. Start with the migration checks below, then open the reference file
whose topic matches the work. Treat the running instance, configuration, traces,
diagnostics, and tests as authoritative when they differ from general guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [Breaking changes and migrations](references/breaking-changes.md) | Removed actions and entities, state and unit changes, new minimum versions, authentication, and polling changes |
| [Automations, scripts, and templates](references/automation-templates.md) | Purpose-specific building blocks, action semantics, variable scope, template entities, helpers, selectors, and editor behavior |
| [Dashboards and user interface](references/dashboards-ui.md) | Built-in dashboards, cards, layout, search, pickers, graphs, Activity, editors, and device-management pages |
| [Backups, installation, and system operations](references/backup-system.md) | Backup configuration and destinations, supported installations, updates, runtime, logs, Apps, and storage |
| [Assist, voice, and AI](references/assist-ai.md) | Satellites, intents, conversations, AI Task, speech, provider tools, and diagnostics |
| [Integrations, devices, and protocols](references/integrations-devices.md) | Integration additions, device capabilities, protocol support, sensors, actions, setup, and service-specific behavior |
| [Custom integration and frontend development](references/integration-development.md) | Entity and config-entry APIs, discovery models, service registration, selectors, frontend APIs, and developer tooling |

## Breaking changes first

### Installation and runtime

- Do not plan upgrades for Core, Supervised, `i386`, `armhf`, or `armv7`.
  They stopped receiving updates, including security updates, in 2025.12.
  Use Home Assistant OS or Container on a supported architecture.
- Home Assistant uses Python 3.14 as of 2026.3. Official installation methods
  upgrade it automatically; custom tooling and custom integrations must be
  compatible with that runtime.
- Container images use `zstd` compression as of 2026.3. Require Docker 23.0.0+
  or containerd 1.5.0+, unless the installed older runtime explicitly supports
  `zstd` images.
- A Container install without host networking creates a Repair issue. Check the
  deployment network mode before diagnosing discovery or reachability failures.
- Home Assistant OS calls Supervisor-managed sidecar software **Apps**. Older
  documentation and action domains may still use “add-on”; do not confuse Apps
  with integrations.

### YAML, actions, and state models

- Light color temperature is Kelvin-only as of 2026.3. Replace action field
  `color_temp` with `color_temp_kelvin`, and replace `color_temp`, `kelvin`,
  `min_mireds`, and `max_mireds` attributes with the Kelvin attributes.
- Legacy template entities under individual platform keys were removed in
  2026.6. Put them under modern `template:` configuration.
- MQTT publishing takes templates directly in `topic` and `payload`; remove
  `topic_template` and `payload_template`. An omitted payload publishes an
  empty payload. MQTT entity-ID suggestions use `default_entity_id`, not
  `object_id`.
- Google Calendar no longer has `add_event`; use entity action `create_event`.
  Plex no longer has `scan_for_clients`; press the **Scan Clients** button
  entity. Velux gateway reboot likewise moved to its button entity.
- Supervisor actions now raise on failure and stop a sequence. Add
  `continue_on_error: true` only where continuing is intentional.
- Webhook `local_only` accepts only YAML booleans `true` and `false`, and
  Telegram Bot actions reject undefined or unused fields.
- Template binary sensors and fans that return `None` now become `unknown`
  instead of `off`. Return `False` when `off` is intended. A template fan state
  syntax error yields `unavailable`; a percentage syntax error yields `None`.
- Media players that formerly used `standby` may now report `off`; the platform
  `STANDBY` state is deprecated. Audit exact state comparisons.
- Vacuum battery properties and several device-tracker `battery_level`
  attributes moved to dedicated battery sensors. Update templates, cards,
  automations, and scripts to target the sensor entities.

### Purpose-specific automations

- Purpose-specific triggers and conditions are the default editor entry point.
  Generic state blocks, templates, YAML, and existing automations remain valid.
- Preview keys renamed before graduation no longer work. Notable replacements
  include `battery.low` → `battery.became_low`, `schedule.turned_on` →
  `schedule.block_started`, `timer.time_remaining` →
  `timer.remaining_time_reached`, and `vacuum.docked` →
  `vacuum.returned_to_dock`.
- Trigger behavior values changed from `any` to `each` and `last` to `all`,
  with `each` now the default. Reselect an affected block or edit its YAML.
- Zone-oriented person and device-tracker triggers supersede the withdrawn
  home-only preview blocks. They support entering, leaving, occupancy, empty
  state, and `for` durations.

## Backups and restore

- The first backup setup creates a mandatory encryption key and configures
  frequency and retention. Preserve the emergency kit: encrypted backups
  cannot be restored without the key.
- Encryption is configurable per destination except Home Assistant Cloud,
  which stays encrypted. Downloads through Home Assistant are decrypted on the
  fly. Restore works across all installation methods and during Cloud-backed
  onboarding.
- Automatic backup schedules can choose a time, weekdays, and per-location
  retention. Advanced automations call `backup.create_automatic`.
- Automatic retention does not remove manual backups. Pre-update backups keep
  one automatic backup per add-on/App, and a restart waits for an active backup.
- Backup agents include Cloud and numerous integration-provided locations.
  Inspect [the system reference](references/backup-system.md) before choosing a
  destination because capabilities, patch fixes, and upload-progress support
  differ by provider.
- Treat incomplete backups as failures even when an archive exists. Repair
  issues report skipped add-ons or folders; the backup UI can separately show
  creation and per-location upload progress.

## Automations, scripts, and templates

- Nested `variables` actions update an existing outer variable; a new name is
  created in the top-level run scope. `wait` and `response_variable` values
  also propagate outward, including from parallel sequences. Avoid designs
  that rely on local shadowing.
- Time triggers support datetime-helper offsets, selected weekdays, and
  templated webhook IDs. Purpose-specific state blocks offer visual `for`
  durations and domain-aware target summaries.
- Action targets by label now include labeled configuration and diagnostic
  entities. Inspect the expanded target list before using broad floor, area,
  device, or label targets.
- Modern template YAML supports lights, switches, fans, locks, covers, alarm
  panels, vacuums, sensors, events, updates, and device trackers with the
  platform-specific capabilities detailed in the automation reference.
- Useful template additions include set operations, hashes, `clamp`, `wrap`,
  `remap`, `from_hex`, translated attribute lookup, and entity, device, floor,
  area, and label helpers. Prefer `entity_name()` over reading
  `friendly_name` directly.
- The UI editors provide YAML linting, Jinja/YAML autocomplete, hover values,
  notes, traces with template errors, copy/paste, undo/redo, and visual
  continue-on-error controls. Keep exported YAML valid even when authoring in
  the visual editor.

## Dashboards and UI

- The built-in **Overview** is the default for new installations; the former
  customizable dashboard is **Overview (legacy)**. Existing user dashboard
  preferences and system-wide defaults have different scopes.
- Home, Maintenance, Security, Lights, Climate, and protocol dashboards are
  built-in experiences with availability based on configuration. Existing
  Areas dashboards continue to work, but new ones cannot be created.
- Sections dashboards support headers, backgrounds, sticky footers, auto-height
  cards, a 24-pixel default row gap, prediction strategies, and many Tile-card
  features. Themes can restore the old gap with
  `ha-view-sections-row-gap: 8px`.
- Energy configuration supports parent/child devices, live power sensors,
  signed or paired flow sensors, battery state of charge, downstream water
  meters, resource tabs, live flows, and period totals.
- Use `Ctrl+K` or `Cmd+K` for unified Quick search. The entity-first card picker,
  context-rich target picker, and related-object links reduce reliance on raw
  entity IDs while preserving access to YAML.
- Activity is the renamed and redesigned Logbook interface. OS Core logs are
  normally available under **Settings > System > Logs** and `ha core logs`,
  not as a duplicated configuration-folder file.

## Assist, voice, and AI

- `assist_satellite.start_conversation` starts an LLM-backed conversation;
  `assist_satellite.ask_question` accepts local sentence patterns and returns
  the matched answer ID and slots. Use `response_variable` for later choices.
- Satellite broadcasts, thermostat targeting, shopping-list completion,
  mapped-room vacuum cleaning, media volume, fan speed, and lawn-mower control
  depend on language, device, and agent support.
- Conversation agents can stream responses, use exposed calendar or to-do
  entities, continue listening after a question, and share fallback history.
  Audit entity exposure and tool calls in the voice debug interface.
- `ai_task.generate_data` returns text or selector-defined structured data and
  can receive files or camera media. `ai_task.generate_image` returns the
  generated asset in the response variable's `url` field.
- Selecting a default AI Task entity lets calls omit `entity_id`. AI suggestions
  send the full automation or script plus relevant names to that configured
  provider, so review the data boundary before enabling them.
- Home Assistant can consume tools from MCP servers and expose Home Assistant
  context to external MCP clients. Keep tool permissions and Assist exposure
  narrow enough for the intended agent.

## Integration work

- Before changing an integration, search both integration references: use
  [integrations and devices](references/integrations-devices.md) for available
  capabilities and [breaking changes](references/breaking-changes.md) for
  renamed values, removed entities, new credentials, and minimum server or
  firmware versions.
- Exact state comparisons are especially fragile. Many integrations moved from
  title case or vendor codes to lowercase snake-case machine values; update
  action data, templates, and conditions together.
- When an attribute or control disappears, look for a dedicated sensor, binary
  sensor, event, select, number, button, update, or valve entity before creating
  a template replacement.
- Polling intervals increasingly use Home Assistant's integration-independent
  polling customization. Do not assume a removed integration option means the
  integration stopped polling.
- Device and sub-device migrations can preserve entity IDs while changing
  device targets. Re-audit device-targeted automations after ESPHome, Shelly,
  Reolink Duo, or FRITZ!SmartHome registry changes.
- Protocol integrations carry explicit external requirements. Check Z-Wave JS,
  UniFi Protect, ESPHome, Pi-hole, pyLoad, Sentry, Mealie, BSB-LAN, and container
  runtime versions before diagnosing setup or reauthentication failures.

## Custom integration checks

- Review the development reference before updating a custom component. Recent
  changes touch config subentries, backup agents, discovery imports, config-flow
  results, entity descriptions, service registration, target selectors, update
  coordinators, OAuth helpers, device trackers, and frontend components.
- Treat `UnitSystem` as immutable. Stop reading the removed `FlowResult.result`
  key and deprecated `DeviceEntry.suggested_area` attribute.
- Discovery `ServiceInfo` models moved for DHCP, SSDP, USB, and zeroconf. Update
  imports rather than preserving compatibility shims indefinitely.
- Platform entity service registration changed and service helpers deprecate
  the `hass` argument. Service translations are no longer returned from the
  generic service-list endpoints.
- Custom serial integrations must migrate from `pyserial` assumptions to the
  asynchronous `serialx` path used by network-capable serial proxies.
- Validate against a real Home Assistant test instance and run the component's
  tests after every API migration; platform behavior and registry state take
  precedence over stale examples.

## Working method

1. Identify the installation type, Home Assistant release, affected integration,
   and external device, server, firmware, or app versions.
2. Read the breaking-change entry for every exact state, attribute, entity,
   action, option, credential, or unit involved.
3. Read the matching feature reference and confirm prerequisites and disabled-
   by-default entities.
4. Search automations, scripts, templates, dashboards, statistics, and exports
   for old identifiers and exact value comparisons.
5. Apply the smallest migration, reload when supported, and inspect Repairs,
   traces, logs, diagnostics, entity registry, and device targets.
6. Exercise both success and failure paths, especially backup, authentication,
   broad targets, response actions, and `continue_on_error` behavior.
