# Migration and runtime

Use this reference to audit compatibility boundaries and cross-cutting API
changes. The sections are organized by migration task, not release order.

## Choose a distribution and runtime

- Model integrations may ship as mutable GitHub-only tags in addition to
  monthly PyPI releases. A tag such as `v4.49.0-Gemma-3` starts from `main` at
  model release time, can be updated with model-specific fixes, and can contain
  unrelated integrations already present on `main`; pin a commit when exact
  reproducibility matters (4.50.0).
- PyTorch 2.0 was already being phased out in 4.52.1. The minimum became 2.2 in
  4.56.0 and 2.4 in 5.1.0. Resolve the installed Transformers/PyTorch pair
  before diagnosing an import or kernel failure.
- TensorFlow and JAX backends are deprecated as of 4.53.0.
- Python 3.10 or newer is required as of 5.2.0.
- The old `triton_kernels` dependency was replaced by `kernels` in 4.56.0. For
  GPT-OSS specifically, the renamed package is `gpt-oss-triton-kernels` as of
  5.1.0.
- Initial `torch_tpu` backend support appears in 5.9.0; treat it as initial
  support rather than parity with mature backends.
- The MUSA backend gains TF32-flag support in 4.57.0. The unmaintained `jieba`
  dependency is replaced by `rjieba` in the same release.

## Remove deprecated packages and arguments

- `transformers.agents` was deprecated for `smolagents` in 4.50.0 and removed
  in 4.52.1. Migrate imports and agent implementations to the separate library.
- The deprecated `pad_to_max_length` argument was removed in 4.52.1.
- `dtype` became the preferred public argument in 4.56.0; `torch_dtype` remained
  accepted during the transition. In 5.0.0, `from_pretrained()` began defaulting
  `dtype` to `auto`, preserving a checkpoint's stored dtype rather than forcing
  float32. Specify a dtype when the application requires one.
- Replace `use_auth_token` with `token`; the old name is deprecated everywhere
  as of 5.0.0.
- Top-level `load_in_4bit` and `load_in_8bit` loading arguments were removed in
  5.0.0. Express them through a quantization configuration.
- `AnnotionFormat`, a misspelling, was removed in 5.1.0; use
  `AnnotationFormat`. The ASR pipeline's `num_frames` result entry was also
  removed.
- `plot_keypoint_matching` was deprecated in 4.55.0; use
  `visualize_keypoint_matching`.
- The Apex integration, including Apex RMSNorm use in T5 and related models,
  was removed in 5.8.0. Replace Apex mixed precision and fused operations with
  native PyTorch equivalents.

## Migrate configuration code

- In 5.0.0, nested `from_xxx_config` helpers were removed in favor of ordinary
  constructors. Configs can no longer load from arbitrary URL files; use a
  local path or a Hub repository.
- Rotary settings such as `rope_theta` and `rope_type` moved under
  `config.rope_parameters` in 5.0.0. Some models use a nested mapping by layer
  type, and direct `config.rope_theta` access fails.
- Read Qwen-VL properties from subconfigs in 5.0.0, for example
  `config.text_config.vocab_size`, rather than expecting top-level mirrors.
- Non-generative configs no longer expose `generation_config` in 5.0.0;
  `model.config.generation_config` raises `AttributeError` for those models.
- Backbone models use `config.backbone_config` as their single source of truth
  in 5.1.0. Redundant backbone-selection arguments were removed, construction
  is configuration-driven, and pretrained weights load only when the
  checkpoint actually contains backbone weights.
- `BeitConfig` migrates legacy `segmentation_indices` to `out_indices` in 5.1.0.
- Bamba, FalconH1, and GraniteMoeHybrid expose previously hardcoded `time_step`
  parameters through their configs in 5.1.0.
- In 5.4.0, `PreTrainedConfig` and model config classes became dataclasses and
  reject positional values. Pass every setting by keyword.
- `AutoConfig.from_pretrained(..., model_type=...)` can explicitly override
  model selection in 5.5.0. A registered config takes precedence over remote
  code.

## Replace removed modeling and graph features

- Head masking, head pruning, and relative positional biases for BERT-like
  models were removed in 5.0.0. Keep those workloads on 4.x or redesign them.
- Transformers' `torchscript` and `torch.fx` integrations were removed in
  5.0.0 in favor of PyTorch `dynamo` and `export`.
- The `torchao.autoquant` integration was removed in 5.1.0. This is separate
  from the remaining torchao quantization integrations.
- `EncoderDecoderCache.batch_split` was removed in 5.2.0.
- Direct imports from `image_processing_utils_fast` break in 5.4.0 because the
  fast/slow base split was replaced by a unified image-processor backend. Custom
  processors should import and extend `image_processing_utils`.

## Account for changed defaults and outputs

- Pipeline `dtype` defaults to `auto` as of 4.53.0.
- `TrainingArguments.average_tokens_across_devices` defaults to enabled in
  4.54.0. Non-generative models also stopped using KV caches, and Whisper word
  timestamps now interpret a timestamp token as the end of the token's span.
- The default model-save shard size grew from 5 GB to 50 GB in 5.0.0.
- `CLIPOutput` includes attentions in 5.1.0.
- `BeitImageProcessorFast.reduce_label` returns `labels`, not `label`, in 5.1.0.
- Janus image resizing rounds rather than truncates dimensions in 5.1.0, which
  can produce small numerical differences.
- Llama 3 tokenizer conversion sets `clean_up_tokenization_spaces=False` in
  5.4.0.
- Checkpoint loading ties weights even when both tied keys with equal values are
  present as of 5.4.0. Verify `.bin` checkpoints containing duplicate tied keys.
- Qwen2.5-VL stopped applying temporal RoPE scaling incorrectly to still images
  in 5.6.0; re-baseline outputs after upgrading.
- `Zamba2MambaMixer` honors `use_mamba_kernels=False` in 5.6.0 instead of
  continuing to invoke Mamba kernels.
- The automatic tokenizer mapping for DeepSeek OCR was corrected in 5.8.0 so
  it selects the intended class.

## Validate renamed or relocated model APIs

- The Ernie 4.5 VL MoE model and config class names changed in 5.3.0 to align
  with vLLM and SGLang conventions; replace old imports and serialized
  architecture names.
- `AutoModelForTimeSeriesPrediction` is available as a direct import beginning
  in 4.53.0.
- `DeepseekV3ForTokenClassification` is available in 4.57.0.
- `ParakeetForCTC` is the CTC speech-recognition class introduced in 4.57.0;
  Parakeet TDT, added in 5.9.0, is a distinct integration.
- `AutoModel` can load `T5Gemma2Encoder` as of 5.1.0.
- The generic `get_input_embeddings()` and `set_input_embeddings()` helpers
  recognize a multimodal model's nested `language_model` in 5.9.0.
