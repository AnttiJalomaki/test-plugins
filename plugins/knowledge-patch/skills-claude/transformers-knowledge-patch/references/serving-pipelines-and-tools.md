# Serving, pipelines, and tools

## Run the local server

`transformers serve` is a local, cross-modality serving utility introduced in
4.54.0 for experimentation and private use. It is separate from a production
serving platform. The initial OpenAI-compatible surface includes:

- `POST /v1/chat/completions`
- `POST /v1/responses`
- `POST /v1/audio/transcriptions`
- `GET /v1/models`

`transformers chat` and compatible clients can connect to the same server.
Continuous batching was integrated into the server in 4.57.0.

The 5.6.0 server adds:

- the legacy `POST /v1/completions` endpoint;
- audio and video request inputs;
- `--compile` and `--model-timeout` options;
- forwarding of `tool_calls` and `tool_call_id` to processor inputs;
- `parse_response` handling for tool calls; and
- HTTP 400 when the request's model differs from the server's pinned model.

In 5.9.0, `GET /v1/models` returns `owned_by` as a string rather than the
previous erroneous list. Continuous batching gains tensor parallelism, fixes
request offsets, and restores `_attn_implementation`; its OpenTelemetry
integration is removed.

## Use the chat CLI

The simplified entry point is `transformers chat MODEL` as of 4.52.1.
Generation settings follow the model as `GenerationConfig`-style `key=value`
arguments instead of a closed list of flags.

```bash
transformers chat Qwen/Qwen2.5-0.5B-Instruct do_sample=False max_new_tokens=10
```

## Migrate pipeline task names and defaults

- The `image-text-to-text` pipeline accepts post-processing keyword arguments
  beginning in 4.50.0.
- Pipeline `dtype` defaults to `auto` from 4.53.0.
- Image-text inference supports batches larger than one in 4.57.0.
- The v5 pipeline cleanup changes or removes the `question-answering`,
  `visual-question-answering`, and `image-to-image` tasks in 5.3.0. Migrate
  callers to the replacement pipeline or updated task name and verify output
  schemas.
- EoMT image segmentation is pipeline-compatible from 4.54.0.

## Use multimodal pipelines

Gemma 3n supports text, image, video, and audio input with text output in
4.53.0. The `image-text-to-text` pipeline accepts an image URL and a prompt with
the `<image_soft_token>` placeholder.

```python
from transformers import pipeline

pipe = pipeline("image-text-to-text", model="google/gemma-3n-e4b")
output = pipe(
    "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/bee.jpg",
    text="<image_soft_token> in this image, there is",
)
```

## Inspect attention layouts

`AttentionMaskVisualizer` loads a tokenizer and model ID and renders the
resulting mask layout, including sliding-window and multimodal masks (4.50.0).

```python
from transformers.utils.attention_visualizer import AttentionMaskVisualizer

visualizer = AttentionMaskVisualizer("meta-llama/Llama-3.2-3B-Instruct")
visualizer("A normal attention mask")
```

Use the visualizer to check whether prompt tokens, media tokens, or sliding
windows receive the intended visibility before investigating numerical output.

## Monitor speech processing

- `Speech2TextFeatureExtractor` exposes dithering in its API as of 4.50.0.
- Whisper transcription accepts a progress callback in 4.55.0.
- Whisper word timestamps changed in 4.54.0: a timestamp token marks the end of
  the token's time span.
- `WhisperFeatureExtractor` aligns `input_features` and `attention_mask` lengths
  in 4.57.0.
- The ASR pipeline's `num_frames` output entry is removed in 5.1.0.

## Visualize keypoint matching

Use `visualize_keypoint_matching` for keypoint correspondences. The older
`plot_keypoint_matching` name is deprecated as of 4.55.0.

LightGlue is useful with SuperPoint for feature matching, pose estimation, and
homography estimation (4.53.0). The native LightGlue integration in 5.5.0 no
longer supports remote-code execution; remove `trust_remote_code=True` and load
it through the standard API.

## Execute custom generation safely

Hub-hosted custom generation is available in 4.52.1 through
`custom_generate=...` plus `trust_remote_code=True`. Since repository code runs
inside the process, inspect it, pin its revision, and isolate it according to
the application's threat model. Relative imports inside the custom repository
are supported as of 4.57.0.
