# Models, Media, and Lifecycle

## Choose current production IDs

The GA Interactions IDs (`gemini-3.6`) are:

| Model | Default thinking | Context | Maximum output |
| --- | --- | --- | --- |
| `gemini-3.6-flash` | `medium` | 1M tokens | 64k tokens |
| `gemini-3.5-flash-lite` | `minimal` | 1M tokens | 64k tokens |

Both provide native Computer Use. For these models, `temperature`, `top_p`, and
`top_k` are deprecated and ignored; remove them because later generations are
expected to reject them with HTTP 400. For 3.6 Flash, replace
`thinking_budget` with string-valued `thinking_level` (`"medium"` or `"high"`).
Gemini 3.x does not support `candidate_count`.

A request whose last non-empty turn has role `model` returns HTTP 400. Do not
prefill a model response. Put formatting or preamble-suppression requirements
in `system_instruction` or `response_format`:

```python
client.interactions.create(
    model="gemini-3.6-flash",
    input="Translate 'Hello world' to Spanish.",
    system_instruction="Output only the translation.",
)
```

## Pin concrete IDs and read lifecycle metadata

Model records can publish lifecycle stage and a deprecation timeline
(`release-lifecycle`). Treat aliases as mutable:

- `gemini-pro-latest` and `gemini-flash-latest` moved to Gemini 3 previews in
  January 2026.
- `gemini-3-pro-preview` redirected to `gemini-3.1-pro-preview` in March 2026.
- `gemini-flash-latest` moved again to `gemini-3.5-flash` in May 2026.

Pin a concrete ID when reproducibility matters.

`gemini-3.5-flash` is GA.
`gemini-3.1-pro-preview-customtools` is a separate 3.1 Pro endpoint tuned to
prioritize custom tools such as bash. `gemini-3.1-flash-lite` is GA, but it has
a May 7, 2027 shutdown date in favor of `gemini-3.5-flash-lite`.

## Complete scheduled migrations

Apply the lifecycle cutoffs after July 28, 2026:

- Move the Embedding 2 preview to GA `gemini-embedding-2` by August 10, 2026.
- Move Imagen 4 Standard, Ultra, and Fast
  (`imagen-4.0-*-generate-001`) to `gemini-3.1-flash-image` by August 17, 2026.
- Leave `gemini-2.5-flash-image` by October 2, 2026.
- Move `gemini-2.5-pro`, `gemini-2.5-flash`, and
  `gemini-2.5-flash-lite` by October 16, 2026. Their listed successors are
  `gemini-3.1-pro-preview`, `gemini-3.6-flash`, and
  `gemini-3.1-flash-lite`, respectively.
- Move `gemini-3.1-flash-lite` to `gemini-3.5-flash-lite` by May 7, 2027.
- Move `gemini-embedding-001` to `gemini-embedding-2` by May 14, 2028.

Do not use recently retired endpoints:

- Gemini 2.0 Flash and Flash-Lite stable IDs shut down June 1, 2026.
- 3.1 Flash Image and 3 Pro Image preview IDs shut down June 25, 2026 in favor
  of their GA IDs.
- Stable Veo 2.0 and 3.0 generation IDs shut down June 30, 2026 in favor of Veo
  3.1 preview endpoints or enterprise GA endpoints.

## Generate images and video

Current GA native-image IDs are:

- `gemini-3.1-flash-image`
- `gemini-3-pro-image`
- lower-latency `gemini-3.1-flash-lite-image`

Only `gemini-3.1-flash-image` accepts video context for image generation.

`gemini-omni-flash-preview` generates 3–10 second, 720p video from text or a
still image and supports conversational edits to the video.

## Embed multimodal content and search files

GA `gemini-embedding-2` embeds text, images, video, audio, and PDFs into a
shared space. File Search can use it to index and search images. Visual
grounding citations expose `media_id` and `page_numbers`.

The API accepts Cloud Storage buckets and public or private presigned URLs as
input sources. The per-file limit for those inputs is 100 MB, increased from
20 MB.

## Stream Live and speech sessions

`gemini-3.1-flash-live-preview` is the current 3.1 audio-to-audio preview.
`gemini-3.1-flash-tts-preview` is the current TTS preview. TTS output can stream
through `streamGenerateContent` or Interactions with `stream: true`.

Live sessions can keep server-side state for up to 24 hours and return a
`session_resumption` handle. Sliding-window context compression extends long
sessions, and `GoAway` warns before disconnect.

Automatic VAD can be tuned or disabled in favor of explicit `activityStart`
and `activityEnd`. Live configuration has separate controls for interruption,
turn coverage, media resolution, streamed text, and modality-level
`usageMetadata`.

## Use event-driven long-running operations

Batch jobs and other long-running operations support event-driven completion,
so integrations can replace polling. Batch processing also accepts
embedding-model requests.

## Account for product availability

The API no longer supports model tuning: the last tunable model, Gemini 1.5
Flash 001, shut down in May 2025.
