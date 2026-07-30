# Models, endpoints, and lifecycle operations

Source batches: `gemini-3.6` and `release-lifecycle`.

Use this reference when selecting or pinning an endpoint, migrating generation
behavior, planning a shutdown, or implementing current multimodal and
long-running capabilities.

## Gemini 3.6 production behavior

The GA Interactions model IDs are `gemini-3.6-flash` and
`gemini-3.5-flash-lite`. Their default thinking levels are `medium` and
`minimal`, respectively. Both provide a 1M-token context window, a 64k-token
maximum output, and native Computer Use support.

```python
interaction = client.interactions.create(
    model="gemini-3.6-flash",
    input="Describe this image.",
)
```

For these models, `temperature`, `top_p`, and `top_k` are deprecated and ignored;
remove them because later model generations will reject them with HTTP 400.
When moving to 3.6 Flash, replace `thinking_budget` with string-valued
`thinking_level` (`"medium"` or `"high"`). Remove `candidate_count`, which
Gemini 3.x does not support.

A request whose last non-empty turn has role `model` returns HTTP 400. Do not
prefill a partial model answer to steer completion. Put formatting and
preamble-suppression requirements in `system_instruction` or `response_format`.

```python
interaction = client.interactions.create(
    model="gemini-3.6-flash",
    input="Translate 'Hello world' to Spanish.",
    system_instruction="Output only the translation.",
)
```

The managed agent `antigravity-preview-05-2026` uses 3.6 Flash by default and
can run through Interactions in the remote environment.

```python
interaction = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="Complete the browser task.",
    environment="remote",
)
```

When using legacy `generateContent` with Gemini 3.x, include both `call_id` and
function `name` in every `FunctionResponse`.

## Lifecycle metadata and moving aliases

Some model records publish both lifecycle stage and deprecation timeline. Treat
aliases as mutable and pin concrete IDs when reproducibility matters:

- `gemini-pro-latest` and `gemini-flash-latest` moved to Gemini 3 previews in
  January 2026.
- `gemini-3-pro-preview` redirected to `gemini-3.1-pro-preview` in March.
- `gemini-flash-latest` moved again to `gemini-3.5-flash` in May.

`gemini-3.5-flash` is GA. `gemini-3.1-pro-preview-customtools` is a distinct
3.1 Pro endpoint tuned to prioritize custom tools such as bash.
`gemini-3.1-flash-lite` is GA but has a May 7, 2027 shutdown date in favor of
`gemini-3.5-flash-lite`.

## Scheduled migrations after July 28, 2026

Plan for these concrete cutoffs:

- Move the Embedding 2 preview to GA `gemini-embedding-2` by August 10, 2026.
- Move Imagen 4 Standard, Ultra, and Fast (`imagen-4.0-*-generate-001`) to
  `gemini-3.1-flash-image` by August 17, 2026.
- Leave `gemini-2.5-flash-image` by October 2, 2026.
- Move `gemini-2.5-pro`, `gemini-2.5-flash`, and
  `gemini-2.5-flash-lite` by October 16, 2026. Their listed successors are
  `gemini-3.1-pro-preview`, `gemini-3.6-flash`, and
  `gemini-3.1-flash-lite`, respectively.
- Move `gemini-3.1-flash-lite` to `gemini-3.5-flash-lite` by May 7, 2027.
- Move `gemini-embedding-001` to `gemini-embedding-2` by May 14, 2028.

## Endpoints already unavailable

The Gemini 2.0 Flash and Flash-Lite stable IDs shut down June 1, 2026. The 3.1
Flash Image and 3 Pro Image preview IDs shut down June 25 in favor of their GA
IDs. Stable Veo 2.0 and 3.0 generation IDs shut down June 30 in favor of Veo 3.1
preview endpoints or the enterprise GA endpoints.

## Image, video, embeddings, and File Search

The GA native-image IDs are:

- `gemini-3.1-flash-image`;
- `gemini-3-pro-image`;
- lower-latency `gemini-3.1-flash-lite-image`.

Only `gemini-3.1-flash-image` accepts video context for image generation.
`gemini-omni-flash-preview` instead generates 3–10 second 720p video from text
or a still image and supports conversational edits to that video.

GA `gemini-embedding-2` embeds text, images, video, audio, and PDFs into one
space. File Search can use it to index and search images. Visual grounding
citations expose `media_id` and `page_numbers`.

External-file inputs can come from Cloud Storage buckets or public/private
presigned URLs. The per-file limit for these inputs increased from 20 MB to
100 MB.

## Live audio, speech, and session continuity

`gemini-3.1-flash-live-preview` is the current 3.1 audio-to-audio preview.
`gemini-3.1-flash-tts-preview` is the current TTS preview. TTS output can stream
through `streamGenerateContent` or through Interactions with `stream: true`.

Live sessions can keep server-side state for up to 24 hours and return a
`session_resumption` handle. Sliding-window context compression extends long
sessions, while `GoAway` warns before disconnect. Automatic VAD can be tuned or
disabled in favor of `activityStart` and `activityEnd`. Separate controls cover
interruption, turn coverage, media resolution, streamed text, and
modality-level `usageMetadata`.

## Tools, Computer Use, and research agents

A single request can combine built-in tools with custom functions. Computer Use
is also in public preview on `gemini-3.5-flash`, with browser, mobile, and
desktop environments plus configurable safety and prompt-injection controls.

New Deep Research agent variants add collaborative planning, visualization, MCP
server integration, and File Search. They can stream results to a client UI or
run a more comprehensive automated research path.

## Long-running and operational work

Batch API jobs and other long-running operations support event-driven
completion, so integrations can replace polling. Batch processing also supports
embedding-model requests.

In the v1beta Interactions API, `total_reasoning_tokens` was renamed to
`total_thought_tokens`. Supported Interactions calls expose developer logs for
operational inspection.

## Tuning removal

The final tunable model, Gemini 1.5 Flash 001, shut down in May 2025. The API no
longer supports tuning on any model.
