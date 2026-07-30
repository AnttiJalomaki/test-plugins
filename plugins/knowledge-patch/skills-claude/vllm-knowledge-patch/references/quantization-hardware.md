# Quantization Formats, Hardware, and Extensions

## Hardware support boundaries (`quantization`)

Quantization support is narrower than format names suggest:

| Method | Supported hardware |
| --- | --- |
| AWQ | NVIDIA Turing through Hopper, Intel GPU, x86 CPU |
| GPTQ | NVIDIA Volta through Hopper, Intel GPU, x86 CPU |
| Marlin | NVIDIA Turing or newer; MXFP4 is unavailable on Turing |
| `llm-compressor` INT8 W8A8 | NVIDIA Turing or newer, x86 CPU, Arm CPU |
| `llm-compressor` INT8 W4A8 | Arm CPU only |
| `llm-compressor` FP8 W8A8 | NVIDIA Ada/Hopper and AMD |
| bitsandbytes, DeepSpeedFP | NVIDIA Volta or newer |
| GGUF | NVIDIA Volta or newer and AMD |

NVIDIA Volta, Turing, Ampere, Ada, and Hopper map to SM 7.0, 7.5, 8.0/8.6,
8.9, and 9.0. Gaudi quantization lives in vLLM-Gaudi; TPU compatibility is
documented separately.

## Per-layer configuration (`engine-and-openai-server`)

`--quantization-config` accepts `QuantSpec` settings by `linear` and `moe`
layer kind plus ignore patterns. Selecting an online shorthand through
`--quantization` automatically fills this structured configuration.
`--allow-deprecated-quantization` is required to choose a deprecated scheme.

When the resolved KV-cache dtype is TurboQuant, an unset FlashAttention version
or a version of 3 or newer is overridden to FlashAttention 2. Set
`--attention-config.flash_attn_version=2` explicitly to avoid the warning.

## Checkpoint and kernel expansion

### `0.7-0.10`

Checkpoint support expanded across 0.8-0.9 to:

- ModelOpt FP4, NVFP4, and `nvidia/DeepSeek-R1-FP4`;
- GPTQAllSpark and DeepSeek GGUF;
- Quark MXFP4;
- torchao models using `AOPerModuleConfig`.

Version 0.10 adds MXFP4 for MoE, BitsAndBytes for Mixtral and more MoE models,
and in-flight MoE quantization. FP4 emulation is removed; pre-SM100 hardware
falls back to Marlin.

Hardware-specific additions include TPU W8A8 in 0.7, CPU FP8 KV cache in 0.8,
AMD FP8 KV cache on V1 in 0.9, and TPU FP8 KV-cache quantization plus Arm CPU
INT8 in 0.10.

### `0.11-0.14`

The added formats and kernels include:

- dense NVFP4;
- per-token-group and blocked-MoE FP8;
- Turing AWQ compressed tensors;
- Hopper W4A8 grouped GEMM;
- online FP8 streaming and runtime weight reload;
- MoE AWQ/GPTQ Marlin and Turing Marlin;
- Quark int4-FP8 W4A8 MoE;
- dense MXFP4 W4A16;
- new ModelOpt FP8 variants;
- NVFP4 Marlin.

Version 0.14 removes deprecated quantization schemes and makes Marlin the
default MXFP4 LoRA backend. GGUF selectors can use `repo_id:quant_type` from
0.12; Mistral-format and multimodal Gemma 3 GGUF are auto-detected.

### `0.15-0.18`

This range adds compressed-tensors MXFP4 W4A16 MoE, per-head FP8 KV-cache
scales, block-FP8 W8A16, dense and MoE ModelOpt MXFP8, directly loaded
quantized LoRA, mixed-precision ModelOpt, and ROCm Quark W4A8 MXFP4/FP8 paths.

DeepSpeedFp8 and RTN are removed in 0.15; BitBlas and Marlin 24 are removed in
0.16. Qwen3.5 with an FP8 KV cache on B200 has a known degraded-accuracy issue
in 0.18.

### `0.19-0.22`

Version 0.19 adds online MXFP8 for dense and MoE models. Version 0.20 moves it
into a general online-quantization frontend and adds TurboQuant 2-bit KV cache
with FlashAttention 3/4 prefill. Version 0.21 extends TurboQuant to hybrid
models and adds NVFP4 KV cache.

Additional checkpoint and kernel support includes:

- CPU W4A16 and XPU W4A8 compressed tensors;
- ROCm AWQ Marlin;
- compressed-tensors W8A8 MXFP8;
- ModelOpt NVFP4 W4A16;
- MXFP4 linear-layer loading;
- Quark NVFP4 checkpoints;
- AutoRound W4A16.

Version 0.22 consolidates `gptq_marlin` under `auto_gptq`. Version 0.19 removes
per-tensor/per-channel FP8 and Sparse24 integration and deprecates
`--calculate-kv-scales`.

### `0.23-0.26`

Version 0.23 adds ModelOpt LM-head and non-gated-MoE MXFP8, plus
compressed-tensors WNA8O8Int linear layers, WNInt embeddings, and asymmetric
MoE WNA16.

Version 0.24 adds online FP8 per-token-per-channel quantization, permits
`fp8_e5m2` KV cache with non-FP8 checkpoints, and moves GGUF quantization into
a plugin.

Versions 0.25-0.26 add:

- Humming packed 2-, 3-, 5-, 6-, and 7-bit weight-only formats;
- W2-W7/A4-A8 compressed-tensors paths;
- Triton INT4 per-token-head KV cache;
- XPU INT2 weight-only linear layers;
- `nvfp4_per_token` online MoE quantization.

Version 0.25 drops `gptq_marlin` on ROCm and deprecates the old online FP8 MoE
class.

## Host and accelerator coverage

Native Apple Silicon and out-of-tree platform support arrived in 0.7, s390x CPU
inference in 0.8, and PPC64LE plus Arm V1 in 0.10 (`0.7-0.10`).

Version 0.11 adds RISC-V 64-bit CPU. Version 0.13 adds SM103/GB300 with CUDA 13,
CUDA 13 AArch64 wheels, CPU Whisper, and an x86 CPU wheel pipeline
(`0.11-0.14`). RISC-V CPU forces prefix caching and chunked prefill off.

## Out-of-tree quantization (`quantization`)

### Register the configuration

Decorate a `QuantizationConfig` subclass with
`@register_quantization_config("name")`. Implement:

- `get_name`
- `get_supported_act_dtypes`
- `get_min_capability`
- `get_config_filenames`
- `from_config`
- `get_quant_method`

`get_quant_method` dispatches on layer type and returns a quantization method or
`None`. Import the module containing the registration before selecting its
name:

```python
import my_quant_plugin
from vllm import LLM

llm = LLM(model="your-model", quantization="my_quant")
```

### Linear method contract

For `LinearBase`, return a `QuantizeMethodBase`; `UnquantizedLinearMethod` is a
usable starting point. Weight creation receives metadata, and application
receives an optional bias:

```python
class MyQuantLinearMethod(UnquantizedLinearMethod):
    def create_weights(self, layer, *weight_args, **extra_weight_attrs):
        ...

    def apply(self, layer, x, bias=None):
        ...
```

### Fused-MoE method contract

For `FusedMoE`, return a `FusedMoEMethodBase` initialized from
`layer.moe_config`, or use `UnquantizedFusedMoEMethod` to leave MoE
unquantized. A custom implementation supplies creation, routed application, and
its `FusedMoEQuantConfig`:

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
