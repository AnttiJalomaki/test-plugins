# Loading, quantization, and kernels

Use this reference for checkpoint loading and conversion, quantized execution, GGUF, attention implementations, downloadable kernels, and serialization.

## Checkpoint loading and conversion

### Dtype, shards, and memory mapping

- Pipelines default to `dtype="auto"` (since 4.53.0), and `dtype` is preferred over `torch_dtype` throughout the API (since 4.56.0).
- `from_pretrained` also defaults to `dtype="auto"`, preserving the saved checkpoint dtype; request float32 or another dtype explicitly when required (since 5.0.0).
- The default maximum save shard size is 50 GB rather than 5 GB (since 5.0.0).
- `from_pretrained(disable_mmap=...)` can disable memory mapping and automatically detects hf-mount (since 5.6.0).

### Declarative weight conversion

`WeightConverter` maps source keys to model keys and applies reversible reshape, merge, split, quantization, or parallelism operations. Integrations can declare fused QKV conversion instead of embedding it inside `from_pretrained` (since 5.0.0):

```python
conversion = WeightConverter(
    ["self_attn.q_proj", "self_attn.k_proj", "self_attn.v_proj"],
    "self_attn.qkv_proj",
    operations=[Concatenate(dim=0)],
)
```

Conversion recurses through nested model structures (since 5.4.0).

### Format-specific constraints

- PyTorch loading accepts FP8 safetensors such as DeepSeek checkpoints (since 4.51.0).
- GGUF files cannot be offloaded to disk. A GGUF device map must target compute devices only (since 4.51.0).
- Gemma 3 text-backbone and Gemma 3 QAT GGUF checkpoints are supported (since 4.52.1).
- Qwen3 MoE GGUF loading arrived in 4.55.0 and uses the corrected architecture in 4.56.0.
- GPT-OSS has full GGUF loading support (since 5.6.0).
- Loading `.bin` files that contain duplicate tied keys can behave differently because equal checkpoint keys are now tied even when both are present; verify the result (since 5.4.0).

## Quantization registration and configuration

### Custom quantizers

Register a configuration and quantizer under the same method name. The resulting config is accepted by `from_pretrained` (since 4.50.0):

```python
@register_quantization_config("custom")
class CustomConfig(QuantizationConfigMixin):
    pass

@register_quantizer("custom")
class CustomQuantizer(HfQuantizer):
    pass

model = AutoModelForCausalLM.from_pretrained(
    model_id,
    quantization_config=CustomConfig(),
    dtype="auto",
)
```

Top-level `load_in_4bit` and `load_in_8bit` are removed. Put the choice in a configuration such as `BitsAndBytesConfig` (since 5.0.0).

### Supported methods and integrations

- Quark-quantized repositories load through ordinary `from_pretrained` after installing `amd-quark` (since 4.50.0).
- The torchao integration supports autoquant, CPU quantization, and advanced `AOBaseConfig` configurations (since 4.50.0); `torchao.autoquant` itself was removed in 5.1.0.
- AutoRound low-bit rounding and clipping is supported (since 4.52.1).
- SINQ is available as a v5 quantization strategy (since 5.2.0).
- MLX quantization is supported on MPS devices (since 5.3.0).
- Four Over Six (4/6) NVFP4 is supported on NVIDIA Blackwell GPUs (since 5.3.0), gained configurable dtype choices in 5.6.0, and torchao NVFP4 models serialize correctly in 5.7.0.
- Static FP8 experts work in multi-GPU configurations (since 5.4.0).
- torchao 0.15.0 or newer is required by the integration (since 5.4.0).

### Tensor parallel restrictions

Quantized tensor-parallel distributed inference supports only `compressed-tensors`, `fp8`, and `fp8-fbgemm` in 4.52.1. Do not combine other quantization methods with tensor parallelism unless a later model-specific path explicitly supports it.

## FP-Quant and MXFP4

`FPQuantConfig` performs on-the-fly FP-Quant loading. Its initial implementation supports post-training MXFP4. Accelerated execution requires a Blackwell-generation NVIDIA GPU and QuTLASS; `pseudoquant=True` emulates quantization without QuTLASS (since 4.54.0):

```python
model = AutoModelForCausalLM.from_pretrained(
    "qwen/Qwen3-8B",
    quantization_config=FPQuantConfig(),
    device_map="cuda",
    dtype=torch.bfloat16,
)
```

Later MXFP4 behavior includes:

- Native GPT-OSS MXFP4 MoE loading. The 20B checkpoint fits in 16 GB and the 120B checkpoint fits in 80 GB; the larger checkpoint supplies a default plan selectable with `tp_plan="auto"` (since 4.55.0).
- CPU dequantization is selected automatically when a device map includes CPU (since 4.56.0).
- GPT-OSS MXFP4 execution extends to NVIDIA `sm75+` GPUs (since 4.56.0).
- MXFP4 supports a quantization-aware save path, and int4 models can execute on CPU (since 4.56.0).
- Quantizing an already quantized model raises an error (since 4.56.0).

## Attention implementation selection

### Registration and explicit opt-in

Custom attention functions can be registered (since 4.51.0). Installing `kernels` does not automatically replace forward methods: `@use_kernel_forward_from_the_hub` records a kernel name, `kernelize` applies it, and loading must opt in with `use_kernels=True` (since 4.53.0):

```python
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    use_kernels=True,
)
```

Flash Attention 3 is available across widely used architectures. Unsupported combinations of an attention implementation and `output_attentions=True` fail early, and implementations do not silently fall back to eager attention (since 4.53.0).

### Hub-selected kernels

Switch at runtime with `set_attn_implementation`. A Hub reference fetches a build matching installed CUDA and PyTorch versions (since 4.54.0):

```python
model.set_attn_implementation("kernels-community/flash-attn3")
```

Hub kernel references accept `@revision` suffixes (since 4.56.0):

```python
model.set_attn_implementation("kernels-community/flash-attn3@main")
```

Treat fetched kernels as executable dependencies: review and pin their source.

### GPT-OSS kernels

On Hopper with PyTorch 2.7 or 2.8, GPT-OSS can use sink-aware Flash Attention 3 after upgrading `kernels` (since 4.55.0):

```python
model = AutoModelForCausalLM.from_pretrained(
    "openai/gpt-oss-20b",
    attn_implementation="kernels-community/vllm-flash-attn3",
    device_map="auto",
    dtype="auto",
)
```

If MXFP4 is unavailable, `use_kernels=True` opts into downloadable MegaBlocks MoE. That path requires bfloat16 and consumes more memory (since 4.55.0). The GPT-OSS Triton-kernel package is named `gpt-oss-triton-kernels` (since 5.1.0).

## Attention and kernel migration

- Flash Attention sliding-window size is corrected by one position; output can change when initial context exceeds the window (since 4.56.0).
- Flash Attention 2 can continue from an existing cache, and Flash Attention causality handling supports bidirectional attention (since 4.56.0).
- Bidirectional attention is supported across all models, and Attention and Experts components are reusable standalone modules (since 5.1.0).
- Flash Attention utilities accept one-dimensional `position_ids` (since 5.1.0).
- ModernBERT no longer selects Flash Attention implicitly; select it explicitly if required (since 5.2.0).
- Custom attention integrations must migrate to the current attention-mask interface (since 5.2.0).
- Custom attention code must call the rotary function directly; it is no longer registered for `self.rotary_fn(...)` (since 5.6.0).
- Flash Attention 2 requires version 2.3.3 or newer. Initial Flash Attention 4 support includes a `kernels` fallback (since 5.4.0).
- Kernel integrations provide `paged_attention` for continuous batching and custom kernels on Neuron (since 5.4.0).
- XPU has a MegaBlocks MoE kernel implementation (since 5.1.0).

## Kernel correctness fixes

- Hub-registered custom expert kernels load correctly (since 5.7.0).
- FP8 checkpoint kernel configuration and error handling are corrected (since 5.7.0).
- Gemma3n and Gemma4 support the rotary kernel (since 5.7.0).
- Qwen3.5 Gated DeltaNet linear attention correctly handles cached forwards containing multiple tokens (since 5.7.0).
- T5Gemma2 long-input cross-attention uses the correct cache-layer type (since 5.7.0).
- Attention-only GraniteMoeHybrid configurations no longer update a nonexistent Mamba mask (since 5.7.0).

## Loading checklist

1. Select dtype, device map, sharding, and memory-mapping policy explicitly.
2. Check the exact quantizer's hardware, dependency, tensor-parallel, save, and CPU-offload support.
3. Keep GGUF weights off disk device-map targets.
4. Pin Hub kernels by revision and test unsupported attention-output combinations.
5. Re-test long sliding-window contexts and cached multi-token forwards after attention upgrades.
6. Round-trip saved quantized and tied-weight checkpoints before relying on them.
