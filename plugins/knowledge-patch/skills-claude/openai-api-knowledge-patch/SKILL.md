---
name: openai-api-knowledge-patch
description: OpenAI API
version: null
license: MIT
metadata:
  author: Nevaberry
---


# OpenAI API Compatibility

Use this skill when building, migrating, debugging, or operating OpenAI API integrations that involve Responses, current GPT-5.6 models, tool calling, structured output, prompt caching, service tiers, or Realtime.

Treat application manifests, pinned SDK behavior, returned effective fields, and observed API errors as authoritative when they disagree with general guidance. Before changing an integration, identify its endpoint, model ID, storage mode, tool protocol, service tier, and whether behavior must remain pinned to a dated snapshot.

## Reference index

| Reference | Topics |
| --- | --- |
| [endpoint-state-and-lifecycle.md](references/endpoint-state-and-lifecycle.md) | Responses state, storage, streaming, endpoint migration, deprecations, shutdown replacements |
| [models-and-reasoning.md](references/models-and-reasoning.md) | GPT-5.6 tiers, limits, reasoning effort and context, Pro mode, multimodal defaults, safeguards |
| [tools-and-structured-output.md](references/tools-and-structured-output.md) | Function round trips, strictness, parsing, refusals, namespaces, tool search, custom tools, streaming |
| [prompt-caching.md](references/prompt-caching.md) | Implicit and explicit caching, keys, breakpoints, TTL, retention, compatibility |
| [service-tiers.md](references/service-tiers.md) | Flex, Priority, timeouts, retries, capacity, effective tier, ramp limits |
| [realtime.md](references/realtime.md) | Realtime models, translation, safety identifiers, client credentials, WebRTC, GA events |

## Breaking migrations first

### Move Assistants integrations to Responses and Conversations

The Assistants API shuts down on August 26, 2026. Represent generation and tools with Responses, and move durable conversation state to the Conversations API. Do not assume Assistants threads, runs, or beta event shapes transfer unchanged.

### Remove retired beta and product interfaces

- The `OpenAI-Beta: realtime=v1` interface was removed on May 12, 2026; use the released Realtime API and its GA shapes.
- The Videos API and listed `sora-2`/`sora-2-pro` IDs shut down September 24, 2026 without a listed replacement.
- Reusable prompt objects and `v1/prompts` shut down November 30, 2026; keep prompt content in application code.
- Agent Builder shuts down November 30, 2026; migrate workflows to the Agents SDK or ChatGPT Workspace Agents. ChatKit remains available.
- Existing Evals become read-only October 31, 2026; the dashboard, API, and documented graders shut down November 30. The documented replacement is Promptfoo.

See [endpoint-state-and-lifecycle.md](references/endpoint-state-and-lifecycle.md) for the complete capability-organized replacement map, including model and fine-tuning shutdowns.

### Pin behavior-sensitive production models

Moving aliases such as `chat-latest` track changing behavior. Unversioned audio, realtime, transcription, and Sora aliases can also move between dated snapshots. Use dated IDs where reproducibility matters and consult the lifecycle reference before retaining an older snapshot.

## Responses request and state rules

### Expect one generation

Responses has no `n` parameter and returns one generation per request. Issue separate requests for multiple candidates.

### Carry state deliberately

`previous_response_id` carries prior response context but not top-level `instructions`. Resend stable instructions on every request. Chained earlier input tokens remain billable input.

Responses are stored by default, as are Chat Completions for new accounts. Set `store: false` for stateless operation. In stateless reasoning flows, replay every returned reasoning Item with its default `encrypted_content`; Zero Data Retention disables storage automatically.

For full manual replay, preserve all user inputs and output items plus item IDs, call IDs, caller metadata, and assistant phase values.

### Stream with the correct transport assumptions

HTTP `stream=true` uses server-sent events. Persistent WebSocket Responses supports incremental inputs chained with `previous_response_id`. Generation-time moderation scores arrive only after complete output, never with partial deltas.

## GPT-5.6 selection and reasoning

### Choose a family tier

| ID | Intended use | Input context | Maximum output |
| --- | --- | --- | --- |
| `gpt-5.6` / `gpt-5.6-sol` | Flagship | about 1.05M | 128K |
| `gpt-5.6-terra` | Balanced, lower cost | about 1.05M | 128K |
| `gpt-5.6-luna` | Efficient, high volume | 400K | 128K |

Requests to Sol or Terra above 272K input tokens enter different full-request pricing.

### Set reasoning explicitly during migration

Supported effort values are `none`, `low`, `medium`, `high`, `xhigh`, and `max`; omission defaults to `medium` in standard and Pro modes.

```json
// Responses
{"reasoning":{"effort":"none"}}
```

```json
// Chat Completions
{"reasoning_effort":"none"}
```

Preserve the old effective effort first, then tune. Chat Completions function tools require effective effort `none`; its default `medium` is incompatible. Set `reasoning_effort` or move reasoning-plus-tools work to Responses.

### Scope persisted reasoning

Use `reasoning.context: "all_turns"` with `previous_response_id` only while goals and assumptions remain stable. Choose `current_turn` when earlier reasoning is stale, or use `auto`/omission and inspect the returned effective value.

Pro is a Responses-only mode on a normal family ID, not a separate slug. Mode and effort are independent, and supported Pro efforts begin at `medium`.

```json
{
  "model": "gpt-5.6-sol",
  "reasoning": {"mode": "pro", "effort": "medium"}
}
```

## Tool and structured-output essentials

### Complete every function-call round trip

Each Responses `function_call` has `name`, JSON-encoded `arguments`, and `call_id`. Preserve all response output items, execute calls, and append `function_call_output` with the matching `call_id`. Its `output` is usually a string but can be an array of image or file objects.

```python
input_items += response.output
for call in response.output:
    if call.type == "function_call":
        input_items.append({
            "type": "function_call_output",
            "call_id": call.call_id,
            "output": json.dumps(run(call.name, json.loads(call.arguments))),
        })
```

### Handle strictness and refusals explicitly

For Responses functions, omitting `strict` attempts strict mode. An incompatible schema falls back to best effort and reports `strict: false`; set `strict: false` when non-strict behavior is intentional.

Raw structured formats belong under `text.format`. SDK parse helpers use Python `text_format` for Pydantic and JavaScript `text.format` for Zod. Inspect message content Items: safety refusals are separate `refusal` items, not schema-shaped parsed data.

### Keep tool protocol items in history

Deferred functions produce `tool_search_call` and `tool_search_output` before the eventual function call. Programmatic Tool Calling introduces `program`, program-issued `function_call`, `function_call_output`, and `program_output`. Multi-agent Responses adds `multi_agent_call`, `multi_agent_call_output`, and `agent_message`, including function calls issued by subagents. Preserve and replay every applicable Item.

The multi-agent beta requires both `OpenAI-Beta: responses_multi_agent=v1` and bounded `multi_agent.max_concurrent_subagents`.

## Prompt caching essentials

For GPT-5.6 and later, set `prompt_cache_key` for the more reliable matching path. Exact rendered prefixes must still match. Keep aggregate traffic across prefixes sharing one key near 15 RPM and shard busier traffic with a stable mapping that keeps identical prefixes on the same key.

Implicit caching places a managed breakpoint near the latest user or tool message, so changing suffixes can displace stable prefixes. Use `prompt_cache_options` and `prompt_cache_breakpoint` for a measured stable boundary.

```json
{
  "prompt_cache_key": "tenant:acme:kb-v1",
  "prompt_cache_options": {"mode": "explicit", "ttl": "30m"}
}
```

`prompt_cache_options.ttl` replaces deprecated `prompt_cache_retention` on the newer family. `30m` is the only supported value and is a minimum lifetime. Cache writes are reported as `cache_write_tokens` and cost 1.25 times uncached input.

## Service-tier essentials

Flex uses Batch API token rates on Responses and Chat Completions while retaining cache discounts. SDK calls default to a ten-minute timeout and retry `408 Request Timeout` twice, so long jobs may need a larger timeout.

A Flex capacity shortage returns uncharged `429 Resource Unavailable`. Back off to preserve Flex pricing, or retry with `service_tier: "auto"`/omission to use the project default.

Priority can be selected per request or gradually enabled as the project default. Always inspect the response `service_tier` to learn what actually processed a call. Priority shares Standard rate limits; sharp ramps above one million TPM can fall back to `default` and Standard billing.

Priority supports prompt-cache discounts and image inputs, but not long context, fine-tuned models, or embeddings.

## Realtime essentials

- Use `gpt-realtime-2.1` on `/v1/realtime` for stateful voice agents; Realtime 2 voice sessions expose `reasoning.effort`.
- Use `gpt-realtime-translate` on `/v1/realtime/translations` for continuous translation and `gpt-realtime-whisper` for live transcript deltas.
- Translation begins without `response.create` or a client turn commit and does not use the normal assistant-turn lifecycle.
- Realtime carries the safety ID in `OpenAI-Safety-Identifier`, not the Responses `safety_identifier` parameter.
- Obtain browser/mobile ephemeral credentials from `POST /v1/realtime/client_secrets`; use `/v1/realtime/calls` for GA WebRTC setup.
- GA configuration sets `session.type`, nests output settings below `session.audio.output`, and streams `response.output_text.delta`, `response.output_audio.delta`, and `response.output_audio_transcript.delta`.

## Implementation workflow

1. Inventory endpoints, model IDs, aliases, storage settings, state links, tools, caching fields, and service tiers.
2. Check every dated or moving ID against [endpoint-state-and-lifecycle.md](references/endpoint-state-and-lifecycle.md).
3. Migrate state and tool protocols before tuning reasoning, caching, latency, or cost.
4. Preserve response Items and identifiers exactly; do not flatten protocol history to message text.
5. Test refusals, stream pauses, capacity failures, cache misses, tool concurrency, and effective tier fields.
6. Pin behavior-sensitive IDs, set cost-sensitive multimodal detail explicitly, and attach a stable privacy-preserving safety identifier.
