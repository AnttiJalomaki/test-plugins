# Engine, Runtime, and Configuration

## V1 engine and scheduling

### Default and removal path (`0.7-0.10`, `0.11-0.14`)

V1 was opt-in with `VLLM_USE_V1=1` in 0.7 and became the default for supported
uses in 0.8; `VLLM_USE_V1=0` selected V0 only while V0 remained available. The
0.10 cleanup removed the V0 CPU, XPU, TPU, and HPU backends, long-context LoRA,
Phi3-Small and BlockSparse Attention, and the V0 speculative-decoding workers.
Version 0.11 then removed `AsyncLLMEngine`, `LLMEngine`, `MQLLMEngine`, V0
attention backends and executors, and the remaining V0 interfaces. V1 is the
only engine after that removal.

### EngineCore architecture (`v1-architecture-and-batching`)

V1 puts the scheduler and model executor in an isolated `EngineCore`.
Tokenization, multimodal preprocessing, detokenization, and response streaming
can overlap outside it. Workers cache request state and receive incremental
updates, giving single-GPU and tensor-parallel execution the same symmetric
worker design.

Scheduling is token-based, not divided into prefill and decode phases. Each
step resembles `{request_id: num_tokens}`; one budget therefore represents
normal decoding, chunked prefill, prefix-cache hits, and speculative decoding.

### Compilation and CUDA graphs

- In `0.7-0.10`, `torch.compile` became fully integrated and enabled by default
  in V1; `-O3` explicitly enabled it. Experimental `--async-scheduling` arrived
  in 0.10 to overlap engine-core scheduling with the GPU runner.
- In `0.11-0.14`, `FULL_AND_PIECEWISE` became the default CUDA graph mode and
  standalone Inductor compilation was disabled. Optimization levels `-O0`
  through `-O3` express startup/performance tradeoffs. `compile_ranges` enables
  selective compilation, and the deprecated `-O.xx` form was removed.
- In `0.15-0.18`, Model Runner V2 added piecewise and mixed CUDA-graph capture.
- In `0.23-0.26`, Model Runner V2 added dynamic speculation under full CUDA
  graphs.

### Async scheduling compatibility

`--async-scheduling` in 0.10.2 and 0.11 can corrupt output under preemption and
some other cases. Version 0.14 enables it by default except with pipeline
parallelism, CPU, and speculation methods other than MTP/Eagle;
`--no-async-scheduling` opts out. Version 0.15 makes async scheduling compatible
with pipeline parallelism and 0.16 marks the combination fully supported. NGram
speculation can use async scheduling from 0.18, and 0.19 adds zero-bubble
overlap for broader speculative decoding.

### Model Runner V2 progression

The experimental GPU Model Runner V2 arrived in 0.12. By 0.14 it supported
M-RoPE, `logit_bias`, `allowed_token_ids`, and `min_tokens`, but remained off by
default. Across `0.15-0.18`, it gained VLMs, pipeline and decode-context
parallelism, piecewise and mixed CUDA graphs, pooling models, Whisper state, and
probabilistic rejection sampling.

Across `0.19-0.22`, it added EPLB, multimodal speculative embeddings,
greedy/logprob rejection-sampling modes, multiple prompt logprobs, and
Qwen3.5/Mamba hybrid support. It became the default for dense Qwen3 models in
0.22 with automatic fallback when an unsupported feature was requested.

In `0.23-0.26`, the default expanded to dense Llama and Mistral models in 0.23,
quantized models in 0.24, and every dense model in 0.25. Version 0.25 deleted the
legacy PagedAttention implementation and added EVS, Mamba-hybrid prefix caching,
and full-graph dynamic speculation to the newer runner.

## Sampling and generation behavior

### Seeds, top-k, and model defaults (`0.7-0.10`)

The default seed became `None` in 0.8, then V1 changed its default to `0` in
0.9. Separate runs can therefore be deterministic even with nonzero
temperature. Caller RNG isolation does not apply with
`VLLM_USE_V1_MULTIPROCESSING=0`. From 0.9, `top_k=0` disables top-k sampling;
`-1` remained temporarily accepted.

Starting in 0.8, the model's `generation_config` supplies defaults for sampling
and chat-template settings such as temperature. Pin behavior-sensitive values
because a model or vLLM upgrade can otherwise change an unchanged request.

### Logprobs and validation (`0.11-0.14`)

Prompt logprobs are returned for every token from 0.11, and `logprobs=-1`
requests the full vocabulary. Version 0.12 moved flat-logprob control from an
environment variable to `SamplingParams` and deprecated `seed=None`. Version
0.14 rejects unsupported speculative-decoding sampling parameters instead of
silently ignoring them.

### Token-limit meaning (`0.15-0.18`)

From 0.17, `generation_config.max_tokens` supplies a default rather than a hard
ceiling; an explicit request value may exceed it.

## Runtime input and lifecycle

Version 0.11 adds CPU KV-cache offloading with LRU management, prompt
embeddings, sharded state loading, and `LLM.apply_model`. Version 0.12 adds
pause/resume generation for asynchronous reinforcement-learning training and
audio embeddings in Chat Completions.

Version 0.15 accepts async generators of `StreamingInput` objects while
preserving KV-cache alignment. This provides session-oriented incremental input
for workloads such as ASR.

Mamba and hybrid models can cache block-aligned states with
`--enable-prefix-caching --mamba-cache-mode align`; speculative decoding works
with the alignment mode from 0.17.

## Attention and performance controls

Version 0.13 replaced `VLLM_ATTENTION_BACKEND` with `--attention-backend` and
introduced `AttentionConfig`; version 0.14 also accepts `attention_config` in
`LLM`. Version 0.17 adds `--performance-mode
{balanced,interactivity,throughput}`, and 0.18 adds
`--attention-backend auto`.

On Blackwell, version 0.15 defaults MLA to FlashInfer and prefill to TRTLLM.
Version 0.18 disables cascade attention by default. Version 0.19 defaults sparse
MLA with FP8 KV cache to FlashInfer and enables `tcmalloc` by default on CPU.
Version 0.20 makes FlashAttention 4 the default MLA prefill backend on SM90+;
0.21 enables RayExecutorV2 and the FlashInfer top-k/top-p sampler by default.

Version 0.26 allows each KV-cache group to use a different attention backend
and makes sliding-window support an explicit backend capability. Mixed-backend
hybrid models can therefore select per-group implementations.

Generation models can set `head_dtype` from 0.26 to retain the `lm_head` in
FP32 for accuracy. The setting also applies through LoRA.

## Configuration mechanics (`engine-and-openai-server`)

Dataclass-backed options accept a complete JSON object or dotted keys such as
`--attention-config.flash_attn_version=2`. Bare JSON values accept decimal
`k/m/g/t` and binary `K/M/G/T` suffixes; human-readable integers also work for
batch-token, scheduled-token, KV-memory, and safetensors-prefetch block sizes.

### Scheduler defaults

On GPUs with at least 70 GiB other than A100, offline/server defaults are
16,384/8,192 batched tokens and 1,024 sequences. Other GPUs use 8,192/2,048
tokens and 256 sequences. CPU token/sequence defaults are 4,096/256 offline and
2,048/128 for the server, multiplied by `PP × TP`. TPU V6E, V5E, and V5P token
defaults are 2,048/1,024, 1,024/512, and 512/256 offline/server. Throughput mode
doubles token or sequence defaults that were not explicitly overridden.

Without chunked prefill, an unspecified batched-token limit is raised to at
least the model context length. A multimodal prefix-LM can raise it again to fit
its largest single media item. The result is capped at
`max_num_seqs × max_model_len`; an unspecified sequence limit is capped at the
final batched-token count.

Prefix caching defaults on only when the model supports it and is not hybrid;
chunked prefill follows the model's support declaration. Disabling supported
chunked prefill on a generation model or enabling it on an unsupported pooling
model can crash or corrupt output. RISC-V CPU forces both features off.

### Model context and inspection

Version 0.14 accepts `--max-model-len auto` to fit the context to available GPU
memory. Set `VLLM_LOG_MODEL_INSPECTION=1` or print an `LLM` object to inspect
modules, attention backends, and quantization.

### Offline identifiers and tokenizer control

In Hugging Face offline mode, non-cloud model and tokenizer repository IDs are
resolved to revision-specific local paths. Cloud-storage URIs are unchanged.
`EngineArgs(tokens_only=True)` independently skips tokenizer initialization.

## Runtime, wheels, and build baseline

### `0.7-0.10`

Version 0.8 moved to PyTorch 2.6 and CUDA 12.4 wheels. Version 0.9 moved to
PyTorch 2.7, made CUDA 12.8 the default, dropped CUDA 12.4, and offered CUDA
12.6 as a GitHub artifact; the CUDA 12.8 flow supports
`--torch-backend=auto`. Version 0.10 uses PyTorch 2.7.1.

### `0.11-0.14`

Version 0.11 moved CPU builds to PyTorch 2.8 and ROCm to 7.0. Version 0.12
requires PyTorch 2.9.0 with CUDA 12.9 and Transformers 4.57.3. Version 0.14
requires PyTorch 2.9.1 and its default wheel uses CUDA 12.9 (`cu129`).

### `0.15-0.18`

Version 0.17 upgrades to PyTorch 2.10.0; an updated wheel fixes the noted CUDA
12.9+ library mismatch. XPU moves from IPEX to `vllm-xpu-kernels`, ROCm renames
`aiter` to `amd-aiter`, and 0.18 removes Ray from default dependencies. Install
Ray explicitly when selecting it.

### `0.19-0.22`

Version 0.20 makes CUDA 13.0 the default PyPI wheel and OpenAI-compatible image,
upgrades CUDA and XPU builds to PyTorch 2.11, adds Python 3.14, and requires
`transformers>=5`. For CUDA 12.9, use `uv` with `--torch-backend=cu129`.
Version 0.21 deprecates Transformers v4 and requires a C++20 compiler.

CUDA 13.0 wheels in 0.21 and CUDA 12.9 wheels in 0.22 use PyTorch's
`manylinux_2_28` base. Version 0.22 adds a non-root `vllm-openai` Docker target
and permits Python-only installation.

### `0.23-0.26`

Version 0.24 moves ROCm to PyTorch 2.11, XPU to torch-xpu 2.12, CUDA container
builds to GCC 12, and makes `mistral_common` optional. Starlette must be at
least 1.0.1 for its security fix. Version 0.26 moves the Transformers
integration to 5.13.0.

## Removed and deprecated configuration

- Version 0.12 removes `num_lookahead_slots`, `best_of`, and LoRA extra
  vocabulary; deprecates xFormers; and deprecates direct-child EPLB fields on
  `ParallelConfig`, `guided_*`, `override_pooler_config`,
  `disable_log_requests`, `CompilationConfig.use_inductor`, and already
  deprecated metrics.
- Version 0.13 removes deprecated plugin, compilation, task, seed, and
  multimodal settings; `embed_input_ids`/`embed_multimodal` fallbacks; and the
  tokenizer setter. Use `--convert embed` rather than `--convert reward`.
  Version 0.14 deprecates `seed_everything`.
- Version 0.15 removes `vllm:time_per_output_token_seconds` in favor of
  `vllm:inter_token_latency_seconds`, plus DeepSpeedFp8 and RTN. Version 0.16
  removes BitBlas, Marlin 24, `reasoning_content`, and
  `VLLM_ALL2ALL_BACKEND`; 0.17 removes per-request logits processors; 0.18
  removes `swap_space`.
- Version 0.19 deprecates `--calculate-kv-scales`, the `score` task, virtual
  engines, and `--disable-frontend-multiprocessing`, while removing
  per-tensor/per-channel FP8 and Sparse24 integration.
- Version 0.22 removes old import locations for `get_tokenizer` and
  `resolve_hf_chat_template` plus deprecated MLA-prefill arguments. Use
  `--moe-backend` and `--linear-backend` instead of their backend environment
  variables.
