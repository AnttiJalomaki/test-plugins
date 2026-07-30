# Multimodal processing and pipelines

Use this reference for image, video, audio, chat-template, processor, pipeline, and embedding input contracts, plus preprocessing changes that can affect results.

## Processor architecture

### Fast image processors

Most vision and vision-language architectures can use fast processors backed by Torch and Torchvision functional transforms on CPU or CUDA (since 4.52.1). Fast implementations were later added for SuperPoint, SegFormer, Janus, DeepSeek-VL, and DeepSeek-VL Hybrid (since 4.55.0).

Fast `center_crop` now matches the slow implementation (since 4.57.0). Keep parity tests when switching processor backend.

### Unified image-processing backend

The separate `BaseImageProcessor` and `BaseImageProcessorFast` architecture was replaced by a unified backend. `image_processing_utils_fast` is removed, so direct imports and custom processors must migrate to `image_processing_utils` (since 5.4.0).

PIL-backed processors do not require Torchvision when they only use PIL operations (since 5.5.0).

Video processors are separate classes rather than image-processor variants (since 4.52.1). `make_batched_video` handles five-dimensional arrays (since 5.1.0).

### Hub and configuration handling

- `AutoProcessor.from_pretrained` forwards Hub keyword arguments rather than discarding them (since 5.4.0).
- Timm backbones preserve `out_features` across save/load (since 5.2.0).
- A registered configuration is preferred to remote code, and `AutoConfig.from_pretrained` can take an explicit `model_type` (since 5.5.0).

## Chat-template inputs and outputs

### Return shape

`apply_chat_template` returns a `BatchEncoding`, not bare token IDs or a tensor. Select the desired field (since 5.0.0):

```python
chat_inputs = processor.apply_chat_template(
    messages,
    return_tensors="pt",
)
input_ids = chat_inputs["input_ids"]
```

Multiple raw chat-template files can be saved and loaded (since 4.52.1).

### Media inputs

Depending on processor support, chat templates can:

- load audio embedded in video inputs (since 4.51.0);
- accept in-memory video in addition to paths and URLs (since 4.55.0);
- correctly load PIL image inputs (since 4.57.0);
- accept OpenAI-style `image_url` content entries (since 5.2.0);
- prefill custom fields such as `reasoning_content` and `thinking` (since 5.9.0).

Serve forwards `tool_calls` and `tool_call_id` into processor inputs and uses `parse_response` for tool output parsing (since 5.6.0).

## Image and video input contracts

### Unified position and embedding inputs

Vision-language models use a shared Qwen2-VL-derived three-dimensional position-ID interface. Migrate custom processors and manual position-ID construction for affected Ernie and GLM4V-style integrations (since 5.3.0).

For SAM3, EdgeTAM, and SAM3-Lite-Text, `text_embeds` means full text embeddings, not pooler outputs (since 5.9.0).

Generic `get_input_embeddings` and `set_input_embeddings` discover a multimodal model's nested `language_model` component (since 5.9.0).

Multimodal callers can pass `mm_token_type` as non-padded lists (since 5.4.0). LLaVA-OneVision accepts `image_sizes` (since 5.1.0).

### Gemma 4 preprocessing

Gemma 4 preserves aspect ratio while selecting a fixed budget of 70, 140, 280, 560, or 1,120 soft tokens per image; 280 is the default. Total pixels must fit the patch budget, and processed height and width must both be divisible by 48 (since 5.5.0).

Do not apply ImageNet mean/std normalization. Gemma 4 patch embedding performs final `[-1, 1]` scaling internally.

### Resolution and resize semantics

- LFM2-VL preserves native resolution through 512×512 without forced upscaling or aspect-ratio distortion. Larger images are divided into 512×512 patches, and the 1.6B variant also receives a thumbnail for global context (since 4.57.0).
- Janus resizing rounds dimensions rather than truncating them, which can cause small numerical differences (since 5.1.0).
- SigLIP2 NaFlex supports variable aspect ratios and resolutions (since 4.50.0).

## Pipeline behavior

### Image-text and batch inference

- `image-text-to-text` accepts post-processing keyword arguments (since 4.50.0).
- The Gemma 3n `image-text-to-text` pipeline accepts an image URL and a prompt containing `<image_soft_token>` (since 4.53.0):

```python
pipe = pipeline("image-text-to-text", model="google/gemma-3n-e4b")
output = pipe(
    image_url,
    text="<image_soft_token> in this image, there is",
)
```

- Image-text inference supports batch sizes greater than one (since 4.57.0).
- GLM-Image supports batch sizes greater than one (since 5.1.0).

### Pipeline task cleanup

The v5 cleanup removes or changes the `question-answering`, `visual-question-answering`, and `image-to-image` task names. Update pipeline selection and tests rather than retaining old strings (since 5.3.0).

EoMT is compatible with the image-segmentation pipeline (since 4.54.0).

### Visualization helpers

Replace deprecated `plot_keypoint_matching` with `visualize_keypoint_matching` (since 4.55.0). LightGlue can be combined with SuperPoint for pose or homography estimation (since 4.53.0).

## Audio and speech processing

- `Speech2TextFeatureExtractor` exposes dithering (since 4.50.0).
- Whisper word timestamps interpret a timestamp token as the end of that token's time span (since 4.54.0).
- Whisper transcription accepts a progress callback (since 4.55.0).
- `WhisperFeatureExtractor` keeps `input_features` and `attention_mask` lengths consistent (since 4.57.0).
- The ASR pipeline no longer emits `num_frames` (since 5.1.0).
- VITS `forward` accepts optional speaking-rate control (since 5.3.0):

```python
outputs = model(**inputs, speaking_rate=1.2)
```

- Audio models have vLLM compatibility (since 5.6.0).

## Model-specific processing and output fixes

### Vision and multimodal correctness

- `Dinov2ForImageClassification` handles models containing register tokens correctly (since 4.52.1).
- PerceptionLM receives video and correctly preprocesses non-tiled images (since 4.56.0).
- Fuyu image inference is repaired (since 4.56.0).
- Qwen-VL video beam search is repaired (since 4.56.0).
- LLaVA-OneVision batch inference is corrected (since 4.56.0).
- Tensor/device handling is corrected for Idefics2, Idefics3, and SmolVLM (since 4.56.0).
- Qwen2.5-VL no longer applies temporal RoPE scaling incorrectly to still images, so still-image output can change after upgrade (since 5.6.0).
- `Zamba2MambaMixer` honors `use_mamba_kernels=False` rather than continuing to use the kernels (since 5.6.0).

### Output shape and metadata

- `CLIPOutput` includes attentions (since 5.1.0).
- `BeitImageProcessorFast.reduce_label` returns `labels` instead of `label` (since 5.1.0).
- Flash Attention utilities accept one-dimensional `position_ids` (since 5.1.0).
- Non-generative models do not create KV caches (since 4.54.0).

## Input regression checklist

1. Compare fast and slow image preprocessing and PIL-only dependency sets.
2. Check chat-template output fields and every supported media carrier: path, URL, PIL object, and in-memory video.
3. Rebuild custom 3D position IDs with the unified contract.
4. Pass full text embeddings to SAM3-family models.
5. Validate model-specific resize, normalization, register-token, and temporal-RoPE behavior.
6. Recheck audio feature/mask lengths, timestamp interpretation, speaking rate, and removed ASR output fields.
