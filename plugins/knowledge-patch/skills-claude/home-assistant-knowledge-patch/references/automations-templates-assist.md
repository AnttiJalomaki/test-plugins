# Automations, Templates, Assist, and AI

## Breaking automation migrations

### Strict webhook booleans

*Source batch: 2026.5.*

Webhook `local_only` now accepts only the booleans `true` or `false`; replace formerly accepted truthy values such as `1` or `"yes"`.

## Automation and script authoring

### Automation and script metadata

*Source batch: 2025.1.*

When creating or renaming an automation or script, the editor can now set its description, category, labels, and areas directly.

### Automation and script variable scopes

*Source batch: 2025.4.*

A `variables` action in a nested scope now updates an existing variable in an outer scope; a newly named variable is created in the top-level script-run scope. Rename variables that previously relied on local shadowing.

```yaml
actions:
  - variables:
      x: 1
  - sequence:
      - variables:
          x: 2
          y: 3
  - action: persistent_notification.create
    data:
      message: "{{ x }}, {{ y }}"  # 2, 3
```

### Automation and sensor schema additions

*Source batch: 2025.10.*

Webhook triggers can template `webhook_id`, while lawn-mower entities gain voice intents for starting mowing and docking. Accepted units now include `MCF` for volume, `m/min` for speed, and `inH₂O` for pressure.

### Automation editor context and notes

*Source batch: 2026.6.*

Floor, area, device, and label targets now show their expanded entity count, honor domain and device-class filters, and reveal the included entities when selected. Conditions show live pass, fail, neutral, or error badges in automations and dashboard visibility rules, while every trigger, condition, action, option, and script field can carry a note that survives duplication, export, and blueprint sharing.

### Automation editor refinements

*Source batch: 2025.10.*

The automation sidebar is resizable, and `Ctrl+V` pastes a previously copied block immediately below the selected block. Undo and redo retain up to 75 editing steps with `Ctrl+Z` and `Ctrl+Y`; the repeat chooser is split into fixed-count, until, while, and for-each blocks, but repeat YAML is unchanged.

### Automation editor sidebar

*Source batch: 2025.9.*

Selecting a trigger, condition, or action now opens its settings in a right-hand sidebar while preserving the automation overview. On small screens this becomes a resizable bottom sheet, and mobile editing now supports drag and drop.

### Backup and subscription diagnostics

*Source batch: 2025.6.*

An incomplete backup now raises a Repair issue when any add-on or folder was not successfully backed up, and an automatic-backup event entity tracks automatic backups. Home Assistant Cloud also raises a Repair issue for an expired subscription.

### Blueprint device-model filtering

*Source batch: 2025.4.*

Blueprint device selectors can now filter eligible devices by model ID.

### Custom and automation-driven backup schedules

*Source batch: 2025.2.*

Automatic backups can run at a chosen time, and weekly schedules can select specific weekdays. Advanced schedules can invoke the new automatic-backup action:

```yaml
actions:
  - action: backup.create_automatic
```

### Dashboard and automation editor improvements

*Source batch: 2025.5.*

Dashboard header badges can retain the default wrapping behavior or scroll horizontally. Automation and script editors keep structured controls around individual templated fields, and pasted YAML can be converted into UI form whether it contains a whole automation or a single trigger, condition, or action.

### Datetime-helper trigger offsets

*Source batch: 2025.2.*

Automation time triggers based on datetime input helpers now support offsets.

### Discovery automation event

*Source batch: 2025.5.*

Device discovery no longer creates the `config_entry_discovery` persistent notification. Automations must trigger on the `config_entry_discovered` event instead.

### Media, camera, and automation integrations

*Source batch: 2026.6.*

Alexa Devices gains media-player entities for Echo playback, volume, and mute plus event entities for heard voice commands; Sonos gains cross-service media search. Reolink battery cameras can connect directly without a hub or NVR, although support is incomplete and a camera cannot use both connection methods simultaneously; Google Nest adds `nest.set_fan_timer`, OneDrive a delete action, and Portainer container-recreation actions.

### Recorder statistics action

*Source batch: 2025.6.*

The new `recorder.get_statistics` action queries statistics directly from the recorder for use in automations and scripts.

### Script variable propagation

*Source batch: 2025.3.*

`wait` and `response_variable` values created in an inner script or automation scope now propagate outward even when that scope contains a `variables` action. A `response_variable` also propagates out of `parallel` sequences, so flows relying on the former isolation need adjustment.

### Weekdays in time triggers

*Source batch: 2025.8.*

Time triggers can now be restricted to specified weekdays, allowing a time-based automation to run only on selected days without a separate weekday condition.

## Purpose-specific triggers and conditions

### Cross-domain purpose-specific automations

*Source batch: 2026.4.*

The Labs preview now offers triggers and conditions organized by real-world meaning across entity domains: doors, garage doors, gates, windows, motion, occupancy, temperature, humidity, illuminance, power, battery, air quality, and climate. They can target areas, floors, or labels, so a concept such as an upstairs window automatically includes matching binary sensors and covers.

### Duration-aware purpose-specific automations

*Source batch: 2026.5.*

State-based purpose-specific triggers gain a visual **for** duration across domains such as motion, occupancy, doors, lights, climate, and covers. Every Labs entity condition can likewise require that its state has held for a specified duration.

### More purpose-specific automation building blocks

*Source batch: 2026.5.*

New conditions cover update availability, remote state, to-do-list completion, media-player mute state, and numeric volume; media players also gain playback, power, mute, and volume triggers. Timers gain started, paused, restarted, cancelled, finished, and time-remaining triggers, and standardized doorbell event entities gain a brand-independent doorbell-rang trigger.

### Purpose-specific automations become the default

*Source batch: 2026.7.*

Purpose-specific triggers and conditions have graduated from Labs and are now the automation editor's default starting point; existing automations, generic building blocks, templates, and YAML continue to work without migration. These building blocks can target areas, handle `unknown`, `unavailable`, and repeated event-entity events according to their purpose, and can now be supplied by integrations, including custom integrations.

### Purpose-specific key migrations

*Source batch: 2026.7.*

Several preview trigger and condition keys were renamed for consistency, and the old keys no longer work; reselect and save affected blocks in the editor or replace the YAML keys:

```text
battery.low                 -> battery.became_low
battery.not_low             -> battery.no_longer_low
lawn_mower.docked           -> lawn_mower.returned_to_dock
schedule.turned_off         -> schedule.block_ended
schedule.turned_on          -> schedule.block_started
timer.time_remaining        -> timer.remaining_time_reached
update.update_became_available -> update.became_available
vacuum.docked               -> vacuum.returned_to_dock
climate.target_humidity     -> climate.is_target_humidity
climate.target_temperature  -> climate.is_target_temperature
```

### Purpose-specific trigger expansion

*Source batch: 2026.1.*

The Labs preview adds triggers for button presses; device-tracker arrivals and departures, including first arrival and last departure; humidifier activity and humidity; lock state; scene activation; siren state; and available updates. Climate triggers now cover HVAC mode, target and current temperature, and humidity changes or threshold crossings, while light triggers cover brightness changes and thresholds; the automation flow also has a redesigned target summary.

### Purpose-specific triggers and conditions

*Source batch: 2025.12.*

A Labs preview lets domains such as Light, Climate, and Fan provide target-aware triggers and conditions instead of requiring generic state logic. Entity, device, area, and label targets are supported, so an automation can react when any matching light turns on or test whether any is on, while area targets automatically follow membership changes.

### Purpose-specific triggers and conditions

*Source batch: 2026.2.*

The Labs preview adds calendar event start/end, person home arrival/departure, and vacuum dock-return triggers. New conditions cover alarm-panel armed variants/disarmed/triggered; Assist-satellite activity; climate, humidifier, mower, lock, media-player, and vacuum states; and home/not-home or on/off checks for trackers, people, fans, sirens, and switches.

### Zone-oriented purpose-specific automations

*Source batch: 2026.6.*

The Labs preview adds triggers for a person or device tracker entering or leaving any zone and for a zone becoming occupied or empty, plus matching in-zone, not-in-zone, occupied, and empty conditions. All eight building blocks support `for` durations and replace the home-only Person and Device Tracker options withdrawn in 2026.5.

## Templates and helpers

### Active issues in templates

*Source batch: 2025.12.*

The `issues()` template function now returns only active issues; fixed issues are no longer included.

### Media search and template helpers

*Source batch: 2025.5.*

Media players gain the `media_player.search_media` action. Templates gain `device_name`, while `floor_id` and `area_id` also resolve configured aliases.

### MQTT and template device trackers

*Source batch: 2026.6.*

MQTT gains a message-expiry interval and extends subentry support to date, datetime, and time entities. Template entities now support the device-tracker platform, providing the modern replacement for the deprecated `device_tracker.see` action.

### Reloadable shell commands and template vacuum rooms

*Source batch: 2026.5.*

A new Shell Command reload action rereads its YAML without restarting Home Assistant. Template vacuums can expose room segments and a `clean_segment` action, allowing them to participate in area-based cleaning.

### Relocated tools, themes, and template previews

*Source batch: 2026.2.*

Theme selection moves to each user's profile and follows that user across signed-in devices, while Developer tools move under **Settings**. The template editor now shows a live inline result as a template is edited.

### Removed template syntax and Velux action

*Source batch: 2026.6.*

Legacy template entities under the individual `alarm_control_panel`, `binary_sensor`, `cover`, `fan`, `light`, `lock`, `sensor`, `switch`, `vacuum`, and `weather` platform keys are removed; migrate them under modern `template:` configuration. The deprecated `velux.reboot_gateway` action is also removed in favor of the gateway's reboot button entity.

### Template and dashboard utilities

*Source batch: 2025.12.*

Templates gain the `clamp`, `wrap`, and `remap` math functions. The Activity card can filter by state, Tile-card bar gauges accept `min` and `max`, and the Blueprints panel shows how many automations and scripts use each blueprint.

### Template binary-sensor `None` semantics

*Source batch: 2025.8.*

A template binary sensor whose state template returns `None` now becomes `unknown` instead of `off`. Return `False` explicitly when the intended state is `off`.

### Template entities and units

*Source batch: 2025.9.*

All modern template-entity YAML syntax can set a default entity ID, and the template integration can now create event and update entities. Volume-flow-rate entities also support `m³/min`.

### Template entity and filter additions

*Source batch: 2025.6.*

Modern-style YAML now supports template fans, locks, alarm control panels, and vacuums, while covers can be trigger-based. Trigger `for` clauses can use `trigger_variables`; templates also gain the `from_hex` filter, and `base64_encode` accepts both bytes and strings.

### Template entity configuration

*Source batch: 2025.5.*

Switch and light template entities can now be trigger-based, and cover template entities support modern YAML syntax.

### Template fan error and unknown states

*Source batch: 2026.3.*

A template fan whose `state` template has a syntax error is now `unavailable`, and a syntax error in its `percentage` template yields `None` rather than `0`. A `state` template returning `None` now produces `unknown` instead of `off`.

### Template integration extensions

*Source batch: 2025.7.*

Variables, icons, and pictures can now be used across all compatible template platforms. Template alarm control panels, locks, vacuums, and fans can be trigger-based, and `label_description` returns a configured label's description.

### Template integration extensions

*Source batch: 2025.8.*

Trigger-based numeric template sensors can explicitly become `unknown`, and template locks support the `opening` state. Covers, fans, lights, locks, and vacuums can be configured in the UI; availability templates are available there across supported platforms; and alarm panels, fans, lights, locks, switches, and vacuums support all optimistic YAML modes.

### Template name and translation helpers

*Source batch: 2026.4.*

The new `state_attr_translated` template function retrieves translated values for attributes such as fan modes, HVAC actions, and presets. The new `entity_name` function returns an entity's name and is preferred over reading its `friendly_name` attribute directly.

### Template YAML and data helpers

*Source batch: 2025.4.*

Template lights and switches now support the modern YAML style. New template helpers include `combine`, `difference`, `flatten`, `intersect`, `union`, `symmetric_difference`, `shuffle`, `floor_entities`, `typeof`, and the `md5`, `sha1`, `sha256`, and `sha512` hashing functions.

### Timer, Matter, media, and template additions

*Source batch: 2026.7.*

A running timer's duration can now be changed from its dialog, and automation or script traces always include template errors. Matter adds soil-moisture sensors, media players gain a projector device class, template lights gain xy-color support, and Google Assistant can request vacuum cleaning for a specific mapped room.

### YAML and template editor assistance

*Source batch: 2026.5.*

Home Assistant's code editors now provide context-aware YAML and Jinja autocomplete, including signatures and argument placeholders for template functions, filters, tests, and globals. ID arguments suggest matching entities, devices, areas, floors, or labels, while hover details show documentation and current entity or attribute values.

## Assist and voice

### Assist and dashboard additions

*Source batch: 2026.3.*

Assist gains an intent for removing to-do items. Statistics graph cards can select a yearly period, the Security dashboard includes window covers, Sections views support sticky footer cards, and the Matter, Z-Wave, Zigbee, and Bluetooth settings pages have been reorganized.

### Assist Ask Question action

*Source batch: 2025.7.*

`assist_satellite.ask_question` lets an automation initiate a conversation, define expected sentence patterns for local Speech-to-Phrase recognition, and receive the matched answer ID and slots in a response variable. Optional `preannounce` and `preannounce_media_id` fields can precede the question.

```yaml
actions:
  - action: assist_satellite.ask_question
    data:
      entity_id: assist_satellite.living_room_voice_assistant
      question: "What kind of music do you want to listen to?"
      answers:
        - id: genre
          sentences: ["genre {genre}"]
    response_variable: answer
  - choose:
      - conditions: "{{ answer.id == 'genre' }}"
        sequence:
          - action: music_assistant.play_media
            data:
              media_id: "My {{ answer.slots.genre }} playlist"
              media_type: playlist
            target:
              entity_id: media_player.living_room_speakers
```

### Assist reasoning details

*Source batch: 2026.4.*

On the desktop web interface, each LLM-backed Assist response can expose a collapsible detail view containing thinking steps, tool calls, arguments, and results. The mobile companion apps do not yet show this view.

### Broadcast and thermostat voice intents

*Source batch: 2025.2.*

The Broadcast intent sends a spoken message to every other voice assistant, subject to language support. Voice commands can also set a thermostat target temperature, resolving the target by the speaker's area, its floor, or an explicitly named device.

### Default Assist exposure

*Source batch: 2025.1.*

Voice Assistant settings now control whether newly created entities are exposed to Assist by default.

### Default voice-agent intents

*Source batch: 2025.9.*

The default non-LLM agent now uses fuzzy matching for English intent handling. Built-in intents can also change the volume of active media players and control fan speeds.

### Home Assistant Labs

*Source batch: 2025.12.*

**Settings > System > Labs** now contains optional preview features that are off by default and may change or disappear later. A feature can be enabled with an optional backup and disabled again without restarting Home Assistant.

### Home Assistant OS log-file removal

*Source batch: 2025.11.*

Home Assistant OS no longer duplicates Core logs into the configuration-folder log file. Logs remain viewable and downloadable under **Settings > System > Logs** and accessible through `ha core logs`.

### Music Assistant action responses

*Source batch: 2025.1.*

Music Assistant integration actions can now return response values for use by the calling automation or script.

### Sentence-trigger context

*Source batch: 2025.1.*

Sentence triggers now receive the full conversation input, allowing their automations to use more than the matched sentence alone.

### Shopping-list completion intents

*Source batch: 2025.7.*

Shopping-list voice intents can now check off or mark list items complete.

### Streaming Assist chat

*Source batch: 2025.3.*

LLM-backed conversation agents now stream responses into Assist text chat. Commands can execute as soon as they arrive instead of waiting for the rest of the response to finish.

### Voice wake words and local confirmations

*Source batch: 2025.10.*

ESPHome-based voice assistants can assign two wake words, each to its own assistant, enabling per-language or local-versus-cloud routing on one satellite. For non-AI Assist agents, a command whose actions all affect the satellite's own area now produces a short beep instead of a spoken confirmation.

### Voice-controlled area cleaning

*Source batch: 2026.4.*

The `vacuum.clean_area` capability introduced in 2026.3 can now be invoked by voice, allowing a request such as cleaning a named room to use its mapped vacuum segments.

## AI tasks and conversation providers

### AI conversation diagnostics

*Source batch: 2025.12.*

The voice-assistant debug interface now shows an AI conversation's system prompt and tool calls, allowing its entity selection and actions to be audited from the voice-assistant configuration panel.

### AI Task image generation

*Source batch: 2025.10.*

Capable AI Task entities can use `ai_task.generate_image` with optional source-media attachments; the response variable exposes the generated asset through its `url` field.

```yaml
actions:
  - action: ai_task.generate_image
    data:
      task_name: Manga
      instructions: Transform this image into a cute manga.
      entity_id: ai_task.google_ai_task
      attachments:
        media_content_id: media-source://media_source/local/doorbell_test.png
        media_content_type: image/png
    response_variable: ai_image
# Generated asset path: {{ ai_image.url }}
```

### AI Task structured generation

*Source batch: 2025.8.*

An AI provider's AI Task sub-entry creates an entity for `ai_task.generate_data`, which can send files or camera images to the provider and return either text or selector-defined structured data to automations, scripts, and template entities.

```yaml
actions:
  - action: ai_task.generate_data
    data:
      task_name: Count chickens
      instructions: How many birds are inside the coop?
      structure:
        birds:
          selector:
            number:
      attachments:
        media_content_id: media-source://camera/camera.chicken_coop
        media_content_type: image/jpeg
    response_variable: result
# Structured output is available as result.data.birds
```

### Assist-satellite calls and conversations

*Source batch: 2025.2.*

An analog phone configured as an Assist satellite can be called with `assist_satellite.announce`. The new `assist_satellite.start_conversation` action lets an LLM-based agent call and begin a conversation with its first message; the default conversation agent cannot yet start this flow.

### Conversation and YAML integration options

*Source batch: 2025.7.*

Google Generative AI now defaults to Gemini 2.5 Flash and adds configurable text-to-speech with 30 voices across 24 languages. Ollama adds control of its `think` parameter, and Trend YAML configuration accepts unique IDs.

### Conversation-agent tools

*Source batch: 2025.4.*

The OpenAI conversation integration adds a content-generation action and optional web search, while the Google AI conversation integration also gains web search.

### Conversation-provider updates

*Source batch: 2026.3.*

OpenAI Conversation supports `gpt-image-1.5` for AI Task image generation. Anthropic supports Claude Opus 4.6 with adaptive thinking effort and provides native structured outputs on models 4.5 and newer.

### Default AI Task entity and suggestions

*Source batch: 2025.8.*

**Settings > System > General** can select a default AI Task entity, allowing `ai_task.generate_data` calls and shared blueprints to omit an entity. With a default selected and **AI suggestions** enabled, automation and script save dialogs can suggest names, descriptions, categories, and labels; using it sends the full automation or script plus other automation, script, and label names to the configured model.

### LLM calendar access and fallback history

*Source batch: 2025.2.*

LLM-based conversation agents can retrieve today's and this week's events from calendar entities that have been exposed to Assist. The default local agent and its LLM fallback now share command history, so a fallback can resolve references from earlier locally handled commands.

### LLM voice conversation flow

*Source batch: 2025.4.*

Assist now detects when an LLM response contains a question and keeps listening for the answer without requiring the wake word again. LLM-backed conversations can also be started on ESPHome-based voice assistants, extending the earlier `assist_satellite.start_conversation` capability beyond analog phones.

### Model Context Protocol integrations

*Source batch: 2025.2.*

Home Assistant can act as both an MCP client, importing tools from MCP servers for conversation agents, and an MCP server, exposing Home Assistant context to external MCP clients.
