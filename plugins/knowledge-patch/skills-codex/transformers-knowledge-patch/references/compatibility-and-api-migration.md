# Compatibility and API migration

Use this reference for upgrades, dependency floors, changed defaults, tokenizer/configuration migrations, and removed or renamed APIs.

## Release and runtime policy

- Model integrations may ship between monthly PyPI releases as mutable GitHub-only tags such as `v4.49.0-Gemma-3`. These tags start from `main` and can be updated with model fixes or other integrations already on `main`; pin a commit when reproducibility matters (since 4.50.0).
- `transformers.agents` was deprecated in favor of the separate `smolagents` library in 4.50.0 and removed in 4.52.1. Migrate agent imports and implementations to `smolagents`.
- PyTorch 2.0 support began phasing out in 4.52.1. The minimum became PyTorch 2.2 in 4.56.0 and PyTorch 2.4 in 5.1.0.
- TensorFlow and JAX backends are deprecated (since 4.53.0).
- Python 3.10 or newer is required (since 5.2.0).
- Initial `torch_tpu` backend support is available (since 5.9.0). MUSA supports TF32 flags (since 4.57.0), and Neuron devices participate in automatic compilation (since 5.6.0).
- The unmaintained `jieba` dependency was replaced by `rjieba` (since 4.57.0).

## Loading and call defaults

- Pipeline `dtype` defaults to `auto` (since 4.53.0).
- Prefer `dtype` over the transitional `torch_dtype` argument throughout the APIs (since 4.56.0).
- Compilation defaults to `fullgraph=False`, which is less restrictive for dynamic and mixture-of-experts models (since 4.56.0).
- `from_pretrained` defaults to `dtype="auto"` and preserves the checkpoint's saved dtype rather than forcing float32 (since 5.0.0).
- Replace `use_auth_token` with the equivalent `token` argument (deprecated since 5.0.0).
- Replace top-level `load_in_4bit` and `load_in_8bit` with a `quantization_config` object (since 5.0.0).
- Use the plural `inputs_embeds`; the singular `input_embeds` integration convention is obsolete (since 5.2.0).

## Tokenizer architecture

### Unified backends and construction

Tokenizers use a single implementation selected from `TokenizersBackend`, `SentencePieceBackend`, `PythonBackend`, or `MistralCommonBackend`; `AutoTokenizer.from_pretrained` selects the backend automatically. `PythonBackend` replaces the old `PreTrainedTokenizer` role for custom Python tokenizers, and `PreTrainedTokenizerBase` is the minimal backend-independent interface (since 5.0.0).

A tokenizers-backed class can be constructed empty for training or directly with `vocab` and `merges`. Constructors do not accept a `vocab_file` path; use `from_pretrained` for file-backed loading:

```python
from transformers import LlamaTokenizer

blank = LlamaTokenizer()
tokenizer = LlamaTokenizer(vocab=vocab, merges=merges)
```

### Calls, results, and target text

- Call `tokenizer(...)` instead of `encode_plus`, which is deprecated (since 5.0.0).
- `decode` handles one sequence or a batch, so `batch_decode` is no longer required merely because input is batched (since 5.0.0).
- `apply_chat_template` returns `BatchEncoding`, not a raw tensor or list of IDs; select `input_ids` or another result field (since 5.0.0).
- `sanitize_special_tokens` and target-mode helpers such as `as_target_tokenizer` are removed. Pass target text with `text_target`; `prepare_seq2seq_batch` is deprecated (since 5.0.0).
- Replace `BatchEncoding.words()` with `word_ids()` (since 5.0.0).

```python
model_inputs = tokenizer(
    source_texts,
    text_target=target_texts,
    max_length=128,
    return_tensors="pt",
)
model_inputs["labels"] = model_inputs.pop("input_ids_target")
```

Custom subclasses must implement `create_token_type_ids_from_sequences`, `prepare_for_model`, `build_inputs_with_special_tokens`, and `truncate_sequences`, or inherit their behavior from `PythonBackend`; the base class no longer supplies these operations (since 5.0.0).

### Serialization and special tokens

New saves put named special tokens in `tokenizer_config.json` and added tokens in `tokenizer.json`. The older `special_tokens_map.json` and `added_tokens.json` remain readable but are not written (since 5.0.0).

`special_tokens_map` now contains only named attributes. Put extra tokens in `extra_special_tokens`; `additional_special_tokens` is converted for compatibility. Extended special-token accessors are removed (since 5.0.0).

### Selection and text-processing corrections

- `AutoTokenizer` no longer guesses a tokenizer type from substrings such as `bert` in a directory name when configuration lacks `model_type`; fix the repository configuration (since 5.2.0).
- The later class-selection change that chose the wrong tokenizer for models such as DeepSeek R1 was reverted (since 5.7.0).
- `PreTrainedTokenizerFast` skips `clean_up_tokenization` for BPE tokenizers (since 5.7.0).
- Automatic tokenizer mapping for DeepSeek OCR selects the intended tokenizer class (since 5.8.0).
- Llama 3 conversion sets `clean_up_tokenization_spaces=False` (since 5.4.0).
- `Siglip2Tokenizer` enforces its training-time text-preprocessing defaults (since 5.1.0).

## Configuration migration

### Nested configurations

- Replace generated helpers such as `from_xxx_config` with normal constructors (since 5.0.0).
- Configurations cannot load from arbitrary URL files; use a local path or Hub repository (since 5.0.0).
- Rotary settings such as `rope_theta` and `rope_type` live in `config.rope_parameters`; some architectures store a nested mapping per layer type. Direct `config.rope_theta` access fails (since 5.0.0).
- Read Qwen-VL fields from subconfigs, for example `config.text_config.vocab_size`, rather than the top-level object (since 5.0.0).
- Non-generative configurations no longer contain `generation_config`; do not read `model.config.generation_config` (since 5.0.0).
- Backbone models use `config.backbone_config` as the single source of truth. Redundant backbone-selection arguments are removed; construction comes from config and pretrained weights load only when the checkpoint contains them (since 5.1.0).
- T5Gemma2 propagates the attention implementation to every subconfiguration, including `config.encoder.text_config` (since 5.1.0).
- `AutoModel` can load `T5Gemma2Encoder` directly (since 5.1.0).

### Dataclasses and configurable fields

`PreTrainedConfig` and model configuration classes are keyword-only dataclasses; positional configuration arguments fail (since 5.4.0):

```python
config = SomeModelConfig(hidden_size=1024, num_hidden_layers=24)
```

- Bamba, FalconH1, and GraniteMoeHybrid expose formerly hardcoded `time_step` values in configuration (since 5.1.0).
- `BeitConfig.segmentation_indices` migrates to `out_indices` (since 5.1.0).
- `AutoConfig.from_pretrained` accepts an explicit `model_type`; a registered configuration is preferred over remote code (since 5.5.0).

## Removed and renamed APIs

### Model and framework facilities

- Head masking, head pruning, and relative positional biases in BERT-like models are removed; workloads requiring them must remain on 4.x (since 5.0.0).
- TorchScript and `torch.fx` integrations are removed in favor of PyTorch Dynamo and Export (since 5.0.0).
- The `torchao.autoquant` integration is removed (since 5.1.0).
- `EncoderDecoderCache.batch_split` is removed (since 5.2.0).
- The Apex integration, including Apex RMSNorm in T5-related models, is removed; use native PyTorch mixed precision and fused operations (since 5.8.0).

### Names and output fields

- `pad_to_max_length` is removed (since 4.52.1).
- Replace deprecated `plot_keypoint_matching` with `visualize_keypoint_matching` (since 4.55.0).
- Replace the removed misspelling `AnnotionFormat` with `AnnotationFormat` (since 5.1.0).
- The ASR pipeline no longer emits the `num_frames` entry (since 5.1.0).
- `BeitImageProcessorFast.reduce_label` returns `labels`, not `label` (since 5.1.0).
- Ernie 4.5 VL MoE model and config names now match vLLM and SGLang conventions; update old class names (since 5.3.0).

## Pipeline and task migration

- The v5 pipeline cleanup removes or changes the `question-answering`, `visual-question-answering`, and `image-to-image` task names. Migrate callers to their current replacement pipelines or updated names (since 5.3.0).
- DoLa and Contrastive Search are no longer built in; load the respective `transformers-community/dola` or `transformers-community/contrastive-search` custom generation implementation when needed (since 4.56.0).
- Group Beam Search and Constrained Beam Search are removed and cannot appear in generation configurations (since 4.57.0).
- The old `triton_kernels` dependency was replaced by `kernels` (since 4.56.0). The GPT-OSS-specific renamed package is `gpt-oss-triton-kernels` (since 5.1.0).

## Migration audit

When upgrading a custom integration:

1. Validate the Python and PyTorch floors and accelerator-specific dependencies.
2. Convert tokenizer construction, target encoding, return-value handling, and serialization.
3. Move configuration fields into their correct subconfigs and use keyword arguments.
4. Replace authentication, quantization, embedding, cache, and renamed helper arguments.
5. Remove obsolete backends, pipeline tasks, decoding modes, and framework integrations.
6. Re-run tokenizer-class, preprocessing, configuration-round-trip, and generation regression tests.
