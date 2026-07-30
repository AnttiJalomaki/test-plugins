# Migrations and Breaking Changes

## Removed integrations, entities, and actions

### 1-Wire raw-value removal

*Source batch: 2025.9.*

The deprecated `raw_value` attribute has been removed from 1-Wire entities. Update any templates, automations, or exports that read it.

### Authentication-failure notification removal

*Source batch: 2025.4.*

An integration authentication failure no longer creates a persistent notification with the ID `config_entry_reconfigure`. Automations triggered by that notification must use another signal.

### Counter unit removal

*Source batch: 2025.1.*

1-Wire and FXCOM RFXtrx counter entities no longer report `count` as a unit of measurement.

### devolo Home Control URL option removal

*Source batch: 2025.1.*

The development-only option for overriding the mydevolo URL has been removed from devolo Home Control.

### Google Calendar action removal

*Source batch: 2025.7.*

The deprecated Google Calendar `add_event` action is removed. Automations and scripts must use the entity-based `create_event` action instead.

### GPSD attribute removal

*Source batch: 2025.3.*

The deprecated attributes of the GPSD main sensor have been removed; use the dedicated sensor entities introduced in 2024.9.

### Hive security entity removal

*Source batch: 2025.12.*

Hive has removed security-product support from its API, so Home Assistant no longer provides Hive alarm-control-panel entities.

### Integration additions, setup, and removal

*Source batch: 2025.7.*

New integrations cover Altruist environmental sensors, PlayStation Network activity, Tilt Pi brewing measurements, and VegeHub garden monitoring and irrigation control. Telegram Bot can now be set up from the UI, while JuiceNet is removed because its API service shut down.

### Integration setup and removal

*Source batch: 2025.9.*

Bayesian can now be configured from the UI. Uonet+ Vulcan is removed because its changed API policy prohibits unofficial clients.

### Integration setup and removal

*Source batch: 2026.6.*

Elgato Avea, openSenseMap, and OPNsense can now be configured through the UI. The legacy Konnected integration is removed after its 2025.10 deprecation; affected hardware must migrate to ESPHome firmware.

### Integration setup and removals

*Source batch: 2025.1.*

Niko Home Control can now be configured from the UI. DTE Energy Bridge, Simulated, and Stookalert are removed; Stookwijzer is the suggested replacement for Stookalert.

### Integration setup and removals

*Source batch: 2025.12.*

DuckDNS can now be configured from the UI. Dominos Pizza and Flick Electric are removed, as are Bluetooth Tracker, CUPS, Decora, dlib Face Detect, dlib Face Identify, Eddystone Temperature, GStreamer, Keyboard, LIRC, Pandora, Raspberry Pi Camera, SMS, Snips, and TensorFlow because they are incompatible with supported installation methods.

### Integration setup and removals

*Source batch: 2026.4.*

Leviton Decora Wi-Fi and Orvibo can now be configured through the UI. BMW Connected Drive/Mini Connected, Duke Energy, and Tfiac are removed; BMW's server restrictions leave the CarData API as the alternative for eligible EU vehicles.

### Integration setup and removals

*Source batch: 2026.5.*

PJLink and Pico TTS can now be configured from the UI. LANnouncer is removed after its companion Android app became unavailable, and the unmaintained pilight integration is disabled until its dependency on `setuptools.pkg_resources` is removed.

### Litter-Robot night-light removal

*Source batch: 2026.4.*

The deprecated Litter-Robot 4 night-light mode switch is removed. Replace any remaining references with the select entity introduced in 2025.10.

### New and removed integrations

*Source batch: 2025.6.*

New integrations cover Alexa devices, Immich, Paperless-ngx, Probe Plus Bluetooth thermometers, Swing2Sleep Smarla cradles, and Zimi Cloud devices; Kaiser Nienhaus is available virtually through Motionblinds. RTSPtoWebRTC is removed and replaced by the go2rtc integration.

### New integrations, UI setup, and removals

*Source batch: 2025.11.*

New integrations add Actron Air, Sunricher DALI, Fing, Firefly III, iNELS, Lunatone Gateway, Meteo.lt, Nintendo Parental Controls, and OpenRGB. London Underground can now be configured from the UI; Vultr, IBM Watson IoT Platform, and Plum Lightpad are removed because their backing services or APIs are no longer functional.

### ONVIF preset speed default removal

*Source batch: 2025.11.*

The `Speed` parameter for ONVIF `GoToPreset` is now optional, but omitting it no longer supplies the former `0.5` default. Set `speed` to `0.5` explicitly when that behavior is required.

### Plex client-scan action removal

*Source batch: 2025.7.*

The deprecated `plex.scan_for_clients` action is removed. Use the Plex **Scan Clients** button entity in automations and scripts instead.

### Removed and suppressed integration entities

*Source batch: 2025.11.*

Xbox removes the non-updating **Account tier**, **Gold tenure**, **In party**, and **In multiplayer** entities. Renault no longer creates entities inferred from unsupported functionality, so previously present invalid entities can disappear.

### Removed entities, attributes, and actions

*Source batch: 2026.3.*

Snapcast group media-player entities and Snapcast-specific grouping actions are removed. StarLine engine-switch `ignition` and `autostart` attributes move to binary sensors, while Tado removes mobile-device tracking and its device-tracker entities.

### Removed integration entities and attributes

*Source batch: 2025.10.*

Home Connect removes the alarm-clock time entity in favor of its number entity, and ZHA removes the unpopulated `target_lift_position` and `target_tilt_position` cover attributes. Shelly Gas removes `Detected` and `Self test` attributes in favor of dedicated entities; Shelly Air removes the Lamp Life entity's `Operational hours` attribute, which now requires a template entity if still needed.

### Removed integration entities and attributes

*Source batch: 2025.2.*

Ecovacs removes main-brush, side-brush, and filter lifespan attributes in favor of dedicated sensors, while Litter-Robot removes vacuum extra-state attributes already migrated to sensors. Home Connect appliances may lose a power entity when their API omits the setting, and IMGW-PIB removes its flood alarm, flood alarm level, flood warning, and flood warning level entities.

### Removed integrations

*Source batch: 2026.7.*

Acer projector, Ampio Smog, ATEN Rack PDU, Avi-on, BeeWi SmartClim, BlinkStick, Clementine, Dovado, ELIQ Online, Greenwave Reality, Logentries, Microsoft Face and its Detect/Identify integrations, MS Teams, Mycroft, SCSGate, ThermoWorks Smoke, Tikteck, UniFi LED, and Watson TTS are removed. Gitter's obsolete API integration is also removed, but Gitter is now discoverable as a virtual integration handled by Matrix.

### Tailscale hairpinning sensor removal

*Source batch: 2026.1.*

The Tailscale **Supports hairpinning** binary sensor has been removed because the upstream API no longer supplies the value.

### Tractive sensor removals

*Source batch: 2026.2.*

Tractive no longer provides the `activity`, `calories burned`, or `sleep` sensors because its API removed them. Update dashboards, automations, scripts, and templates that reference those entities.

### Tuya and Z-Wave removals

*Source batch: 2026.4.*

Tuya removes deprecated valve-control switch entities in favor of valve entities. The hidden YAML-enabled Z-Wave Installer panel is also removed; use the equivalent functionality in Z-Wave JS UI.

### Vacuum battery-property removal

*Source batch: 2025.8.*

Ecovacs, Matter, Miele, Roborock, and Tuya vacuum entities remove their battery property in favor of a dedicated battery-level sensor. Update templates, cards, scripts, and automations to use that sensor; vacuum battery properties are deprecated at the platform API level.

## State, value, and unit migrations

### Modbus and Velux configuration/state corrections

*Source batch: 2025.9.*

As of 2025.9.2, Modbus accepts delays greater than one and non-integer `min_temp` and `max_temp` values for lights. Velux determines closed status from position percentage, which can change the state exposed to automations.

### Coolmaster fan-mode rename

*Source batch: 2026.1.*

Coolmaster climate entities now use `medium` instead of `med`; update action data and exact fan-mode comparisons.

### Entity state and attribute migrations

*Source batch: 2025.11.*

Asuswrt device trackers remove `last_time_reachable`; use `last_changed` instead. An LG webOS TV entity without a turn-on automation trigger now becomes `unavailable` rather than `off`.

For zone-name-only mobile updates, custom-zone `person` and device-tracker states now use the friendly name, such as `School`, instead of the object ID `kids_school`. Nederlandse Spoorwegen changes its entity from a string to a timestamp entity.

### Entity value migrations

*Source batch: 2026.6.*

Certificate Expiry's `error` attribute now returns `None` rather than the string `"None"`, so templates should use `is none`. IronOS uptime changes from elapsed seconds to a startup timestamp, SmartThings media-player sources normalize `D.IN` to `digital_input` and `BT` to `bluetooth`, and Tuya now prefers the unit reported by its API over Home Assistant's default.

### Gardena watering-finish migration

*Source batch: 2026.5.*

Gardena Bluetooth replaces its finish-watering binary sensor with a regular timestamp sensor. Update automations, scripts, and dashboards to reference the new entity.

### Google Maps Travel Time API migration

*Source batch: 2025.5.*

The move from Google's Distance Matrix API to Routes API removes the `Destination addresses` and `Origin addresses` attributes. The sensor also polls every 10 minutes instead of every 5 minutes.

### HDMI-CEC and ONVIF action semantics

*Source batch: 2026.6.*

HDMI-CEC `turn_off` now sends the standard standby command; the former vendor-specific behavior remains possible with `hdmi_cec.send_command` keypresses `0x44` then `0x6c`. An `onvif.ptz` call with `continuous_duration: 0` no longer sends Stop after ContinuousMove, so callers must stop it separately or provide a nonzero duration.

### HomeWizard water-unit normalization

*Source batch: 2025.1.*

The HomeWizard Energy water-usage sensor changes from `l/min` to `L/min`; update exact unit comparisons in automations, scripts, and templates. Long-term statistics remain intact, and Repair issues guide the data update.

### Immutable UnitSystem

*Source batch: 2025.4.*

For custom integrations, the `UnitSystem` dataclass is now frozen and must be treated as immutable.

### Integration attribute and device migrations

*Source batch: 2025.5.*

Deprecated 17TRACK entity attributes and Total Connect alarm-panel attributes are removed; use their dedicated sensors. AVM FRITZ!SmartHome merges all units of a physical device into one registry device, potentially changing device targets, while its climate extra-state attributes are deprecated in favor of dedicated entities and are scheduled for removal in 2025.11.

### Integration state migrations

*Source batch: 2026.3.*

BSB-Lan water heaters rename operation mode `on` to `performance`. Satel Integra binary sensors and switches now begin as `unknown` until the panel reports them instead of assuming `off`, and corrected Z-Wave fan scaling can change an exact value such as `67%` to `66%`.

### Integration state semantics

*Source batch: 2025.10.*

Slide Local's **invert position** option now also inverts open/closed status, so automations around inverted covers may need adjustment. SmartThings renames the AC preset `windFree` to `wind_free`, and ZhongHong climate fan-mode values passed to `set_fan_mode` are now lowercase.

### Jewish Calendar state and attribute changes

*Source batch: 2025.4.*

In Israel, the holiday states change from `Simchat Torah` to `Shmini Atzeret, Simchat Torah`, and the 30th of Shvat now returns `Family Day, Rosh Chodesh`. The `type_id` state attribute is removed; use `type` instead.

### JVC Projector entity migration

*Source batch: 2026.4.*

Picture Mode and HDR Processing move from deprecated sensors to `select.jvc_projector_picture_mode` and `select.jvc_projector_hdr_processing`. Unreferenced disabled sensors are removed; referenced sensors remain temporarily with a Repair issue so automations, dashboards, scripts, and templates can migrate.

### KNX State Updater semantics

*Source batch: 2025.2.*

With State Updater disabled, KNX reads a `state_address` only once when connecting; when enabled, it also rereads an address after one hour without a received value. Existing settings should be reviewed because the option was previously not applied correctly.

### Meater state normalization

*Source batch: 2025.7.*

Meater probe cook states are now lowercase machine values: `Not Started` becomes `not_started`, `Configured` becomes `configured`, `Started` becomes `started`, `Ready For Resting` becomes `ready_for_resting`, `Resting` becomes `resting`, `Slightly Underdone` becomes `slightly_underdone`, `Finished` becomes `finished`, `Slightly Overdone` becomes `slightly_overdone`, and `OVERCOOK!` becomes `overcooked`. Update exact state comparisons.

### Media-player off-state migration

*Source batch: 2025.8.*

Android Debug Bridge, Apple TV, Cambridge Audio, LOOKin, Mediaroom, Roku, Snapcast, and Sony PlayStation 4 media players now report `off` where they previously reported `standby`. Update exact state comparisons; the platform-level `STANDBY` state is deprecated.

### Met Office DataHub migration

*Source batch: 2025.6.*

The Met Office integration moves from the retired Datapoint API to DataHub and requires a new API key with a Global spot dataset subscription. Forecasts become truly hourly, visibility becomes one precise meter-valued sensor, daily and three-hourly sensor sets are consolidated, and `Site ID`, `Site name`, and `Sensor ID` attributes are removed.

### Micro-unit encoding changes

*Source batch: 2025.9.*

The encoding changed for `μSv/h`, `μS/cm`, `μV`, `μg/ft³`, `μg/m³`, `μmol/s⋅m²`, `μg`, and `μs`. Review exact unit consumers and exported state data such as InfluxDB series.

### Miele hob-state migration

*Source batch: 2025.7.*

Miele hob plate values `0` through `18` become `plate_step_0` through `plate_step_18`; `110` and `220` become `plate_step_warm`; and `117`, `118`, and `217` become `plate_step_boost`. Update automations and templates that compare these states.

### MQTT JSON light migration

*Source batch: 2025.3.*

Legacy `color_mode` support has been removed from MQTT JSON lights. YAML and discovery configurations still using the deprecated parameters must be updated; discovery use logs a warning.

### NUT state and polling changes

*Source batch: 2025.5.*

Network UPS Tools status sensors separate multiple statuses with commas instead of spaces. The integration's scan-interval option is removed, polling defaults to 60 seconds, and custom intervals must use Home Assistant's integration-independent polling customization.

### OralB machine-value normalization

*Source batch: 2025.11.*

OralB replaces spaces with underscores in toothbrush states, brushing modes, pressure values, and sectors. Examples include `flight menu` → `flight_menu`, `daily clean` → `daily_clean`, `button pressed` → `button_pressed`, and `sector 1` → `sector_1`; update exact comparisons throughout those value sets.

### Patch-release capability and state changes

*Source batch: 2026.4.*

Patch releases remove Transmission's port-forward sensor, enable SwitchBot Cloud Bot webhooks and Comelit force-alarm actions, add Matter dry/fan modes for Hisense air conditioners, and add missing Miele dishwasher and steam-oven codes. They also correct Z-Wave cover movement states and Gardena water state/device classes, so exact state or class assumptions for affected entities should be reviewed.

### Patch-release capability and state corrections

*Source batch: 2026.3.*

Versions 2026.3.1 and 2026.3.4 expand Miele steam-oven and oven program support, 2026.3.2 adds area-selector reordering, and Ness Alarm now polls every five seconds. Patch releases also correct state handling for legacy Z-Wave covers and speed mappings for specific GE/Jasco fans, so exact state or percentage checks for affected devices may change.

### Pentair ScreenLogic state normalization

*Source batch: 2025.2.*

ScreenLogic dosing states change from title case to `dosing`, `mixing`, and `monitoring`. Climate `preset_mode` values are also lowercase and normalized, including `solar`, `solar_preferred`, `heater`, and `dont_change`; update exact comparisons.

### Person and zone presence semantics

*Source batch: 2026.7.*

When a person's location comes from a presence scanner associated with Home, the person entity no longer reports the Home zone's coordinates; use its `in_zones` attribute for zone membership. Zone counts and `persons` now derive from `in_zones`, allowing one person in overlapping zones to count in each, while a position-aware tracker reports the smallest containing zone instead of the zone with the nearest center.

### Proximity distance semantics

*Source batch: 2025.3.*

Proximity distance is now measured to the edge of a monitored zone, including its radius, rather than to the zone center. Adjust automations that depend on the former distance values.

### Ring doorbell event rename

*Source batch: 2026.5.*

Ring doorbell event entities emit the standardized `ring` event instead of `ding`; update exact event matches.

### Sensor-group unavailable and unknown states

*Source batch: 2026.2.*

A sensor group is now `unavailable` when every member is unavailable or absent from the state machine. Otherwise, with the default `ignore_non_numeric: false`, it is calculated only when every member exists and is numeric; a missing or nonnumeric member makes the group `unknown`.

### SIA alarm state mapping

*Source batch: 2025.9.*

SIA status code `CF` (armed with malfunctions) now maps to `armed_away` instead of `armed_custom_bypass`; update exact state comparisons.

### SmartThings entity and state migrations

*Source batch: 2025.3.*

Energy and power sensors are removed from switch devices that lack the corresponding capabilities. Many appliance, media, and robot-cleaner states were renamed to translatable values, so exact state comparisons must be reviewed.

### SwitchBot vacuum battery migration

*Source batch: 2025.9.*

SwitchBot Bluetooth vacuum entities now also remove the vacuum battery property in favor of a dedicated battery-level sensor. Update cards, templates, scripts, and automations to use that sensor.

### TechnoVE state rename

*Source batch: 2025.3.*

The TechnoVE status sensor value `high_charge_period` is now `high_tariff_period`; update exact comparisons in automations, scripts, and templates.

### UniFi Protect select-state normalization

*Source batch: 2026.1.*

UniFi Protect select values now use translated snake-case machine states instead of mixed case, including `Mechanical` → `mechanical`, `Always` → `always`, and `AutoNoLEDsOn` → `auto_no_leds_on`. Update automations, scripts, and templates that set or compare chime, recording, infrared, status-light, HDR, doorbell-text, LCD-message, or other select values.

### VeSync fan-mode rename

*Source batch: 2026.1.*

VeSync changes the `advancedSleep` fan mode to `advanced_sleep`; update automations and scripts that set or compare it.

### VeSync sleep-preset rename

*Source batch: 2026.2.*

The `advanced_sleep` preset introduced by the 2026.1 normalization is replaced by `sleep`. Update preset selections and exact comparisons in automations and scripts.

### Xbox media and count migrations

*Source batch: 2025.12.*

Xbox media-source identifiers have changed with the multi-account media-browser rewrite, so saved media-source IDs must be updated. The **following** and **followers** sensors no longer include friends in their counts.

### Yale August OAuth migration

*Source batch: 2025.9.*

Yale August replaces unofficial authentication with OAuth against the official API. After upgrading, open the August integration, select **Reconfigure**, and complete the one-time sign-in flow.

## Authentication, versions, and runtime compatibility

### BSB-LAN version 2 API requirement

*Source batch: 2026.7.*

BSB-LAN reduces support for the legacy version 1 JSON API; affected devices raise a Repair issue and must be upgraded to firmware supporting the version 2 API.

### go2rtc debug authentication

*Source batch: 2025.12.*

Enabling the go2rtc debug UI now requires both a username and password.

### Integration compatibility and authentication

*Source batch: 2025.11.*

Mealie 1 is no longer supported; the integration requires Mealie 2 or later. Traccar Server replaces username/password authentication with an API token, so existing entries must generate a token and enter it through **Settings > Devices & services > Traccar Server > Reconfigure**.

### La Marzocco firmware compatibility

*Source batch: 2025.5.*

La Marzocco now supports gateway firmware v5 and drops older firmware, but boiler temperatures, the shot timer, scales, steam-temperature control, and prebrew/preinfusion controls are unavailable.

### Patch-release compatibility

*Source batch: 2025.8.*

As of 2025.8.3, ESPHome 2025.8.0 is the minimum stable BLE version.

### Patch-release compatibility corrections

*Source batch: 2025.7.*

As of 2025.7.2, the `hddtemp` deprecation is reverted. Version 2025.7.3 ignores an empty MQTT sensor unit of measurement, and 2025.7.4 keeps entities belonging to dead Z-Wave devices available and adds confirmation to Z-Wave USB migration.

### Polling and compatibility changes

*Source batch: 2025.10.*

HERE Travel Time's automatic polling interval increases from 5 to 30 minutes to fit one route within the new free Base Plan. Zabbix 5.0 is no longer officially supported, although existing connections are not immediately blocked.

### pyLoad API requirement

*Source batch: 2026.4.*

The pyLoad integration drops the deprecated 0.4.x API and now requires pyLoad-ng 0.5.0 or newer.

### Self-hosted Sentry minimum version

*Source batch: 2026.2.*

The Sentry integration now requires a self-hosted Sentry server at version 20.6.0 or newer because it uses the `/envelope` endpoint. Hosted sentry.io users are unaffected.

### UniFi Protect minimum version and authentication

*Source batch: 2025.8.*

UniFi Protect versions below 6.0.0 are unsupported because the integration is moving to the Public API. On 6.0.0 or newer it attempts to create an API key automatically when the configured user has enough permission; failure starts reauthentication and requires the password and an API key.

### Z-Wave JS server requirement

*Source batch: 2026.7.*

Z-Wave JS now requires zwave-js-server 3.9.0 or newer with schema 49: use Z-Wave JS app 1.4.0 or newer, Z-Wave JS UI Docker 11.19.1 or newer, or a self-managed server at 3.9.0 or newer.

### Z-Wave server requirements

*Source batch: 2025.8.*

Z-Wave JS now requires `zwave-js-server` 3.2.1 or newer with schema 44. Minimum packaged versions are Z-Wave JS add-on 0.20.0, Z-Wave JS UI add-on 4.8.0, or Z-Wave JS UI Docker 10.11.0.

### Zabbix minimum version

*Source batch: 2025.1.*

The Zabbix integration now uses the official Python API and requires Zabbix 5.0 or newer; Zabbix 4 and older are unsupported.

## Polling, validation, and behavior changes

### Device-tracker battery attributes

*Source batch: 2026.7.*

iCloud, StarLine, and Tractive device trackers remove their `battery_level` attribute. Update automations, scripts, templates, and cards to use each integration's dedicated battery sensor.

### Generic Thermostat boundary behavior

*Source batch: 2025.5.*

Generic Thermostat now turns its target switch on only after the current temperature moves outside the target range plus or minus its tolerances, not when the temperature is exactly at a boundary.

### Generic thermostat preset selection

*Source batch: 2025.2.*

Setting a Generic Thermostat temperature equal to one of its preset temperatures now automatically makes that preset active.

### Home Connect restrictions

*Source batch: 2025.3.*

Programs without an `aiohomeconnect` program-key enumeration may no longer be exposed, and undocumented program or option keys no longer work in actions. Only one Home Connect config entry can now be configured.

### Label-target expansion

*Source batch: 2025.10.*

Service actions targeting a label now include configuration and diagnostic entities carrying that label; audit labeled entities so an action does not unexpectedly affect controls that were previously excluded.

### Motion Blinds tilt commands

*Source batch: 2026.4.*

For tilt-capable devices that do not report an absolute tilt position, tilt-open and tilt-close now send jog-up and jog-down commands instead of attempting 0° and 180° positions. Automations that relied on the former absolute movement may need adjustment.

### Motion Blinds tilt inversion

*Source batch: 2025.11.*

Motion Blinds tilt position now follows `0` = closed and `100` = open, so migrate positions with `new = 100 - old`. Automations that depended on the old orientation must also swap `open_cover_tilt` and `close_cover_tilt`.

### Strict HEOS grouping

*Source batch: 2025.1.*

Grouping Denon HEOS players now raises an exception when any member is not a valid HEOS player instead of silently dropping invalid or unknown members.

### Supervisor action failures

*Source batch: 2026.5.*

Supervisor actions such as `hassio.addon_start`, `hassio.backup_partial`, and `hassio.host_reboot` now raise on failure, stopping scripts and automations by default. Add `continue_on_error: true` to an action step only when the previous continue-after-failure behavior is required.

### Synology DSM polling

*Source batch: 2025.3.*

The Synology DSM scan-interval option has been removed and polling now defaults to 15 minutes. Use Home Assistant's integration-independent polling customization when another interval is required.

### Telegram bot action validation

*Source batch: 2026.1.*

Telegram bot actions no longer accept undefined or unused parameters. Remove any fields that are not part of the supported notification-action schema.

## Patch-release corrections

### Z-Wave entity defaults

*Source batch: 2025.6.*

As of 2025.6.2, Z-Wave Indicator Command Class entities and the idle-notification button are disabled by default.

### Patch-release capability changes

*Source batch: 2026.5.*

Version 2026.5.1 adds option matching to to-do triggers and WiZ `wfsens` occupancy; 2026.5.2 adds Duco target-flow and mode-end sensors but removes its temperature sensors after a connectivity-library migration. Version 2026.5.3 adds CalDAV `uid` and `recurrence_id` values and more Overkiz tilt and stop controls, while 2026.5.4 adds missing Miele dishwasher codes.

### Patch-release entity and validation changes

*Source batch: 2025.4.*

As of 2025.4.1, the built-in Music Assistant player no longer creates a Home Assistant media-player entity, and SmartThings climate entities gain preset mode. Version 2025.4.2 permits equal minimum and maximum values in MQTT number configuration; 2025.4.4 creates Home Connect active- and selected-program entities only when the appliance exposes programs.

### Patch-release integration behavior

*Source batch: 2026.2.*

As of 2026.2.1, Denon AVR media players map stopped playback to the `stopped` state. Version 2026.2.2 adds Miele TQ1000WP programs and phases, while 2026.2.3 adds Roborock region selection and Miele dishwasher program codes.

### Patch-release public behavior

*Source batch: 2026.6.*

Version 2026.6.3 makes `config_entry_attr` return enum values. Version 2026.6.4 corrects Growatt V1 `total_output_power` values that were 1,000 times too low, adds Subaru API generation 4 support, includes Sonos favorites in source lists while exposing source selection only when supported, and stops `zwave_js.set_credential` from validating the number of credential slots.

### Patch-release public behavior

*Source batch: 2026.7.*

Patch releases make HomeKit Controller doorbell event entities use the standardized `ring` event, restore SolarEdge energy-sensor units, fix Proximity matching against trackers with `in_zones`, and stop Teslemetry covers from reporting false open or closed states when data is absent. They also correct YoLink water-meter valve status, refresh App update entities after a store reload, and exempt certain protocol integrations from the entity limit.
