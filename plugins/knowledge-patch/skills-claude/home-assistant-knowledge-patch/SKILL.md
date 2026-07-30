---
name: home-assistant-knowledge-patch
description: Home Assistant
version: 2026.7
license: MIT
metadata:
  author: Nevaberry
---


# Home Assistant Knowledge Patch

Load this skill when configuring, automating, extending, upgrading, or
troubleshooting Home Assistant. Start with the migration checks below when an
upgrade changes entity state, action data, YAML validation, device targeting,
or installation support. Open the topic reference that matches the task before
writing configuration or custom-integration code.

## Reference index

| Reference | Topics |
| --- | --- |
| [Automations, Templates, Assist, and AI](references/automations-templates-assist.md) | Automation and script behavior, purpose-specific triggers and conditions, templates, voice, conversation agents, AI Task |
| [Backups, System Operations, and Installation](references/backups-system-installation.md) | Backup encryption, scheduling, storage, restore, supported installations, runtime and operating-system behavior |
| [Custom Integration Development](references/custom-integration-development.md) | Config flows, discovery, entity APIs, services, coordinators, frontend metadata, deprecations |
| [Dashboards, Cards, and Energy](references/dashboards-cards-energy.md) | Built-in dashboards, cards, editor workflows, search, Activity, energy, water, gas, and live power |
| [Integrations, Devices, and Services](references/integrations-devices-services.md) | New integrations, device and protocol capabilities, actions, entities, sensors, and discovery |
| [Migrations and Breaking Changes](references/migrations-breaking-changes.md) | Removed integrations and APIs, state and unit migrations, authentication, minimum versions, polling, validation, patch corrections |

## Upgrade triage

Before changing an automation that stopped working:

1. Check whether the entity or action was removed or replaced.
2. Check exact state, preset, fan-mode, event, unit, and attribute comparisons.
3. Check whether a device was reorganized into sub-devices and therefore has a
   different device target.
4. Check the integration's minimum server, firmware, or API version.
5. Check whether polling moved to Home Assistant's integration-independent
   polling controls.
6. Check whether stricter validation now rejects formerly tolerated action
   fields or YAML values.
7. Read the relevant patch-release note before assuming the major release's
   first behavior is still current.

## High-impact breaking changes

### Supported installations and runtime

- Home Assistant Core and Supervised installations, plus `i386`, `armhf`, and
  `armv7`, no longer receive updates. Migrate to Home Assistant OS or Container
  on a supported architecture.
- Container images use `zstd`; use Docker 23.0.0 or newer, containerd 1.5.0 or
  newer, or another runtime that explicitly supports these images.
- Home Assistant runs on Python 3.14. Supported installation methods perform
  the runtime transition during a normal update.
- A Container installation without host networking raises a Repair issue.

### YAML and template migrations

- Legacy template platforms under `alarm_control_panel`, `binary_sensor`,
  `cover`, `fan`, `light`, `lock`, `sensor`, `switch`, `vacuum`, and `weather`
  are removed. Put them under modern `template:` configuration.
- Template binary sensors and fans returning `None` now become `unknown`, not
  `off`. Return `False` explicitly when `off` is intended.
- A template fan state-template syntax error makes the entity `unavailable`;
  a percentage-template syntax error produces `None` rather than `0`.
- `issues()` returns active issues only.

### Action and schema migrations

- Light actions are Kelvin-only. Use `color_temp_kelvin`,
  `min_color_temp_kelvin`, and `max_color_temp_kelvin`; mired-based fields and
  attributes are removed.
- `mqtt.publish` templates belong directly in `topic` and `payload`;
  `topic_template` and `payload_template` are removed. An omitted payload sends
  an empty payload.
- MQTT `object_id` is removed from YAML and ignored in discovery. Use
  `default_entity_id` to suggest an ID.
- Webhook `local_only` accepts only the booleans `true` and `false`.
- Supervisor actions now raise on failure and stop the sequence. Apply
  `continue_on_error: true` to a step only when continuing is deliberate.
- Google Calendar event creation uses the entity-based `create_event` action;
  the old integration action is removed.
- Plex client discovery and Velux gateway reboot use their replacement button
  entities; the old actions are removed.

### Entity, state, and target migrations

- Media players in several integrations now report `off` instead of
  `standby`; the platform `STANDBY` state is deprecated.
- Vacuum battery properties and device-tracker `battery_level` attributes have
  moved to dedicated sensors.
- Reolink dual-lens cameras create one sub-device per lens. Entity IDs remain,
  but device-targeted automations must target the lens sub-device.
- Shelly multi-channel hardware and ESPHome gateways can expose sub-devices;
  audit device targets after migration.
- Person and zone membership uses `in_zones`; overlapping zones can each count
  a person, and position-aware trackers choose the smallest containing zone.
- Label-targeted actions include labeled configuration and diagnostic entities,
  so audit broad label targets before running mutating actions.

### Purpose-specific automation keys

Purpose-specific triggers and conditions are the editor default. Existing YAML
and generic state logic remain valid, but old preview keys do not. Important
renames include:

```text
battery.low                    -> battery.became_low
battery.not_low                -> battery.no_longer_low
lawn_mower.docked              -> lawn_mower.returned_to_dock
schedule.turned_off            -> schedule.block_ended
schedule.turned_on             -> schedule.block_started
timer.time_remaining           -> timer.remaining_time_reached
update.update_became_available -> update.became_available
vacuum.docked                  -> vacuum.returned_to_dock
climate.target_humidity        -> climate.is_target_humidity
climate.target_temperature     -> climate.is_target_temperature
```

Preview behavior values also changed from `any` to `each` and from `last` to
`all`, with `each` as the default. Reselect and save affected editor blocks or
update their YAML.

## Backup quick reference

- Backups use a mandatory generated encryption key. Preserve the emergency kit;
  restoration requires the key.
- Encryption can be disabled per destination except Home Assistant Cloud.
  Downloads made through Home Assistant are delivered decrypted.
- Automatic schedules support a chosen time, weekdays, and
  `backup.create_automatic`; retention is configured per location.
- Automatic retention never deletes manual backups. An update backup is kept
  separately per App, and an incomplete backup raises a Repair issue.
- Restore works across supported installation methods and can read local,
  Cloud, or integration-provided locations.
- Upgrade and restart flows wait for configured backups; the backup page can
  show per-location upload progress where the agent supports it.
- Available destinations include major object stores, file-transfer services,
  cloud drives, NAS providers, and integration-provided agents. Consult the
  backup reference before selecting an agent or relying on multipart behavior.

## Automation and script quick reference

- Inner `variables` actions update an existing outer variable; new names land
  in the top-level run scope. `wait` and `response_variable` values also
  propagate outward, including from parallel sequences.
- Time triggers support weekdays, and datetime-helper time triggers support
  offsets.
- Purpose-specific building blocks understand entity meaning, handle
  `unknown` and `unavailable`, expand area/floor/label targets, and support
  `for` durations where applicable.
- Zone-oriented person and tracker triggers replace the withdrawn home-only
  preview blocks.
- The editor exposes live condition status, target expansion, per-block notes,
  visual continue-on-error, YAML linting, autocomplete, and template hover data.
- Response actions can return structured values; preserve the documented
  `response_variable` shape rather than treating every action as one-way.

## Templates

- Modern template YAML supports lights, switches, covers, fans, locks, alarm
  panels, vacuums, event entities, update entities, and device trackers.
- Trigger-based entities cover the supported template platforms; numeric
  sensors can explicitly become `unknown`, locks support `opening`, and lights
  support xy color.
- Prefer `entity_name()` to reading `friendly_name` directly, and use
  `state_attr_translated()` when comparing or displaying translated attribute
  values.
- Useful helpers include set operations, `flatten`, `shuffle`, hashing,
  `device_name`, alias-aware floor and area lookup, `from_hex`, `clamp`, `wrap`,
  and `remap`.

## Assist, voice, and AI tasks

- `assist_satellite.start_conversation` begins an LLM-backed interaction, while
  `assist_satellite.ask_question` constrains expected answers and returns an ID
  plus captured slots.
- Broadcast, thermostat, shopping-list, to-do removal, lawn-mower, and mapped
  vacuum-area intents extend built-in voice control.
- Conversation agents can stream responses, inspect exposed calendar and to-do
  data, call imported MCP tools, and expose Home Assistant context to MCP
  clients.
- `ai_task.generate_data` accepts files or camera media and returns text or a
  selector-defined object. Configure a default AI Task entity when shared
  automations should omit an explicit target.
- Capable AI Task entities support `ai_task.generate_image`; read the generated
  asset from the response variable's `url` field.
- Treat AI suggestions as data sharing: enabling them can send the full
  automation or script plus related names to the selected provider.

## Dashboards and energy

- Overview is the default dashboard for new installations; the former
  customizable version is available as the legacy template. Built-in Home,
  Maintenance, Security, and protocol views have distinct configuration rules.
- Sections support backgrounds, content-driven auto height, footer cards, and
  custom spacing. The default row gap is 24 px; themes can override
  `ha-view-sections-row-gap`.
- Tile cards support media, weather, trend, gauge, date, fan, valve, counter,
  switch, favorite, and timestamp controls. Verify which feature combinations
  can be inline.
- Energy configuration accepts cumulative energy and live power sensors,
  signed or paired flow sensors, battery state of charge, custom source names,
  parent-child device relationships, and downstream water meters.
- Activity is the current UI name for Logbook and presents data as a contextual
  day-grouped timeline.

## Integration and device checks

- Read the integration reference before assuming that a new platform, action,
  sensor, UI setup path, virtual integration, or sub-entry is unavailable.
- Read the migration reference before writing exact comparisons. Many
  integrations changed title-cased values to lowercase snake-case machine
  values or moved attributes into dedicated entities.
- For Z-Wave JS, use a server and schema that meet the current requirement;
  earlier minimums in upgrade notes are not sufficient for a current system.
- UniFi Protect, BSB-LAN, pyLoad, Mealie, Sentry, Zabbix, Reolink, and other
  integrations have explicit server, API, firmware, or authentication
  requirements. Validate the upstream endpoint before debugging entity code.
- Polling options removed from an integration should be replaced with Home
  Assistant's integration-independent polling customization.
- Infrared, radio-frequency, and serial proxy platforms separate transport from
  device protocol. Select a supported proxy or transmitter; an appliance
  integration must still implement the protocol.

## Custom integration workflow

Before updating a custom integration:

1. Open the custom-integration reference and identify affected config-entry,
   discovery, entity, service, coordinator, storage, selector, and frontend APIs.
2. Replace relocated discovery `ServiceInfo` imports and stop reading removed
   typed-dictionary fields.
3. Treat `UnitSystem` as immutable and use Kelvin color-temperature APIs.
4. Migrate deprecated entity domains, device-tracker APIs, service registration,
   advanced-mode checks, config-entry listeners, and trigger initialization.
5. Verify reconfiguration, unique IDs, subentries, backup-agent progress,
   translated metadata, and brand-image behavior with the running Core version.
6. Run the integration's tests against Python 3.14 and exercise unload/reload,
   reauthentication, retry, and coordinator retrigger paths.
