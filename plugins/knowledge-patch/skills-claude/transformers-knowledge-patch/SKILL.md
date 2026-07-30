---
name: transformers-knowledge-patch
description: Transformers
version: 5.9.0
license: MIT
metadata:
  author: Nevaberry
---


# Transformers Knowledge Patch

Use this skill when upgrading or writing Python code against recent Hugging Face
Transformers APIs, especially for tokenizer migrations, checkpoint loading,
generation caches, attention kernels, quantization, distributed execution,
multimodal processors, pipelines, or local serving.

## How to use this patch

1. Read the project's declared `transformers`, Python, PyTorch, quantization,
   and accelerator versions before proposing code.
2. Apply only guidance introduced at or below the installed Transformers
   version. If the project is newer than this skill's frontmatter version,
   treat the patch as potentially stale and verify against project code/tests.
3. Search the reference index for the developer task, then keep model-specific
   constraints alongside generic API advice.
4. Prefer the project's manifests, code, tests, checkpoint metadata, and
   observed behavior whenever they disagree with this guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration and runtime](references/migration-and-runtime.md) | v5 removals, configuration changes, runtime floors, renamed APIs, output behavior |
| [Loading, quantization, and kernels](references/loading-quantization-and-kernels.md) | checkpoint conversion, dtypes, quantizers, GGUF/FP8/MXFP4, Hub kernels |
| [Generation, attention, and caches](references/generation-attention-and-caches.md) | decoding removals, continuous batching, assisted/custom generation, cache contracts |
| [Training and distributed execution](references/training-and-distributed.md) | Trainer behavior, tensor/expert/sequence parallelism, optimizers, export and compilation |
| [Tokenizers, processors, and multimodal inputs](references/tokenizers-processors-and-multimodal.md) | tokenizer backends, chat templates, image/video/audio preprocessing |
| [Serving, pipelines, and tools](references/serving-pipelines-and-tools.md) | `transformers serve`, chat CLI, task cleanup, visualization and callbacks |
| [Model and task integrations](references/model-and-task-integrations.md) | model-family capabilities, task coverage, model-specific caveats and corrections |

## Breaking-change quick reference

### Runtime floors

- Current v5-era code requires Python 3.10 or newer.
- The PyTorch floor moved to 2.2 and then 2.4. Pin a compatible pair instead of
  assuming an older environment will import successfully.
- TensorFlow and JAX backends were deprecated. Plan new integrations around the
  PyTorch path.

### Loading and dtype

- Use `dtype=...`; `torch_dtype` is transitional compatibility syntax.
- `from_pretrained()` defaults `dtype` to `auto`, preserving the checkpoint's
  saved dtype. Pass an explicit dtype when float32 or another dtype is required.
- Replace `use_auth_token=` with `token=`.
- Replace top-level `load_in_4bit=` and `load_in_8bit=` with a
  `quantization_config`, such as `BitsAndBytesConfig`.
- Saved model shards default to a 50 GB maximum rather than 5 GB.

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-3B",
    token=token,
    dtype="auto",
    device_map="auto",
    quantization_config=BitsAndBytesConfig(load_in_4bit=True),
)
```

### Tokenizers

- Call the tokenizer instead of `encode_plus()`.
- `decode()` accepts single and batched inputs; `batch_decode()` is no longer
  required just because the input is batched.
- `apply_chat_template()` returns `BatchEncoding`; select `input_ids` or the
  required field rather than treating the return value as a tensor.
- Use `text_target=` instead of `as_target_tokenizer()` and related target-mode
  helpers. Use `word_ids()` instead of `BatchEncoding.words()`.
- Tokenizer saves no longer write `special_tokens_map.json` or
  `added_tokens.json`; named special tokens live in `tokenizer_config.json`,
  while added tokens live in `tokenizer.json`.

```python
encoded = tokenizer(["hello", "world"])
texts = tokenizer.decode(encoded["input_ids"])
chat = tokenizer.apply_chat_template(messages, return_tensors="pt")
input_ids = chat["input_ids"]
```

### Configurations

- Construct `PreTrainedConfig` and model configs with keyword arguments only;
  they are dataclasses and reject positional arguments.
- Replace removed `from_xxx_config` helpers with ordinary constructors.
- Load configs from a local path or Hub repository, not an arbitrary URL.
- Read rotary settings from `config.rope_parameters`, not direct attributes
  such as `config.rope_theta`.
- Read Qwen-VL values from subconfigs such as `config.text_config.vocab_size`.
- Do not read `model.config.generation_config` on non-generative models.
- Use `config.backbone_config` as the source of truth for backbone models.

### Caches and generation hooks

- Model code should initialize and return `Cache` objects and use the plural
  argument name `past_key_values`.
- Native Mamba and hybrid cache classes replace custom cache workarounds.
- Remove `cache_position` from direct `forward()` calls. `generate()` manages
  cache positions and passes full `input_ids` to `prepare_inputs_for_generation`.
- Custom generation overrides must no longer slice inputs based on
  `cache_position`.
- Configured sliding-window limits are enforced and can change outputs that
  previously relied on an effectively unbounded cache.
- `EncoderDecoderCache.batch_split` has been removed.

### Removed features and names

- `transformers.agents` is gone; use the separate `smolagents` library.
- `pad_to_max_length`, DoLa, Contrastive Search, Group Beam Search, and
  Constrained Beam Search are no longer built in. The first two decoding
  strategies are available as remote custom-generation repositories.
- Head masking, head pruning, and BERT-style relative positional biases require
  staying on Transformers 4.x.
- The `torchscript` and `torch.fx` integrations were replaced by PyTorch
  `dynamo` and `export` workflows.
- Use `AnnotationFormat`, not the removed `AnnotionFormat`; the ASR pipeline no
  longer returns `num_frames`.
- Use `visualize_keypoint_matching`, not deprecated `plot_keypoint_matching`.
- The Apex integration was removed; use native PyTorch mixed precision and
  fused operations.

## High-value current APIs

### Continuous batching

Use `generate_batch()` with paged attention for stable continuous batching.
Left-pad tokenizer inputs, configure generation on the model, and consume the
result mapping by request ID. The implementation supports full and
sliding-window attention, CPU offload, tensor parallelism, and per-request
sampling. Recent fixes preserve arrival order, long-generation memory
estimates, request offsets, and the caller's attention implementation.

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen3-4B-Instruct-2507",
    dtype=torch.bfloat16,
    _attn_implementation="sdpa_paged",
    device_map="auto",
)
model.generation_config.max_new_tokens = 32
tokenizer = AutoTokenizer.from_pretrained(
    "Qwen/Qwen3-4B-Instruct-2507", padding_side="left"
)
requests = [tokenizer(text)["input_ids"] for text in ["Explain KV caches.", "Write a haiku."]]
outputs = model.generate_batch(inputs=requests)
```

### Explicit kernels and attention

- Installing `kernels` does not alter a model automatically. Opt in with
  `use_kernels=True`, call `kernelize`, or select an implementation explicitly.
- Select Hub attention implementations at runtime with
  `set_attn_implementation()`. References may include an `@revision` suffix.
- Unsupported `output_attentions=True` combinations fail early; do not expect a
  silent eager-attention fallback.
- Custom attention integrations must follow the current attention-mask
  interface and call the rotary function directly rather than
  `self.rotary_fn(...)`.

```python
model.set_attn_implementation("kernels-community/flash-attn3@main")
```

### Custom quantization

Register a quantization config and quantizer under the same method name. Once
registered, `from_pretrained()` accepts the custom configuration.

```python
@register_quantization_config("custom")
class CustomConfig(QuantizationConfigMixin):
    pass

@register_quantizer("custom")
class CustomQuantizer(HfQuantizer):
    pass
```

### Custom generation from the Hub

`generate(custom_generate=...)` can execute a generation implementation stored
in a Hub repository. This runs repository code, so review and pin that code and
opt in explicitly with `trust_remote_code=True`.

```python
output = model.generate(
    **inputs,
    custom_generate="transformers-community/custom_generate_example",
    trust_remote_code=True,
)
```

### Multimodal chat inputs

`apply_chat_template()` accepts in-memory video and OpenAI-style `image_url`
content. Modern integrations standardize embedding inputs on `inputs_embeds`.
Model-specific processors still impose shape, token-budget, and embedding-form
requirements; consult the multimodal and model references before normalizing or
pooling inputs yourself.

## Upgrade verification checklist

- Exercise one real checkpoint load and save, including device mapping,
  quantization, tied weights, and dtype assertions.
- Compare generation beyond any sliding window and continue once from a
  populated cache.
- Test custom attention/model overrides against mask and input-preparation
  contracts.
- Snapshot tokenizer special tokens, chat-template return fields, decode output,
  and image/video preprocessing shapes.
- Run a partial final gradient-accumulation window and the project's distributed
  strategy; several releases changed loss scaling and collective behavior.
- Validate any renamed or cleaned-up pipeline task against the concrete model
  and expected output schema.
- Treat model-integration fixes as result-affecting when the references say so;
  re-baseline numerical tests after an upgrade.
