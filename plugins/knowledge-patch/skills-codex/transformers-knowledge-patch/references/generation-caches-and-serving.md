# Generation, caches, and serving

Use this reference for assisted or custom generation, cache contracts, continuous batching, the chat CLI, and local Serve endpoints.

## Generation tools and extension points

### Inspect attention layouts

`AttentionMaskVisualizer` loads a tokenizer and model by ID and renders the resulting attention layout, including sliding-window and multimodal masks (since 4.50.0):

```python
from transformers.utils.attention_visualizer import AttentionMaskVisualizer

visualizer = AttentionMaskVisualizer("meta-llama/Llama-3.2-3B-Instruct")
visualizer("A normal attention mask")
```

### Universal assisted generation

An assistant can belong to a different model family from the target, and assisted generation works while sampling with `do_sample=True` (since 4.50.0):

```python
pipe = pipeline(
    "text-generation",
    model=target_id,
    assistant_model=assistant_id,
    do_sample=True,
)
result = pipe("Alice and Bob", max_new_tokens=50, do_sample=True)
```

Gemma 4 Assistant is a text-only Multi-Token Prediction draft model specifically for Gemma 4. It reuses the target KV cache to skip its own prefill and cross-attends to the target context while drafting (since 5.8.0).

### Hub-hosted custom generation

`generate(custom_generate=...)` loads an implementation from a Hub repository. This executes repository code and therefore requires `trust_remote_code=True` (since 4.52.1):

```python
output = model.generate(
    **inputs,
    custom_generate="transformers-community/custom_generate_example",
    trust_remote_code=True,
)
```

Custom generation repositories can use relative imports (since 4.57.0). Pin and review the revision as an executable dependency.

DoLa and Contrastive Search no longer ship in the package. Their implementations live in `transformers-community/dola` and `transformers-community/contrastive-search` (since 4.56.0). Group Beam Search and Constrained Beam Search are removed entirely (since 4.57.0).

## Generation input contract

- Passing a tensor `cache_position` into `generate` no longer fails argument handling (since 4.55.0).
- Model code initializes caches explicitly and returns `Cache` objects. Cache arguments are standardized on `past_key_values`, not `past_key_value`; deprecated cache objects are removed and `from_legacy_cache` is heading toward removal (since 4.56.0).
- Generation no longer uses `cache_position` to prepare inputs and always sends full `input_ids` to `prepare_inputs_for_generation`. Custom overrides must stop slicing with `cache_position` (since 5.3.0).
- Most major direct model `forward` methods no longer accept `cache_position`; only `generate` manages cache positions (since 5.4.0).
- Repetition-penalty generation requires `input_ids` (since 5.9.0).
- Gemma 4 generation accepts `inputs_embeds` and `per_layer_inputs`; `per_layer_inputs` is available on every Gemma 4 variant (since 5.9.0).

## Cache architecture

### Per-layer and sliding-window caches

KV caches are represented per layer, enabling hybrid caches that mix attention types. `CacheProcessor` encapsulates cache quantization and offload as independently customizable behavior (since 4.54.0).

`DynamicSlidingWindowLayer` and its cache retain and pass only the state required by sliding-window and chunk attention. A checkpoint default of `cache_implementation="hybrid"` is ignored in favor of dynamic sliding-window caching, avoiding unusually slow static-hybrid first generation (since 4.56.0).

Generation cache preparation receives model configuration and enforces configured sliding-window limits. Code that depended on an effectively unbounded cache can produce different output or require shorter input (since 5.1.0).

### Model cache behavior

- Non-generative models no longer allocate a KV cache (since 4.54.0).
- Vision encoder-decoder models support static caches (since 4.55.0).
- Flash Attention 2 can continue generation from an existing cache (since 4.56.0).
- Paged caches can reside on CPU (since 5.1.0).
- Mamba-only and mixed Mamba-attention architectures use first-class cache classes; replace custom caches and workarounds (since 5.5.0).
- Gemma 4 and Gemma 3n share KV states regardless of whether the caller supplies a `Cache` object (since 5.6.0).
- T5Gemma2 long-input cross-attention selects the correct cache-layer type, and Qwen3.5 Gated DeltaNet linear attention handles multi-token cached forwards (since 5.7.0).

### Result-affecting attention corrections

Flash Attention's sliding-window size is corrected by one position. Long-context output can change when initial context exceeds the configured window. Causality handling also supports bidirectional attention (since 4.56.0). Re-run generation tests instead of treating old and new caches as numerically equivalent.

## Continuous batching

### Basic usage

Stable `generate_batch` supports batched generation for full- and sliding-window models. The documented configuration uses paged SDPA and left-padded inputs; results are keyed by request ID (since 4.57.0):

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained(
    model_id,
    dtype=torch.bfloat16,
    _attn_implementation="sdpa_paged",
    device_map="auto",
)
tokenizer = AutoTokenizer.from_pretrained(model_id, padding_side="left")
inputs = [
    tokenizer("Explain continuous batching.")["input_ids"],
    tokenizer("Write a haiku.")["input_ids"],
]
outputs = model.generate_batch(inputs=inputs)
for request_id in outputs:
    print(tokenizer.decode(outputs[request_id].generated_tokens))
```

The feature is suitable for GRPO training, evaluation, and Serve workloads.

### Scheduling, memory, and parallelism

- Incoming request order is preserved (since 5.1.0).
- Requests can offload to CPU. KV deduplication and memory estimation are corrected for generations of 16K tokens or more, and per-request sampling parameters are documented (since 5.7.0).
- Tensor parallelism is supported. `generate_batch` restores `_attn_implementation` and uses corrected request offsets (since 5.9.0).
- The continuous-batching OpenTelemetry integration was removed (since 5.9.0).

## Chat CLI

Use `transformers chat MODEL`. Generation settings follow the model as `GenerationConfig`-style `key=value` arguments rather than a fixed flag set (since 4.52.1):

```bash
transformers chat Qwen/Qwen2.5-0.5B-Instruct do_sample=False max_new_tokens=10
```

The CLI can use the same local Serve instance as API clients.

## Transformers Serve

### Intended scope and endpoints

`transformers serve` is a local serving utility for supported modalities, intended for experimentation and private local use. It initially exposes OpenAI-compatible:

- `/v1/chat/completions`
- `/v1/responses`
- `/v1/audio/transcriptions`
- `/v1/models`

These endpoints and cross-client use with `transformers chat` arrived in 4.54.0.

### Expanded request handling

Serve also provides legacy `/v1/completions`, accepts audio and video, and supports `--compile` and `--model-timeout` (since 5.6.0). Tool-aware handling:

- forwards `tool_calls` and `tool_call_id` to processor inputs;
- uses `parse_response` for tool calls;
- returns HTTP 400 when the request names a model other than the server's pinned model.

Continuous batching is integrated into Serve (since 4.57.0) and later supports tensor-parallel execution (since 5.9.0).

`GET /v1/models` returns `owned_by` as a string, correcting the earlier list shape (since 5.9.0).

## Generation regression checklist

1. Verify full `input_ids` handling in custom `prepare_inputs_for_generation`.
2. Remove `cache_position` from direct forwards and old cache conversions.
3. Test sliding-window limits, static vision caches, hybrid layers, and cache reuse.
4. Supply `input_ids` whenever repetition penalties are active.
5. For continuous batching, test request order, long-generation memory, sampling, offload, and tensor-parallel offsets.
6. Keep Serve private unless the surrounding deployment supplies production authentication, isolation, limits, and observability.
