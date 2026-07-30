# Quantization and Hardware

## Hardware boundaries

- AWQ supports NVIDIA Turing through Hopper, Intel GPU, and x86 CPU. GPTQ
  additionally supports Volta.
- Marlin requires NVIDIA Turing or newer; MXFP4 is unavailable on Turing.
- `llm-compressor` INT8 W8A8 supports NVIDIA Turing or newer plus x86 and Arm
  CPU. Its INT8 W4A8 path is Arm-only; FP8 W8A8 supports Ada, Hopper, and AMD.
- bitsandbytes and DeepSpeedFP support NVIDIA Volta or newer. GGUF supports
  those NVIDIA generations plus AMD.
- Volta, Turing, Ampere, Ada, and Hopper are SM 7.0, 7.5, 8.0/8.6, 8.9, and
  9.0 respectively.
- Gaudi quantization is maintained in vLLM-Gaudi; TPU compatibility is
  documented separately.
- TPU gained W8A8 in `0.7-0.10`; CPU gained FP8 KV cache, AMD gained V1 FP8 KV
  cache, TPU gained FP8 KV quantization, and Arm CPU gained INT8.
- Version 0.18 has a known accuracy degradation when Qwen3.5 uses FP8 KV cache
  on B200.

## Quantized checkpoint and online-format growth

- In `0.7-0.10`, checkpoint support added ModelOpt FP4, NVFP4, GPTQAllSpark,
  DeepSeek GGUF, Quark MXFP4, torchao `AOPerModuleConfig`,
  `nvidia/DeepSeek-R1-FP4`, MXFP4 MoE, Mixtral and broader MoE
  BitsAndBytes, and in-flight MoE quantization.
- FP4 emulation was removed; pre-SM100 devices fall back to Marlin.
- In `0.11-0.14`, formats added dense NVFP4, per-token-group and blocked-MoE
  FP8, Turing AWQ compressed tensors, Hopper W4A8 grouped GEMM, online FP8
  streaming/reload, MoE AWQ/GPTQ Marlin, Turing Marlin, Quark int4-FP8 W4A8
  MoE, dense MXFP4 W4A16, ModelOpt FP8 variants, and NVFP4 Marlin.
- Version 0.14 removed deprecated schemes and made Marlin the default MXFP4
  LoRA backend.
- In `0.15-0.18`, formats added compressed-tensors MXFP4 W4A16 MoE, per-head
  FP8 KV scales, block-FP8 W8A16, dense/MoE ModelOpt MXFP8, directly loaded
  quantized LoRA, mixed-precision ModelOpt, and ROCm Quark W4A8 MXFP4/FP8.
- In `0.19-0.22`, online MXFP8 arrived for dense/MoE, moved into the general
  online-quantization frontend, and TurboQuant 2-bit KV gained FA3/FA4
  prefill. TurboQuant then expanded to hybrid models and NVFP4 KV cache
  arrived.
- The same batch added CPU W4A16 and XPU W4A8 compressed tensors, ROCm AWQ
  Marlin, compressed-tensors W8A8 MXFP8, ModelOpt NVFP4 W4A16, MXFP4
  linear-layer loading, Quark NVFP4, and AutoRound W4A16.
- Version 0.22 consolidated `gptq_marlin` under `auto_gptq`.
- In `0.23-0.26`, additions include ModelOpt LM-head/non-gated-MoE MXFP8,
  compressed-tensors WNA8O8Int linears, WNInt embeddings, asymmetric MoE
  WNA16, online FP8 per-token-per-channel quantization, `fp8_e5m2` KV with
  non-FP8 checkpoints, Humming packed 2/3/5/6/7-bit weight-only,
  W2-W7/A4-A8 compressed tensors, Triton INT4 per-token-head KV, XPU INT2
  weight-only linear, and `nvfp4_per_token` online MoE.
- GGUF quantization moved to a plugin in 0.24.

## Model loading formats

- Version 0.12 accepts `repo_id:quant_type` for GGUF, auto-detects Mistral
  format, and loads multimodal Gemma 3 GGUF.
- `--quantization-config` provides per-layer-kind `QuantSpec` values for
  `linear` and `moe`, plus ignore patterns.
- An online quantization shorthand in `--quantization` populates the
  structured config. `--allow-deprecated-quantization` is required to permit
  deprecated schemes.
- TurboQuant KV formats currently override an unset or FlashAttention 3+
  selection to FlashAttention 2. Set
  `--attention-config.flash_attn_version=2` explicitly to avoid the warning.

## Registering an out-of-tree method

Decorate a `QuantizationConfig` subclass with
`@register_quantization_config("name")`. It must implement:

- `get_name`
- `get_supported_act_dtypes`
- `get_min_capability`
- `get_config_filenames`
- `from_config`
- `get_quant_method`

`get_quant_method` dispatches by layer type and returns a method or `None`.
Import the registration module before selecting the method:

```python
import my_quant_plugin
from vllm import LLM

llm = LLM(model="your-model", quantization="my_quant")
```

## Linear plugin contract

For `LinearBase`, return a `QuantizeMethodBase`; start from
`UnquantizedLinearMethod` when useful. Weight creation receives metadata and
application receives an optional bias:

```python
class MyQuantLinearMethod(UnquantizedLinearMethod):
    def create_weights(self, layer, *weight_args, **extra_weight_attrs):
        ...

    def apply(self, layer, x, bias=None):
        ...
```

## Fused-MoE plugin contract

For `FusedMoE`, return a `FusedMoEMethodBase` initialized from
`layer.moe_config`, or `UnquantizedFusedMoEMethod` to leave it unquantized.
A custom implementation supplies weight creation, routed application, and a
`FusedMoEQuantConfig`:

```python
class MyQuantMoEMethod(FusedMoEMethodBase):
    def create_weights(
        self, layer, num_experts, hidden_size,
        intermediate_size_per_partition, params_dtype, **extra_weight_attrs,
    ):
        ...

    def apply(self, layer, router, x, router_logits):
        ...

    def get_fused_moe_quant_config(self, layer):
        ...
```

## Hardware and platform growth

- In `0.7-0.10`, runtime support added native Apple Silicon, out-of-tree
  platforms, s390x CPU, PPC64LE, and Arm V1.
- Version 0.11 added RISC-V 64-bit CPU.
- Version 0.13 added SM103/GB300 with CUDA 13, CUDA 13 AArch64 wheels, CPU
  Whisper, and an x86 CPU wheel pipeline.
- Version 0.20 added Python 3.14. Later CUDA, ROCm, XPU, compiler, and wheel
  baselines are tracked in
  [operations and migrations](operations-and-migrations.md).
