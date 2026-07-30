# Release Lifecycle and Migrations

## Interpret lifecycle states and pin behavior

Deprecation begins when announced and includes a shutdown date. `legacy` means
the API or model no longer receives updates and is likely to be deprecated
later, but does not itself supply a shutdown date.

Unless safety or compliance requires faster action, generally available models
normally receive at least six months' notice; specialized chat, Codex, and
deep-research variants receive three months; preview models may receive only
about two weeks.

`chat-latest` tracks the regularly updated Instant model used in ChatGPT. Use
it to test that changing behavior, not as a stable production pin. Unversioned
audio, realtime, transcription, and Sora aliases have also moved between dated
snapshots. Use dated IDs wherever behavior must remain pinned.

## Migrate APIs and application platforms

### Realtime beta

The `OpenAI-Beta: realtime=v1` interface was removed May 12, 2026. Use the
released Realtime API and update the integration for its different interface.

### Assistants

The Assistants API shuts down August 26, 2026. Migrate persistent assistants
to the Responses API plus the Conversations API.

### Videos

The Videos API, all listed `sora-2` aliases and snapshots, and all listed
`sora-2-pro` aliases and snapshots shut down September 24, 2026. No replacement
is listed.

### Reusable prompts

Reusable prompt objects and `v1/prompts` shut down November 30, 2026. Move
prompt content into application code instead of creating or referencing
prompt objects.

### Evals

Existing Evals become read-only October 31, 2026. The Evals dashboard, API, and
documented graders shut down November 30. The documented migration path uses
Promptfoo. Fine-tuning follows a separate schedule.

### Agent Builder

Agent Builder shuts down November 30, 2026. Move workflows to the Agents SDK
or ChatGPT Workspace Agents. ChatKit remains available.

## Migrate chat, completion, and reasoning models

### July 23 replacement wave

The July 23 shutdown covered:

- computer-use and GPT-4o search previews;
- `gpt-5-chat-latest` and `gpt-5.1-chat-latest`;
- GPT-5, GPT-5.1, and GPT-5.2 Codex variants;
- o3 and o4 deep-research models.

Use `gpt-5.6-terra` for computer/search and mini-Codex workloads. Use
`gpt-5.6-sol` for the other affected chat, Codex, and research IDs.

### Chat-latest snapshots

`gpt-5.2-chat-latest` and `gpt-5.3-chat-latest` shut down August 10, 2026.
Migrate both to `gpt-5.6-sol`.

### Older completion and chat models

These models shut down September 28, 2026:

- `gpt-3.5-turbo-instruct`
- `babbage-002`
- `davinci-002`
- `gpt-3.5-turbo-1106`

Their listed replacements are `gpt-5.4-mini` or `gpt-5-mini`.

### GPT-3.5, GPT-4, GPT-4.1, and o-series models

The following shut down October 23, 2026:

- `gpt-3.5-turbo-0125` aliases;
- GPT-4 and GPT-4 Turbo aliases and snapshots;
- `gpt-4o-2024-05-13`;
- `gpt-4.1-nano`;
- `o1`, `o1-pro`, `o3-mini`, and `o4-mini`.

Map workloads to the corresponding GPT-5.6 tier:

| Removed workload | Replacement |
| --- | --- |
| GPT-4, GPT-4o, o1, o3 | `gpt-5.6-sol` |
| `o1-pro` | `gpt-5.6-sol` with `reasoning.mode: "pro"` |
| GPT-3.5 and o4-mini | `gpt-5.6-terra` |
| GPT-4.1 nano | `gpt-5.6-luna` |

### GPT-5 and o3 dated snapshots

These snapshots shut down December 11, 2026:

- `gpt-5-2025-08-07`, `gpt-5-mini-2025-08-07`,
  `gpt-5-nano-2025-08-07`, and `gpt-5-pro-2025-10-06`;
- `o3-2025-04-16` and `o3-pro-2025-06-10`.

Use Sol for GPT-5 and o3, Terra for GPT-5 mini, Luna for GPT-5 nano, and Sol
with `reasoning.mode: "pro"` for the Pro variants.

## Migrate fine-tuning workloads

### Fine-tuned model bases

These fine-tuned model families shut down October 23, 2026:

| Removed family | Replacement base |
| --- | --- |
| `ft-gpt-3.5-turbo` | GPT-5.4 mini |
| `ft-gpt-4` | GPT-5.5 |
| `ft-gpt-4.1-nano-2025-04-14` | GPT-5.4 nano |
| `ft-babbage-002` | GPT-5.4 mini |
| `ft-davinci-002` | GPT-5.4 mini |
| `ft-o4-mini-2025-04-16` | GPT-5.6 Terra |

### Self-serve training

New training is already unavailable to organizations without prior
fine-tuning, and has been unavailable since July 2 to organizations without
fine-tuned-model inference during the preceding 60 days.

Remaining active customers lose the ability to create fine-tuning jobs on
January 6, 2027. Inference continues only until the underlying base model is
deprecated.

## Migrate image generation

`dall-e-2` and `dall-e-3` were removed May 12, 2026. `gpt-image-1` shuts down
October 23. `gpt-image-1-mini`, `gpt-image-1.5`, and
`chatgpt-image-latest` shut down December 1. Move all of these image workloads
to `gpt-image-2`.

## Migrate audio and realtime models

On January 20, 2027:

- move `gpt-realtime` and GPT-4o realtime families to `gpt-realtime-2.1`;
- move their mini variants to `gpt-realtime-2.1-mini`;
- move GPT audio and GPT-4o audio families to `gpt-audio-1.5`;
- move `gpt-4o-mini-transcribe-2025-03-20` to
  `gpt-4o-mini-transcribe-2025-12-15`.
