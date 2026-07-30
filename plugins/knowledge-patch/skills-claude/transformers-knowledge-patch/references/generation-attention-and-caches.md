# Generation, attention, and caches

## Select a generation path

### Assisted and custom generation

- Universal assisted generation accepts any assistant model, even from a
  different family, and works with sampling (`do_sample=True`) as of 4.50.0.

```python
from transformers import pipeline

pipe = pipeline(
    "text-generation",
    model="google/gemma-2-9b",
    assistant_model="double7/vicuna-68m",
    do_sample=True,
)
pipe("Alice and Bob", max_new_tokens=50, do_sample=True)
```

- `generate()` can load a Hub-hosted implementation through `custom_generate`
  as of 4.52.1. This executes repository code and therefore also requires
  `trust_remote_code=True`; review and pin the repository.
- Custom generation repositories can use relative imports as of 4.57.0.
- DoLa and Contrastive Search were removed from the library in 4.56.0. Their
  implementations moved to `transformers-community/dola` and
  `transformers-community/contrastive-search` remote-code repositories.
- Group Beam Search and Constrained Beam Search were removed in 4.57.0;
  generation configs and callers can no longer select them.
- Generation calls using a repetition penalty must provide `input_ids` as of
  5.9.0. An `inputs_embeds`-only call is insufficient for that penalty.

### Continuous batching

Stable `generate_batch()` continuous batching arrives in 4.57.0 for full- and
sliding-window attention. The documented setup uses paged SDPA and left-padded
tokenizer input. It is intended for throughput-sensitive workloads such as GRPO
training, evaluation, and `transformers serve`.

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
inputs = [
    tokenizer("Explain continuous batching.")["input_ids"],
    tokenizer("Write a haiku.")["input_ids"],
]
outputs = model.generate_batch(inputs=inputs)
```

- Arrival order is preserved as of 5.1.0.
- CPU request offload, corrected KV deduplication and memory estimation for
  generations of 16K tokens or more, and documented per-request sampling
  parameters arrive in 5.7.0.
- Tensor parallelism arrives in 5.9.0. That release also restores the caller's
  `_attn_implementation` after `generate_batch()` and corrects request offsets.
- The continuous-batching OpenTelemetry integration was removed in 5.9.0.

## Implement attention correctly

- Custom attention functions can be registered beginning in 4.51.0.
- Flash Attention 3 is supported across popular models in 4.53.0. Unsupported
  combinations of an attention implementation with `output_attentions=True`
  now fail early, and implementations no longer silently fall back to eager
  attention.
- `set_attn_implementation()` selects an implementation at runtime from
  4.54.0, including downloadable Hub kernels.
- Flash Attention sliding-window size was corrected by one position in 4.56.0,
  which can change output when initial context exceeds the window. Flash
  Attention 2 can also continue from an existing cache, and Flash Attention
  causality handling supports bidirectional attention.
- Bidirectional attention is supported across all models as of 5.1.0. Attention
  and Experts components can also be instantiated as reusable standalone
  modules.
- Flash Attention utilities accept one-dimensional `position_ids` in 5.1.0.
- The library-wide attention-mask interface changed in 5.2.0. Custom attention
  and model integrations written for the old interface must migrate.
- ModernBERT no longer selects Flash Attention implicitly in 5.2.0; choose the
  desired implementation explicitly.
- Custom attention implementations must call the rotary function directly as
  of 5.6.0; the formerly hidden kernel function is no longer registered as
  `self.rotary_fn`.
- T5Gemma2 long-input cross-attention selects the correct cache-layer type in
  5.7.0. Qwen3.5 Gated DeltaNet handles multi-token cached forwards correctly,
  and attention-only GraniteMoeHybrid configs no longer update a nonexistent
  Mamba mask.

## Migrate cache objects and call contracts

### Per-layer and sliding-window caches

- KV caches became per-layer objects in 4.54.0, allowing hybrid caches that mix
  attention types. `CacheProcessor` encapsulates cache quantization and
  offloading independently.
- `DynamicSlidingWindowLayer` and its cache in 4.56.0 retain and pass only the
  state needed by sliding-window and chunk-attention models. A checkpoint's
  `cache_implementation="hybrid"` default is ignored in favor of dynamic
  sliding-window caching, avoiding the slow first generation of a static hybrid
  cache.
- Generation cache preparation passes the model config and enforces configured
  sliding-window limits in 5.1.0. Code that relied on an effectively unbounded
  cache can change output or require shorter inputs.
- CPU paged caches are supported as of 5.1.0.
- Native cache classes replace custom caches and workarounds for Mamba-only and
  mixed Mamba/attention models in 5.5.0.

### Arguments and returned objects

- Vision encoder-decoder models support static caches in 4.55.0. A tensor-valued
  `cache_position` no longer fails `generate()` argument handling.
- In 4.56.0, model implementations initialize caches explicitly and return
  `Cache` objects. Deprecated cache objects were removed, argument names were
  standardized on `past_key_values` rather than `past_key_value`, and
  `from_legacy_cache` was prepared for deprecation.
- Generation no longer uses `cache_position` to prepare inputs as of 5.3.0 and
  always passes full `input_ids` to `prepare_inputs_for_generation`. Custom
  overrides must not slice input based on `cache_position`.
- Most major `forward()` methods no longer accept `cache_position` in 5.4.0;
  omit it from direct calls because `generate()` manages positions.
- Gemma 4 and Gemma 3n share KV states independently of whether the caller uses
  a `Cache` object in 5.6.0; cache choice no longer controls that sharing.

## Handle model-specific generation inputs

- The standard embedding argument is plural `inputs_embeds` as of 5.2.0;
  integrations using `input_embeds` must rename it.
- Vision-language models use a shared Qwen2-VL-derived 3D position-ID interface
  as of 5.3.0. Custom processors and manual position construction for models
  including Ernie and GLM4V must migrate.
- Gemma 4 generation accepts `inputs_embeds` and `per_layer_inputs` in 5.9.0;
  `per_layer_inputs` is available across every Gemma 4 variant.
- Qwen3.5's Gated DeltaNet correction in 5.7.0 matters specifically for
  multi-token forwards with a populated cache.
