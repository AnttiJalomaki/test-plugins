---
name: transformers-knowledge-patch
description: Transformers
version: 5.9.0
license: MIT
metadata:
  author: Nevaberry
---


# Transformers Knowledge Patch

Use this skill when writing, reviewing, upgrading, or debugging Python code built on Transformers. Start with the migration rules below, then open the topic reference that matches the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [Compatibility and API migration](references/compatibility-and-api-migration.md) | Runtime requirements, tokenizer and configuration changes, renamed and removed APIs, pipeline cleanup, and backend deprecations |
| [Loading, quantization, and kernels](references/loading-quantization-and-kernels.md) | Checkpoint conversion, dtype and sharding, quantizers, GGUF, attention kernels, tensor parallel loading, and serialization |
| [Generation, caches, and serving](references/generation-caches-and-serving.md) | Generation contracts, assisted and custom generation, cache behavior, continuous batching, chat CLI, and Serve endpoints |
| [Training and distributed execution](references/training-and-distributed-execution.md) | Trainer behavior, gradient accumulation, tensor/expert/sequence parallelism, FSDP, export, weight tying, and optimizers |
| [Multimodal processing and pipelines](references/multimodal-processing-and-pipelines.md) | Image, video, audio, processor and chat-template inputs; fast processors; task pipelines; result-affecting fixes |
| [Model and task integrations](references/model-and-task-integrations.md) | Language, vision, multimodal, speech, document, time-series, scientific, robotics, and retrieval model capabilities |

## Breaking changes and defaults

### Treat v5 as an API migration

Before upgrading an integration, audit tokenizers, model loading, custom model hooks, configurations, pipelines, and direct `forward` calls together. Important changes include:

- Python 3.10+ and PyTorch 2.4+ are required.
- TensorFlow and JAX are deprecated; TorchScript and `torch.fx` integration are removed in favor of Dynamo and Export.
- `dtype="auto"` is the loading default and preserves a checkpoint's saved dtype.
- `token` replaces `use_auth_token`.
- `quantization_config` replaces top-level `load_in_4bit` and `load_in_8bit`.
- `inputs_embeds` is the standardized embedding argument.
- configuration constructors are keyword-only dataclasses.
- direct model calls should not pass `cache_position`; generation manages it.

Read the compatibility reference before adapting custom tokenizers, processors, attention functions, model subclasses, or configurations.

### Update tokenizer call sites

Call a tokenizer instead of `encode_plus`. `decode` accepts either one sequence or a batch. `apply_chat_template` returns a `BatchEncoding`, so select `input_ids` or another field explicitly:

```python
encoded = tokenizer(["hello", "world"])
texts = tokenizer.decode(encoded["input_ids"])

chat = tokenizer.apply_chat_template(messages, return_tensors="pt")
input_ids = chat["input_ids"]
```

Use `text_target` for sequence-to-sequence target encoding and `word_ids()` instead of `BatchEncoding.words()`. New tokenizer saves no longer write the legacy special-token and added-token sidecar JSON files.

### Migrate custom generation hooks

`prepare_inputs_for_generation` now receives full `input_ids`; do not use `cache_position` to slice them. Model implementations initialize caches explicitly, return `Cache` objects, and standardize the argument name as `past_key_values`.

Remove dependencies on legacy cache classes, `EncoderDecoderCache.batch_split`, and `from_legacy_cache`. Use native cache objects for Mamba and hybrid Mamba-attention architectures.

### Remove unavailable decoding and pipeline modes

DoLa and Contrastive Search moved to repository-provided custom generation implementations. Group Beam Search and Constrained Beam Search are removed. The v5 pipeline cleanup also removes or changes the old question-answering, visual-question-answering, and image-to-image task names.

Generation with a repetition penalty must receive `input_ids`, even when other model inputs are supplied.

### Opt into kernels explicitly

Installing `kernels` does not replace model forwards. Select kernels with `use_kernels=True`, `attn_implementation`, or `set_attn_implementation`. Unsupported `output_attentions=True` combinations fail early; there is no silent eager-attention fallback.

```python
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    use_kernels=True,
    dtype="auto",
)
model.set_attn_implementation("kernels-community/flash-attn3@main")
```

Custom attention implementations must use the current attention-mask interface and call the rotary function directly rather than through `self.rotary_fn`.

## Loading and quantization

### Make dtype and device placement deliberate

`from_pretrained` preserves checkpoint dtype by default. Pass an explicit dtype when the workload requires float32 or another representation. The default save shard size is 50 GB.

GGUF cannot offload to disk. Keep every GGUF device-map target on an actual compute device. Tensor-parallel quantized inference supports only explicitly documented formats; do not assume every quantizer composes with tensor parallelism.

### Configure quantization objects

Use a concrete configuration such as `BitsAndBytesConfig`, `FPQuantConfig`, or a registered custom `QuantizationConfigMixin`:

```python
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    device_map="auto",
    dtype="auto",
    quantization_config=BitsAndBytesConfig(load_in_4bit=True),
)
```

Never try to quantize an already quantized model. Check hardware, package, serialization, CPU dequantization, and distributed-execution constraints in the loading reference.

### Prefer declarative weight conversion

Use `WeightConverter` to map checkpoint keys and apply reversible concatenate, split, reshape, quantization, or parallelism operations. Conversion recurses through nested model structures, which avoids embedding checkpoint-specific transformations in `from_pretrained`.

## Generation and serving

### Choose cache behavior from model attention

Sliding-window and chunk-attention models use dynamic sliding-window cache layers that retain only required past state. Configured window limits are enforced. Flash Attention window and causality corrections can change outputs, so re-run long-context regression tests after upgrades.

Non-generative models do not allocate KV caches. Per-layer cache representation supports hybrid attention types and lets `CacheProcessor` encapsulate offload or quantization.

### Use continuous batching for request workloads

For paged batched generation, use `generate_batch` with left-padded tokenizer inputs. Request results are keyed by request ID, incoming order is preserved, and per-request sampling is supported.

```python
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    dtype=torch.bfloat16,
    _attn_implementation="sdpa_paged",
    device_map="auto",
)
outputs = model.generate_batch(inputs=input_id_lists)
```

Continuous batching supports full or sliding-window attention, CPU request offload, tensor parallelism, and Serve integration. See the generation reference for long-context fixes and observability changes.

### Treat custom generation as remote code

`custom_generate` can load an implementation from a Hub repository and can use relative imports. It requires explicit `trust_remote_code=True`; apply the same review and revision-pinning policy as for any executable dependency.

### Keep local serving scoped

`transformers serve` is intended for experimentation and private local use. It provides chat completions, responses, completions, audio transcription, and model-listing endpoints, plus audio/video input, compilation, model timeout, and tool-call support.

The server rejects requests naming a model other than its pinned model. Model-list responses expose `owned_by` as a string.

## Training and distributed execution

`Trainer` correctly scales a short final gradient-accumulation window and averages tokens across devices by default. Recheck expected loss scaling when moving an existing run.

Tensor, expert, and sequence parallel paths have model- and quantizer-specific requirements. Update decoder-only tensor-parallel mappings for corrected all-reduce semantics. Prefer releases containing the expert-parallel and FSDP correctness fixes before trusting distributed loss or weights.

Use `StableAdamW` when its stability behavior is desired. `Trainer` also exposes `ddp_static_graph`; adapters can load with tensor parallelism.

## Multimodal inputs

Fast image processors use Torch/Torchvision functional transforms on CPU or CUDA, while PIL-only processing no longer requires Torchvision. Custom image processors must use the unified `image_processing_utils` backend.

`apply_chat_template` accepts file, URL, PIL, in-memory video, and OpenAI-style `image_url` content where supported. It can prefill custom fields such as `reasoning_content` and `thinking`.

For SAM3-family models, pass full text embeddings to `text_embeds`, not pooled outputs. For affected vision-language models, use the unified three-dimensional position-ID contract rather than constructing model-specific layouts.

## Upgrade checklist

1. Confirm Python, PyTorch, Flash Attention, torchao, CUDA, and accelerator requirements.
2. Replace removed tokenizer, authentication, quantization, cache, pipeline, and configuration APIs.
3. Audit direct imports from old image-processor or agent modules.
4. Re-test attention masks, sliding-window generation, cache reuse, repetition penalties, and custom generation hooks.
5. Re-test fast/slow preprocessing parity and all image, video, audio, and chat-template input forms.
6. Verify distributed loss scaling, weight tying, tensor/expert-parallel mappings, and checkpoint serialization.
7. Pin and review any Hub-provided executable generation or kernel code.
