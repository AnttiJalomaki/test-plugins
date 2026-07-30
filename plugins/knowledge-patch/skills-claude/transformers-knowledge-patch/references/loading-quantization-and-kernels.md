# Loading, quantization, and kernels

## Register and select quantizers

Custom quantization integrations register both halves under one method name
(4.50.0). After registration, `from_pretrained()` accepts the configuration.

```python
@register_quantization_config("custom")
class CustomConfig(QuantizationConfigMixin):
    pass

@register_quantizer("custom")
class CustomQuantizer(HfQuantizer):
    pass

model = AutoModelForCausalLM.from_pretrained(
    "facebook/opt-350m",
    quantization_config=CustomConfig(),
    dtype="auto",
)
```

- Quark-quantized repositories load through normal `from_pretrained()` after
  installing `amd-quark` (4.50.0).
- The torchao integration gained autoquant, CPU quantization, and advanced
  `AOBaseConfig` configuration in 4.50.0. The specific `torchao.autoquant`
  integration was later removed in 5.1.0, while other torchao paths remain.
- AutoRound is supported from 4.52.1, including low-bit rounding and clipping
  optimization.
- SINQ is available as a v5 quantization strategy from 5.2.0.
- torchao requires version 0.15.0 or newer as of 5.4.0.
- Attempting to quantize an already quantized model raises an error as of
  4.56.0; detect checkpoint quantization metadata instead of stacking methods.

## Match formats to devices and parallelism

### FP8 and FP-Quant

- PyTorch can load FP8 safetensors, including DeepSeek checkpoints, from
  4.51.0.
- Tensor-parallel inference in 4.52.1 supports only `compressed-tensors`, `fp8`,
  and `fp8-fbgemm` quantization. Other quantizers cannot be combined with tensor
  parallelism in that release.
- `FPQuantConfig` enables on-the-fly FP-Quant loading in 4.54.0. Initially only
  post-training MXFP4 is implemented. Accelerated execution requires a
  Blackwell-generation Nvidia GPU and QuTLASS; `pseudoquant=True` emulates the
  quantization without QuTLASS.
- Static FP8 experts can run in multi-GPU configurations as of 5.4.0.
- Four Over Six (4/6) NVFP4 is supported on NVIDIA Blackwell GPUs in 5.3.0 and
  gains configurable dtype choices in 5.6.0.
- torchao NVFP4 models serialize correctly as of 5.7.0.

```python
import torch
from transformers import AutoModelForCausalLM, FPQuantConfig

model = AutoModelForCausalLM.from_pretrained(
    "qwen/Qwen3-8B",
    quantization_config=FPQuantConfig(),
    device_map="cuda",
    dtype=torch.bfloat16,
)
```

### MXFP4 and GPT-OSS

- Transformers 4.55.0 loads the native MXFP4 MoE weights for
  `openai/gpt-oss-20b` and `openai/gpt-oss-120b`. The 20B checkpoint fits in
  about 16 GB with MXFP4; the 120B checkpoint fits in about 80 GB and exposes a
  default tensor-parallel plan through `tp_plan="auto"`.
- MXFP4 can automatically dequantize on CPU when `device_map` contains CPU as
  of 4.56.0. GPT-OSS MXFP4 also runs on Nvidia `sm75+`, and MXFP4 has a
  quantization-aware save path.
- On Hopper with PyTorch 2.7 or 2.8, GPT-OSS can use the sink-aware Flash
  Attention 3 kernel after upgrading `kernels` (4.55.0).
- If MXFP4 is unavailable, `use_kernels=True` opts into a downloadable
  MegaBlocks MoE path. It requires bfloat16 and consumes more memory (4.55.0).
- GPT-OSS gains full GGUF loading in 5.6.0.

```python
model = AutoModelForCausalLM.from_pretrained(
    "openai/gpt-oss-20b",
    attn_implementation="kernels-community/vllm-flash-attn3",
    device_map="auto",
    dtype="auto",
)
```

### GGUF and CPU formats

- GGUF files cannot be offloaded to disk as of 4.51.0; construct a device map
  without disk offload.
- Gemma 3 text-backbone and Gemma 3 QAT GGUF checkpoints load in 4.52.1.
- Qwen3 MoE GGUF loading appears in 4.55.0 and uses a corrected architecture in
  4.56.0.
- `int4` quantized models can execute on CPU as of 4.56.0.
- MLX quantization on MPS devices is supported in 5.3.0.

## Use checkpoint conversion and I/O controls

The `WeightConverter` API introduced in 5.0.0 declaratively maps checkpoint
keys to model keys. Its reversible operations can reshape, merge, split,
quantize, or apply parallel transformations, so integrations can fuse QKV
weights without hard-coding conversion inside `from_pretrained()`.

```python
conversion = WeightConverter(
    ["self_attn.q_proj", "self_attn.k_proj", "self_attn.v_proj"],
    "self_attn.qkv_proj",
    operations=[Concatenate(dim=0)],
)
```

- Dynamic conversion recurses through nested model structure as of 5.4.0.
- Decoder-only dense and MoE tensor-parallel all-reduce corrections in 5.3.0
  require existing tensor-parallel configs and checkpoint conversion mappings
  to be updated.
- `from_pretrained(..., disable_mmap=...)` is available in 5.6.0 and includes
  automatic hf-mount detection.
- Loading `.bin` checkpoints with duplicate tied keys needs verification after
  the 5.4.0 weight-tying behavior change.

## Opt into downloadable kernels

- Installing `kernels` no longer automatically replaces decorated forward
  methods as of 4.53.0. `@use_kernel_forward_from_the_hub` records the kernel
  name; set `use_kernels=True` or call `kernelize` to apply it.
- Attention implementations can be changed after loading with
  `set_attn_implementation()` as of 4.54.0. A Hub reference automatically
  fetches a build matching installed CUDA and PyTorch.
- Hub kernel references accept `@revision` suffixes as of 4.56.0; pin a revision
  for reproducibility.
- Flash Attention 2 requires version 2.3.3 or newer as of 5.4.0. Initial Flash
  Attention 4 support includes a `kernels` fallback.
- Kernel integrations add `paged_attention` for continuous batching and custom
  kernels on Neuron devices in 5.4.0.
- Hub-registered expert kernels load correctly in 5.7.0. The same release fixes
  kernel configuration and error handling for FP8 checkpoints and enables the
  rotary kernel for Gemma3n and Gemma4.
- Neuron devices joined the automatic compilation hardware list in 5.6.0.

```python
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-1B-Instruct",
    use_kernels=True,
)
model.set_attn_implementation("kernels-community/flash-attn3@main")
```
