# Endpoint State and Lifecycle

Use this reference to migrate endpoint semantics, select stable identifiers, and replace capabilities before shutdown. Responses behavior below is attributed to `responses-api` (2025-03-11); lifecycle dates reflect the release-lifecycle guidance.

## Responses request semantics

### Candidate generation

Responses produces one generation per request and does not accept `n`. Make independent requests when an application needs several candidate outputs.

### Chained context

`previous_response_id` carries earlier response context but does not carry top-level `instructions`. Resend stable instructions with each request. Earlier input tokens brought forward by a response chain remain billed as input tokens.

### Storage and stateless turns

Responses stores results by default. Chat Completions also stores results by default for new accounts. Set `store: false` when stateless use is required. Zero Data Retention flows enforce disabled storage automatically.

Stateless reasoning continuity requires replaying every reasoning Item returned by the API, including its default `encrypted_content`. A complete manual replay also retains every user input and output Item together with item IDs, call IDs, caller metadata, and assistant phase values.

### Transports and moderation timing

HTTP `stream=true` delivers server-sent events. Persistent WebSocket mode accepts incremental inputs chained through `previous_response_id`.

When moderation scores are requested with generation, they arrive only after the full output. Partial deltas do not include them, so a streaming UI must not treat their absence as an approval result.

## Retirement vocabulary and pinning

Deprecation starts when announced and includes a shutdown date. `legacy` means the API or model no longer receives updates and is likely to be deprecated later; it does not itself name a shutdown date.

Unless safety or compliance requires faster action, the general notice windows are:

- At least six months for generally available models.
- Three months for specialized chat, Codex, and deep-research variants.
- Potentially about two weeks for preview models.

`chat-latest` follows the regularly updated Instant model used in ChatGPT and is for testing that changing behavior rather than pinning production behavior. Unversioned audio, realtime, transcription, and Sora aliases have also moved among dated snapshots. Use a dated model ID when behavior must remain stable, while still tracking its shutdown date.

## Endpoint and platform migrations

### Realtime beta

The `OpenAI-Beta: realtime=v1` interface was removed May 12, 2026. Migrate to the released Realtime API and update request, session, and event shapes; the GA interface is not a drop-in beta header change.

### Assistants

The Assistants API shuts down August 26, 2026. Use Responses for generation and tool interactions, plus Conversations for persistent conversation state.

### Videos

The Videos API and every listed `sora-2` and `sora-2-pro` alias and snapshot shut down September 24, 2026. No replacement is listed.

### Reusable prompts

Reusable prompt objects and `v1/prompts` shut down November 30, 2026. Move prompt text and composition into application code instead of creating or referencing prompt objects.

### Evals

Existing Evals become read-only October 31, 2026. The Evals dashboard, API, and documented graders shut down November 30. The documented migration path uses Promptfoo. Fine-tuning is governed by its own schedule.

### Agent Builder

Agent Builder shuts down November 30, 2026. Migrate workflows to the Agents SDK or ChatGPT Workspace Agents. ChatKit remains available.

## Model replacements by workload

### July replacement wave

The July 23 shutdown covered computer-use and GPT-4o search previews, `gpt-5-chat-latest`, `gpt-5.1-chat-latest`, GPT-5/5.1/5.2 Codex variants, and o3/o4 deep-research models.

- Move computer-use, search, and mini-Codex workloads to `gpt-5.6-terra`.
- Move the other affected chat, Codex, and research workloads to `gpt-5.6-sol`.

### Chat snapshots

`gpt-5.2-chat-latest` and `gpt-5.3-chat-latest` shut down August 10, 2026. Replace both with `gpt-5.6-sol`.

### Legacy text models

The following shut down September 28, 2026:

| Retiring ID | Listed replacement |
| --- | --- |
| `gpt-3.5-turbo-instruct` | `gpt-5.4-mini` or `gpt-5-mini` |
| `babbage-002` | `gpt-5.4-mini` or `gpt-5-mini` |
| `davinci-002` | `gpt-5.4-mini` or `gpt-5-mini` |
| `gpt-3.5-turbo-1106` | `gpt-5.4-mini` or `gpt-5-mini` |

### October general-model wave

On October 23, 2026, the `gpt-3.5-turbo-0125` aliases, GPT-4 and GPT-4 Turbo aliases and snapshots, `gpt-4o-2024-05-13`, `gpt-4.1-nano`, `o1`, `o1-pro`, `o3-mini`, and `o4-mini` shut down.

| Retiring workload | Replacement |
| --- | --- |
| GPT-4, GPT-4o, o1, o3 | `gpt-5.6-sol` |
| o1 Pro | `gpt-5.6-sol` with `reasoning.mode: "pro"` |
| GPT-3.5 and o4-mini | `gpt-5.6-terra` |
| GPT-4.1 nano | `gpt-5.6-luna` |

### GPT-5 and o3 dated snapshots

On December 11, 2026, these snapshots shut down:

- `gpt-5-2025-08-07` and `o3-2025-04-16` move to `gpt-5.6-sol`.
- `gpt-5-mini-2025-08-07` moves to `gpt-5.6-terra`.
- `gpt-5-nano-2025-08-07` moves to `gpt-5.6-luna`.
- `gpt-5-pro-2025-10-06` and `o3-pro-2025-06-10` move to `gpt-5.6-sol` with `reasoning.mode: "pro"`.

## Fine-tuning migration

### Retiring fine-tuned bases

The October 23, 2026 wave removes these fine-tuned model families:

| Retiring family | Replacement base |
| --- | --- |
| `ft-gpt-3.5-turbo` | GPT-5.4 mini |
| `ft-gpt-4` | GPT-5.5 |
| `ft-gpt-4.1-nano-2025-04-14` | GPT-5.4 nano |
| `ft-babbage-002` | GPT-5.4 mini |
| `ft-davinci-002` | GPT-5.4 mini |
| `ft-o4-mini-2025-04-16` | GPT-5.6 Terra |

### Self-serve wind-down

New training is already unavailable to organizations that had no prior fine-tuning, and has been unavailable since July 2 to organizations without fine-tuned-model inference during the preceding 60 days. Remaining active customers lose the ability to create fine-tuning jobs January 6, 2027. Inference then remains available only until the underlying base model is deprecated.

## Media model consolidation

### Images

`dall-e-2` and `dall-e-3` were removed May 12, 2026. `gpt-image-1` shuts down October 23. `gpt-image-1-mini`, `gpt-image-1.5`, and `chatgpt-image-latest` shut down December 1. Move these image workloads to `gpt-image-2`.

### Audio and realtime

On January 20, 2027:

- `gpt-realtime` and GPT-4o realtime families move to `gpt-realtime-2.1`.
- Their mini variants move to `gpt-realtime-2.1-mini`.
- GPT audio and GPT-4o audio families move to `gpt-audio-1.5`.
- `gpt-4o-mini-transcribe-2025-03-20` moves to `gpt-4o-mini-transcribe-2025-12-15`.

## Migration checklist

1. Inventory every endpoint, header, prompt object, platform dependency, model alias, snapshot, and fine-tuned base.
2. Separate already-removed interfaces from future shutdowns and schedule the earliest blockers first.
3. Replace state and event protocols before changing model IDs.
4. Pin dated IDs where reproducibility is required, but track their retirement dates.
5. Run contract tests for response Items, tool calls, streams, refusals, and persisted state after each migration.
