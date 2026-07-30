---
name: openai-api-knowledge-patch
description: OpenAI API
version: null
license: MIT
metadata:
  author: Nevaberry
---


# OpenAI API

Use this skill when implementing, reviewing, or migrating API integrations.
Identify the endpoint, model ID, storage policy, service tier, and transport
before choosing fields or event handlers. Treat returned output items as the
protocol record: preserve every item required by the next turn instead of
reconstructing only assistant text.

## Reference index

| Reference | Read when working on |
| --- | --- |
| [responses-and-state.md](references/responses-and-state.md) | Responses requests, chaining, storage, reasoning replay, streaming, safety |
| [models-and-prompt-caching.md](references/models-and-prompt-caching.md) | Model selection, reasoning modes, context limits, media detail, prompt caching |
| [tools-and-structured-output.md](references/tools-and-structured-output.md) | Function/custom tools, schemas, parse helpers, call loops, tool search, streaming |
| [release-lifecycle.md](references/release-lifecycle.md) | Shutdown dates, replacements, alias stability, API and platform migrations |
| [service-tiers-and-realtime.md](references/service-tiers-and-realtime.md) | Flex, Priority, realtime voice/translation/transcription, credentials and events |

## Working method

1. Pin behavior-sensitive production workloads to dated model IDs; use moving
   aliases only when regularly changing behavior is intentional.
2. Check [release-lifecycle.md](references/release-lifecycle.md) before choosing
   any older API, platform feature, model family, image model, or fine-tuning
   base.
3. Prefer Responses for reasoning-plus-tools, Pro reasoning, persisted response
   chains, and new stateful assistant integrations.
4. Preserve output item identity and ordering across turns. Keep item IDs,
   `call_id`, caller metadata, phase values, tool-search records, reasoning
   items, and all other returned items required by the chosen workflow.
5. Set cost-, latency-, state-, and safety-sensitive defaults explicitly.
   Inspect effective values returned by the API where available.
6. Branch on output item type before parsing. Refusals, messages, function
   calls, program items, and agent items have different contracts.
7. Keep stable prompt prefixes byte-identical. Apply explicit cache boundaries
   only after choosing a compatible model and block type.
8. Test streaming integrations against named GA events and aggregate deltas by
   their identifiers rather than arrival assumptions.

## Breaking changes and urgent migrations

### Move stateful assistant applications

The Assistants API shuts down on August 26, 2026. Move persistent assistant
integrations to Responses plus Conversations. Responses generates one
candidate per request and has no `n`; issue separate requests for multiple
candidates.

Do not assume `previous_response_id` carries top-level `instructions`. Resend
stable instructions on every request, and account for prior chain tokens being
billed again as input.

### Remove retired beta and platform interfaces

- Remove the `OpenAI-Beta: realtime=v1` integration; use the released Realtime
  interface and its GA session and event shapes.
- Move reusable prompts into application code before `v1/prompts` shuts down
  on November 30, 2026.
- Move Agent Builder workflows to the Agents SDK or Workspace Agents by
  November 30, 2026; ChatKit remains available.
- Export or replace Evals before the dashboard, API, and documented graders
  shut down on November 30, 2026. Existing Evals become read-only October 31.
- Do not start a Videos migration on the assumption that a replacement exists;
  the listed Videos and Sora removals provide none.

Read [release-lifecycle.md](references/release-lifecycle.md) for every dated
model, fine-tuning, image, audio, realtime, and platform replacement.

### Recheck function strictness

Responses function definitions try strict mode when `strict` is omitted. An
incompatible schema falls back to best effort and reports `strict: false`.
Set `strict: false` explicitly when non-strict behavior is required; validate
schemas when strict guarantees are required.

Chat Completions function tools on the GPT-5.6 family require effective
reasoning `none`. Because omitted reasoning defaults to `medium`, set
`reasoning_effort: "none"` explicitly or move reasoning-plus-tools work to
Responses.

```json
{
  "model": "gpt-5.6-luna",
  "reasoning_effort": "none",
  "tools": [{
    "type": "function",
    "function": {
      "name": "lookup",
      "parameters": {"type": "object", "properties": {}}
    }
  }]
}
```

## Responses quick reference

### Choose state behavior deliberately

Responses are stored by default; Chat Completions are also stored by default
for new accounts. Set `store: false` for stateless use. In a stateless
reasoning loop, replay every returned reasoning item with its default
`encrypted_content`. Zero Data Retention disables storage automatically.

Use `reasoning.context: "all_turns"` with `previous_response_id` only while the
goal and assumptions remain stable. Choose `current_turn` when earlier
reasoning is stale. With `auto` or omission, inspect the returned effective
context value.

### Stream with the correct transport

HTTP `stream=true` uses server-sent events. Persistent WebSocket mode accepts
incremental inputs chained through `previous_response_id`. Moderation scores
requested with generation arrive only after the complete output, never with
partial deltas.

Generation-time cyber or biology review can refuse output or pause a stream
for seconds. Attach a stable, privacy-preserving `safety_identifier` to each
Responses request.

## Model and reasoning quick reference

| Need | Select |
| --- | --- |
| Flagship capability | `gpt-5.6` alias or `gpt-5.6-sol` |
| Balanced lower cost | `gpt-5.6-terra` |
| Efficient high volume | `gpt-5.6-luna` |
| Responses-only extended reasoning | Normal family model plus `reasoning.mode: "pro"` |

Sol and Terra accept roughly 1.05M input tokens; Luna accepts 400K. All three
allow up to 128K output tokens. Sol and Terra requests above 272K input tokens
enter different full-request pricing.

The family accepts reasoning efforts `none`, `low`, `medium`, `high`, `xhigh`,
and `max`; omission defaults to `medium` in standard and Pro modes. Responses
uses `reasoning: {"effort": "..."}`. Chat Completions uses
`reasoning_effort: "..."`. Preserve effective effort during endpoint migration
before tuning it.

Pro is a Responses-only mode, not a separate model slug. Mode and effort are
independent, and supported Pro efforts begin at `medium`.

```json
{
  "model": "gpt-5.6-sol",
  "reasoning": {"mode": "pro", "effort": "medium"}
}
```

Set image or file detail explicitly when token cost and latency matter.
Omitted or `auto` image detail can retain original dimensions; Responses PDF
inputs can use high-detail page images. Chat Completions file inputs do not
offer the same detail control.

## Tools and structured output quick reference

Place raw Responses output formats under `text.format`. In SDK parse helpers,
pass a Pydantic model as Python `text_format` or a Zod format as JavaScript
`text.format`. Inspect message content items before using parsed data because a
safety refusal arrives as its own `refusal` item.

For each function call:

1. Preserve all response output items in the next input.
2. Parse the JSON-encoded `arguments`.
3. Run the named function.
4. Return a `function_call_output` with the unchanged `call_id`.
5. Encode the normal result as a string, or supply the supported image/file
   object array.

Keep the declared tool list stable and use an `allowed_tools` `tool_choice` to
restrict a turn without disturbing prompt-cache reuse. Remember that with tool
search the restriction applies only to tools already loaded for that turn.

Do not combine built-in tools with parallel function calling. Set
`parallel_tool_calls: false` to permit at most one call. Read
[tools-and-structured-output.md](references/tools-and-structured-output.md)
before enabling parallel calls, deferred loading, programmatic calling,
multi-agent Responses, custom grammars, or function-call streaming.

## Prompt caching quick reference

For GPT-5.6-family caching, set a stable `prompt_cache_key` and preserve the
exact prefix through the selected breakpoint. Shard busy traffic while keeping
identical prefixes on the same key; keep aggregate traffic sharing one key
near 15 requests per minute.

Use `prompt_cache_options: {"mode": "explicit", "ttl": "30m"}` with
`prompt_cache_breakpoint` for measured boundaries. The TTL is a minimum and
`30m` is the only supported value. An explicit-mode request with no markers
disables caching and writes. Do not send these fields to pre-GPT-5.6 models.

Cache writes are reported as `cache_write_tokens` and cost 1.25 times uncached
input. Read [models-and-prompt-caching.md](references/models-and-prompt-caching.md)
for write limits, lookup rules, compatible blocks, older retention policies,
and Zero Data Retention defaults.

## Service tier and realtime quick reference

Flex uses Batch API token rates but can run beyond the SDK's ten-minute default
timeout. Increase the timeout when appropriate. Retry its uncharged
`429 Resource Unavailable` with exponential backoff, or fall back to
`service_tier: "auto"` or the project default.

Priority is for steady, latency-sensitive traffic. Inspect the response
`service_tier`: project defaults roll out gradually, and ramp-limited Priority
requests may execute as `default` at Standard rates.

For realtime work, select the dedicated endpoint and lifecycle:

| Workload | Model and endpoint |
| --- | --- |
| Stateful voice agent | `gpt-realtime-2.1` on `/v1/realtime` |
| Continuous translation | `gpt-realtime-translate` on `/v1/realtime/translations` |
| Live transcript deltas | `gpt-realtime-whisper` |

Translation begins without `response.create` or a client-committed user turn.
Use the `OpenAI-Safety-Identifier` header for realtime sessions; a Responses
`safety_identifier` does not carry over. Read
[service-tiers-and-realtime.md](references/service-tiers-and-realtime.md) for
credential setup, GA WebRTC configuration, event names, and tier constraints.
