# Breaking Changes and Migrations

Removed interfaces, renamed values, changed defaults, version requirements, and migration actions.

## 1-Wire raw-value removal (2025.9)

The deprecated `raw_value` attribute has been removed from 1-Wire entities. Update any templates, automations, or exports that read it.

## Authentication-failure notification removal (2025.4)

An integration authentication failure no longer creates a persistent notification with the ID `config_entry_reconfigure`. Automations triggered by that notification must use another signal.

## BSB-LAN version 2 API requirement (2026.7)

BSB-LAN reduces support for the legacy version 1 JSON API; affected devices raise a Repair issue and must be upgraded to firmware supporting the version 2 API.

## Coolmaster fan-mode rename (2026.1)

Coolmaster climate entities now use `medium` instead of `med`; update action data and exact fan-mode comparisons.

## Counter unit removal (2025.1)

1-Wire and FXCOM RFXtrx counter entities no longer report `count` as a unit of measurement.

## Device-tracker battery attributes (2026.7)

iCloud, StarLine, and Tractive device trackers remove their `battery_level` attribute. Update automations, scripts, templates, and cards to use each integration's dedicated battery sensor.

## devolo Home Control URL option removal (2025.1)

The development-only option for overriding the mydevolo URL has been removed from devolo Home Control.

## Entity state and attribute migrations (2025.11)

Asuswrt device trackers remove `last_time_reachable`; use `last_changed` instead. An LG webOS TV entity without a turn-on automation trigger now becomes `unavailable` rather than `off`.

For zone-name-only mobile updates, custom-zone `person` and device-tracker states now use the friendly name, such as `School`, instead of the object ID `kids_school`. Nederlandse Spoorwegen changes its entity from a string to a timestamp entity.

## Entity value migrations (2026.6)

Certificate Expiry's `error` attribute now returns `None` rather than the string `"None"`, so templates should use `is none`. IronOS uptime changes from elapsed seconds to a startup timestamp, SmartThings media-player sources normalize `D.IN` to `digital_input` and `BT` to `bluetooth`, and Tuya now prefers the unit reported by its API over Home Assistant's default.

## Gardena watering-finish migration (2026.5)

Gardena Bluetooth replaces its finish-watering binary sensor with a regular timestamp sensor. Update automations, scripts, and dashboards to reference the new entity.

## Generic Thermostat boundary behavior (2025.5)

Generic Thermostat now turns its target switch on only after the current temperature moves outside the target range plus or minus its tolerances, not when the temperature is exactly at a boundary.

## go2rtc debug authentication (2025.12)

Enabling the go2rtc debug UI now requires both a username and password.

## Google Calendar action removal (2025.7)

The deprecated Google Calendar `add_event` action is removed. Automations and scripts must use the entity-based `create_event` action instead.

## Google Maps Travel Time API migration (2025.5)

The move from Google's Distance Matrix API to Routes API removes the `Destination addresses` and `Origin addresses` attributes. The sensor also polls every 10 minutes instead of every 5 minutes.

## GPSD attribute removal (2025.3)

The deprecated attributes of the GPSD main sensor have been removed; use the dedicated sensor entities introduced in 2024.9.

## HDMI-CEC and ONVIF action semantics (2026.6)

HDMI-CEC `turn_off` now sends the standard standby command; the former vendor-specific behavior remains possible with `hdmi_cec.send_command` keypresses `0x44` then `0x6c`. An `onvif.ptz` call with `continuous_duration: 0` no longer sends Stop after ContinuousMove, so callers must stop it separately or provide a nonzero duration.

## Hive security entity removal (2025.12)

Hive has removed security-product support from its API, so Home Assistant no longer provides Hive alarm-control-panel entities.

## HomeWizard water-unit normalization (2025.1)

The HomeWizard Energy water-usage sensor changes from `l/min` to `L/min`; update exact unit comparisons in automations, scripts, and templates. Long-term statistics remain intact, and Repair issues guide the data update.

## Integration attribute and device migrations (2025.5)

Deprecated 17TRACK entity attributes and Total Connect alarm-panel attributes are removed; use their dedicated sensors. AVM FRITZ!SmartHome merges all units of a physical device into one registry device, potentially changing device targets, while its climate extra-state attributes are deprecated in favor of dedicated entities and are scheduled for removal in 2025.11.

## Integration compatibility and authentication (2025.11)

Mealie 1 is no longer supported; the integration requires Mealie 2 or later. Traccar Server replaces username/password authentication with an API token, so existing entries must generate a token and enter it through **Settings > Devices & services > Traccar Server > Reconfigure**.

## Integration state migrations (2026.3)

BSB-Lan water heaters rename operation mode `on` to `performance`. Satel Integra binary sensors and switches now begin as `unknown` until the panel reports them instead of assuming `off`, and corrected Z-Wave fan scaling can change an exact value such as `67%` to `66%`.

## Integration state semantics (2025.10)

Slide Local's **invert position** option now also inverts open/closed status, so automations around inverted covers may need adjustment. SmartThings renames the AC preset `windFree` to `wind_free`, and ZhongHong climate fan-mode values passed to `set_fan_mode` are now lowercase.

## Jewish Calendar state and attribute changes (2025.4)

In Israel, the holiday states change from `Simchat Torah` to `Shmini Atzeret, Simchat Torah`, and the 30th of Shvat now returns `Family Day, Rosh Chodesh`. The `type_id` state attribute is removed; use `type` instead.

## JVC Projector entity migration (2026.4)

Picture Mode and HDR Processing move from deprecated sensors to `select.jvc_projector_picture_mode` and `select.jvc_projector_hdr_processing`. Unreferenced disabled sensors are removed; referenced sensors remain temporarily with a Repair issue so automations, dashboards, scripts, and templates can migrate.

## KNX State Updater semantics (2025.2)

With State Updater disabled, KNX reads a `state_address` only once when connecting; when enabled, it also rereads an address after one hour without a received value. Existing settings should be reviewed because the option was previously not applied correctly.

## La Marzocco firmware compatibility (2025.5)

La Marzocco now supports gateway firmware v5 and drops older firmware, but boiler temperatures, the shot timer, scales, steam-temperature control, and prebrew/preinfusion controls are unavailable.

## Litter-Robot night-light removal (2026.4)

The deprecated Litter-Robot 4 night-light mode switch is removed. Replace any remaining references with the select entity introduced in 2025.10.

## Meater state normalization (2025.7)

Meater probe cook states are now lowercase machine values: `Not Started` becomes `not_started`, `Configured` becomes `configured`, `Started` becomes `started`, `Ready For Resting` becomes `ready_for_resting`, `Resting` becomes `resting`, `Slightly Underdone` becomes `slightly_underdone`, `Finished` becomes `finished`, `Slightly Overdone` becomes `slightly_overdone`, and `OVERCOOK!` becomes `overcooked`. Update exact state comparisons.

## Media-player off-state migration (2025.8)

Android Debug Bridge, Apple TV, Cambridge Audio, LOOKin, Mediaroom, Roku, Snapcast, and Sony PlayStation 4 media players now report `off` where they previously reported `standby`. Update exact state comparisons; the platform-level `STANDBY` state is deprecated.

## Met Office DataHub migration (2025.6)

The Met Office integration moves from the retired Datapoint API to DataHub and requires a new API key with a Global spot dataset subscription. Forecasts become truly hourly, visibility becomes one precise meter-valued sensor, daily and three-hourly sensor sets are consolidated, and `Site ID`, `Site name`, and `Sensor ID` attributes are removed.

## Miele hob-state migration (2025.7)

Miele hob plate values `0` through `18` become `plate_step_0` through `plate_step_18`; `110` and `220` become `plate_step_warm`; and `117`, `118`, and `217` become `plate_step_boost`. Update automations and templates that compare these states.

## MQTT JSON light migration (2025.3)

Legacy `color_mode` support has been removed from MQTT JSON lights. YAML and discovery configurations still using the deprecated parameters must be updated; discovery use logs a warning.

## NUT state and polling changes (2025.5)

Network UPS Tools status sensors separate multiple statuses with commas instead of spaces. The integration's scan-interval option is removed, polling defaults to 60 seconds, and custom intervals must use Home Assistant's integration-independent polling customization.

## ONVIF preset speed default removal (2025.11)

The `Speed` parameter for ONVIF `GoToPreset` is now optional, but omitting it no longer supplies the former `0.5` default. Set `speed` to `0.5` explicitly when that behavior is required.

## OralB machine-value normalization (2025.11)

OralB replaces spaces with underscores in toothbrush states, brushing modes, pressure values, and sectors. Examples include `flight menu` → `flight_menu`, `daily clean` → `daily_clean`, `button pressed` → `button_pressed`, and `sector 1` → `sector_1`; update exact comparisons throughout those value sets.

## Patch-release capability and state corrections (2026.3)

Versions 2026.3.1 and 2026.3.4 expand Miele steam-oven and oven program support, 2026.3.2 adds area-selector reordering, and Ness Alarm now polls every five seconds. Patch releases also correct state handling for legacy Z-Wave covers and speed mappings for specific GE/Jasco fans, so exact state or percentage checks for affected devices may change.

## Patch-release compatibility (2025.8)

As of 2025.8.3, ESPHome 2025.8.0 is the minimum stable BLE version.

## Patch-release compatibility corrections (2025.7)

As of 2025.7.2, the `hddtemp` deprecation is reverted. Version 2025.7.3 ignores an empty MQTT sensor unit of measurement, and 2025.7.4 keeps entities belonging to dead Z-Wave devices available and adds confirmation to Z-Wave USB migration.

## Pentair ScreenLogic state normalization (2025.2)

ScreenLogic dosing states change from title case to `dosing`, `mixing`, and `monitoring`. Climate `preset_mode` values are also lowercase and normalized, including `solar`, `solar_preferred`, `heater`, and `dont_change`; update exact comparisons.

## Person and zone presence semantics (2026.7)

When a person's location comes from a presence scanner associated with Home, the person entity no longer reports the Home zone's coordinates; use its `in_zones` attribute for zone membership. Zone counts and `persons` now derive from `in_zones`, allowing one person in overlapping zones to count in each, while a position-aware tracker reports the smallest containing zone instead of the zone with the nearest center.

## Plex client-scan action removal (2025.7)

The deprecated `plex.scan_for_clients` action is removed. Use the Plex **Scan Clients** button entity in automations and scripts instead.

## Polling and compatibility changes (2025.10)

HERE Travel Time's automatic polling interval increases from 5 to 30 minutes to fit one route within the new free Base Plan. Zabbix 5.0 is no longer officially supported, although existing connections are not immediately blocked.

## Proximity distance semantics (2025.3)

Proximity distance is now measured to the edge of a monitored zone, including its radius, rather than to the zone center. Adjust automations that depend on the former distance values.

## pyLoad API requirement (2026.4)

The pyLoad integration drops the deprecated 0.4.x API and now requires pyLoad-ng 0.5.0 or newer.

## Removed and suppressed integration entities (2025.11)

Xbox removes the non-updating **Account tier**, **Gold tenure**, **In party**, and **In multiplayer** entities. Renault no longer creates entities inferred from unsupported functionality, so previously present invalid entities can disappear.

## Removed entities, attributes, and actions (2026.3)

Snapcast group media-player entities and Snapcast-specific grouping actions are removed. StarLine engine-switch `ignition` and `autostart` attributes move to binary sensors, while Tado removes mobile-device tracking and its device-tracker entities.

## Removed integration entities and attributes (2025.10)

Home Connect removes the alarm-clock time entity in favor of its number entity, and ZHA removes the unpopulated `target_lift_position` and `target_tilt_position` cover attributes. Shelly Gas removes `Detected` and `Self test` attributes in favor of dedicated entities; Shelly Air removes the Lamp Life entity's `Operational hours` attribute, which now requires a template entity if still needed.

## Removed integration entities and attributes (2025.2)

Ecovacs removes main-brush, side-brush, and filter lifespan attributes in favor of dedicated sensors, while Litter-Robot removes vacuum extra-state attributes already migrated to sensors. Home Connect appliances may lose a power entity when their API omits the setting, and IMGW-PIB removes its flood alarm, flood alarm level, flood warning, and flood warning level entities.

## Removed integrations (2026.7)

Acer projector, Ampio Smog, ATEN Rack PDU, Avi-on, BeeWi SmartClim, BlinkStick, Clementine, Dovado, ELIQ Online, Greenwave Reality, Logentries, Microsoft Face and its Detect/Identify integrations, MS Teams, Mycroft, SCSGate, ThermoWorks Smoke, Tikteck, UniFi LED, and Watson TTS are removed. Gitter's obsolete API integration is also removed, but Gitter is now discoverable as a virtual integration handled by Matrix.

## Ring doorbell event rename (2026.5)

Ring doorbell event entities emit the standardized `ring` event instead of `ding`; update exact event matches.

## Self-hosted Sentry minimum version (2026.2)

The Sentry integration now requires a self-hosted Sentry server at version 20.6.0 or newer because it uses the `/envelope` endpoint. Hosted sentry.io users are unaffected.

## SIA alarm state mapping (2025.9)

SIA status code `CF` (armed with malfunctions) now maps to `armed_away` instead of `armed_custom_bypass`; update exact state comparisons.

## SmartThings entity and state migrations (2025.3)

Energy and power sensors are removed from switch devices that lack the corresponding capabilities. Many appliance, media, and robot-cleaner states were renamed to translatable values, so exact state comparisons must be reviewed.

## SmartThings setup rewrite (2025.3)

SmartThings setup now uses Samsung account login instead of routing configuration, exposed ports, developer accounts, and access tokens. Push updates work without exposing the Home Assistant instance to the internet.

## Strict HEOS grouping (2025.1)

Grouping Denon HEOS players now raises an exception when any member is not a valid HEOS player instead of silently dropping invalid or unknown members.

## SwitchBot vacuum battery migration (2025.9)

SwitchBot Bluetooth vacuum entities now also remove the vacuum battery property in favor of a dedicated battery-level sensor. Update cards, templates, scripts, and automations to use that sensor.

## Tailscale hairpinning sensor removal (2026.1)

The Tailscale **Supports hairpinning** binary sensor has been removed because the upstream API no longer supplies the value.

## TechnoVE state rename (2025.3)

The TechnoVE status sensor value `high_charge_period` is now `high_tariff_period`; update exact comparisons in automations, scripts, and templates.

## Tractive sensor removals (2026.2)

Tractive no longer provides the `activity`, `calories burned`, or `sleep` sensors because its API removed them. Update dashboards, automations, scripts, and templates that reference those entities.

## Tuya and Z-Wave removals (2026.4)

Tuya removes deprecated valve-control switch entities in favor of valve entities. The hidden YAML-enabled Z-Wave Installer panel is also removed; use the equivalent functionality in Z-Wave JS UI.

## UniFi Network device-state values (2025.1)

UniFi Network Device State sensors now expose translatable, lowercase machine values: `connected`, `pending`, `firmware_mismatch`, `upgrading`, `provisioning`, `heartbeat_missed`, `adopting`, `deleting`, `inform_error`, `adoption_failed`, `isolated`, and `unknown`. Update automations, scripts, and templates that compare the former title-cased values.

## UniFi Protect minimum version and authentication (2025.8)

UniFi Protect versions below 6.0.0 are unsupported because the integration is moving to the Public API. On 6.0.0 or newer it attempts to create an API key automatically when the configured user has enough permission; failure starts reauthentication and requires the password and an API key.

## UniFi Protect select-state normalization (2026.1)

UniFi Protect select values now use translated snake-case machine states instead of mixed case, including `Mechanical` → `mechanical`, `Always` → `always`, and `AutoNoLEDsOn` → `auto_no_leds_on`. Update automations, scripts, and templates that set or compare chime, recording, infrared, status-light, HDR, doorbell-text, LCD-message, or other select values.

## Vacuum battery-property removal (2025.8)

Ecovacs, Matter, Miele, Roborock, and Tuya vacuum entities remove their battery property in favor of a dedicated battery-level sensor. Update templates, cards, scripts, and automations to use that sensor; vacuum battery properties are deprecated at the platform API level.

## VeSync fan-mode rename (2026.1)

VeSync changes the `advancedSleep` fan mode to `advanced_sleep`; update automations and scripts that set or compare it.

## VeSync sleep-preset rename (2026.2)

The `advanced_sleep` preset introduced by the 2026.1 normalization is replaced by `sleep`. Update preset selections and exact comparisons in automations and scripts.

## Xbox media and count migrations (2025.12)

Xbox media-source identifiers have changed with the multi-account media-browser rewrite, so saved media-source IDs must be updated. The **following** and **followers** sensors no longer include friends in their counts.

## Yale August OAuth migration (2025.9)

Yale August replaces unofficial authentication with OAuth against the official API. After upgrading, open the August integration, select **Reconfigure**, and complete the one-time sign-in flow.

## Z-Wave JS server requirement (2026.7)

Z-Wave JS now requires zwave-js-server 3.9.0 or newer with schema 49: use Z-Wave JS app 1.4.0 or newer, Z-Wave JS UI Docker 11.19.1 or newer, or a self-managed server at 3.9.0 or newer.

## Z-Wave server requirements (2025.8)

Z-Wave JS now requires `zwave-js-server` 3.2.1 or newer with schema 44. Minimum packaged versions are Z-Wave JS add-on 0.20.0, Z-Wave JS UI add-on 4.8.0, or Z-Wave JS UI Docker 10.11.0.

## Zabbix minimum version (2025.1)

The Zabbix integration now uses the official Python API and requires Zabbix 5.0 or newer; Zabbix 4 and older are unsupported.
