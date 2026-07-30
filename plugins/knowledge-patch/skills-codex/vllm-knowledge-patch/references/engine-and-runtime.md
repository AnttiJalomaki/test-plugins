# Engine and Runtime

## Engine migration and architecture

- In `0.7-0.10`, V1 moved from `VLLM_USE_V1=1` opt-in to the supported-use-case
  default; `VLLM_USE_V1=0` remained a temporary escape hatch. The V0 CPU, XPU,
  TPU, and HPU backends, long-context LoRA, Phi3-Small and BlockSparse
  Attention, and speculative-decoding workers were removed during that
  transition.
- In `0.11-0.14`, V0 was removed completely, including `AsyncLLMEngine`,
  `LLMEngine`, `MQLLMEngine`, V0 attention backends, executors, and remaining
  interfaces. V1 is the only engine.
- V1 isolates the scheduler and model executor in `EngineCore`. Tokenization,
  multimodal preprocessing, detokenization, and response streaming overlap
  outside the hot path. Workers cache request state and receive incremental
  updates, so single-GPU and tensor-parallel execution share a symmetric
  worker design.
- Scheduling is token-based, conceptually `{request_id: num_tokens}`, rather
  than split into prefill and decode phases. One token budget covers ordinary
  decode, chunked prefill, cached prefixes, and speculative tokens.
- V1 prefix caching uses hash lookup, constant-time LRU eviction, and low
  allocation overhead. It is generally enabled by default even with no cache
  hits, subject to the hybrid-model support rules described below.

## Compilation, CUDA graphs, and attention

- V1 integrated `torch.compile` and enabled it by default in `0.7-0.10`; `-O3`
  also enabled it explicitly.
- Version 0.11 made `FULL_AND_PIECEWISE` the default CUDA graph mode and
  disabled standalone Inductor compilation. Versions 0.12-0.13 exposed `-O0`
  through `-O3`, added `compile_ranges`, and removed the deprecated `-O.xx`
  spelling.
- Model Runner V2 began as an experimental GPU runner in 0.12. Through
  0.14 it gained M-RoPE, `logit_bias`, `allowed_token_ids`, and `min_tokens`.
  Through
  `0.15-0.18` it gained VLMs, pipeline and decode-context parallelism,
  piecewise and mixed CUDA graph capture, pooling, Whisper state, and
  probabilistic rejection sampling.
- In `0.19-0.22`, MRV2 added EPLB, multimodal speculative embeddings,
  greedy/logprob rejection modes, multiple prompt-logprobs, and
  Qwen3.5/Mamba-hybrid support. It became the dense-Qwen3 default with
  automatic MRV1 fallback for unsupported features.
- In `0.23-0.26`, MRV2 became the default first for dense Llama/Mistral, then
  quantized models, and finally every dense model. It gained EVS,
  Mamba-hybrid prefix caching, and dynamic speculation under full CUDA
  graphs; the legacy PagedAttention implementation was deleted.
- Attention configuration moved from `VLLM_ATTENTION_BACKEND` to
  `--attention-backend` and `AttentionConfig` in 0.13; `LLM` accepts
  `attention_config`. Version 0.18 added `--attention-backend auto`.
- On Blackwell, 0.15 defaulted MLA to FlashInfer and prefill to TRTLLM.
  Cascade attention became disabled by default in 0.18.
- Version 0.19 defaulted FP8-KV sparse MLA to FlashInfer; 0.20 made
  FlashAttention 4 the SM90+ MLA-prefill default, and 0.21 enabled the
  FlashInfer top-k/top-p sampler by default.
- Version 0.26 permits a distinct attention backend per KV-cache group and
  makes sliding-window support an explicit backend capability. Hybrid models
  can therefore mix backends.
- Version 0.26 adds `head_dtype`, including through LoRA, so generation
  `lm_head` weights can remain FP32 for accuracy.

## Async scheduling and performance modes

- Version 0.10 introduced experimental `--async-scheduling` to overlap
  engine-core scheduling with GPU execution.
- Version 0.11 and 0.10.2 could corrupt output under preemption and some
  other async-scheduling cases. Version 0.14 enabled it by default except
  with pipeline parallelism, CPU, and non-MTP/non-Eagle speculation;
  `--no-async-scheduling` opted out.
- In `0.15-0.18`, async scheduling became compatible and then fully supported
  with pipeline parallelism. N-gram speculation also became compatible with
  it.
- Version 0.17 added
  `--performance-mode {balanced,interactivity,throughput}`. Throughput mode
  doubles automatic token or sequence defaults that were not explicitly
  overridden.
- `--max-model-len auto` chooses a context length that fits GPU memory.
  `VLLM_LOG_MODEL_INSPECTION=1` or printing an `LLM` instance reports modules,
  attention backends, and quantization.

## Automatic scheduler sizing

- GPUs with at least 70 GiB other than A100 default to 16,384/8,192 batched
  tokens for offline/server use and 1,024 sequences. Other GPUs use
  8,192/2,048 tokens and 256 sequences.
- CPU defaults are 4,096 tokens/256 sequences offline and 2,048/128 for the
  server, multiplied by `PP × TP`.
- TPU V6E, V5E, and V5P token defaults are respectively 2,048/1,024,
  1,024/512, and 512/256 for offline/server use.
- Without chunked prefill, an unspecified batched-token limit is raised to
  at least the context length. A multimodal prefix-LM can raise it enough for
  its largest single media item.
- The derived token limit is capped at `max_num_seqs × max_model_len`; an
  unspecified sequence limit is capped at the resulting token count.
- Prefix caching defaults on only when the model declares support and is not
  hybrid. Chunked prefill follows model support. Disabling it for a supported
  generation model or enabling it for an unsupported pooling model can crash
  or corrupt output. RISC-V forces both features off.

## Sampling and generation defaults

- In `0.7-0.10`, the default seed became `None`, then V1 changed it to `0`,
  making separate runs deterministic at nonzero temperature. Caller RNG
  isolation does not apply with `VLLM_USE_V1_MULTIPROCESSING=0`.
- From 0.9, `top_k=0` disables top-k; `-1` remained temporarily accepted.
- A model's `generation_config` supplies chat-template and sampling defaults,
  including temperature. Pin values to keep behavior stable across model or
  engine upgrades.
- Version 0.11 returned prompt logprobs for every token and interpreted
  `logprobs=-1` as the full vocabulary. Version 0.12 moved flat-logprob
  control into `SamplingParams` and deprecated `seed=None`.
- Version 0.14 rejects unsupported speculative-decoding sampling parameters.
- From 0.17, `generation_config.max_tokens` is a default, not a hard maximum;
  an explicit request may exceed it.
- Lifecycle APIs added over time include `LLM.sleep`, `LLM.wake_up`,
  `LLM.collective_rpc`, `LLM.reset_prefix_cache`, `LLM.apply_model`,
  pause/resume generation, and enqueue/wait sleep level 0.
- Runtime post-training controls include weight reload, configuration update,
  a selectable logprobs processing stage, native NCCL and IPC weight sync,
  layerwise reload, and pause/resume that preserves requests.

## Custom model and input contracts

- Custom forward methods no longer receive `kv_cache` or `attn_metadata`;
  attention backends obtain both through `forward_context`.
- The former `SupportsV0Only` protocol was a migration declaration and is not
  a path to a current V0 engine.
- Version 0.11 added prompt embeddings and sharded-state loading; 0.12 added
  audio embeddings in Chat Completions.
- Version 0.15 accepts async generators of `StreamingInput` while preserving
  KV-cache alignment, enabling session-oriented incremental input such as ASR.
- Weight offloading V2 gained prefetch, selective CPU offload, and pinned-copy
  behavior that avoids doubling host memory.

## Engine and CLI configuration mechanics

- Dataclass-backed flags accept a complete JSON object or dotted keys, for
  example `--attention-config.flash_attn_version=2`.
- Bare JSON values accept decimal `k/m/g/t` and binary `K/M/G/T` size
  suffixes. Human-readable integers also work for batch-token,
  scheduled-token, KV-memory, and safetensors-prefetch block sizes.
  For example: `--kv-transfer-config '{"cpu_bytes_to_use":80m}'`.
- `--spec-method`, `--spec-model`, and `--spec-tokens` populate fields in
  `--speculative-config`; shorthand and JSON must not both set a field.
  Automatic speculator detection is skipped for cloud-storage model URIs.
- `--device-ids` indexes an existing device-visibility mask when passed
  integers. UUIDs are accepted, but cannot be mixed with integers; duplicates
  are rejected and Ray executors ignore the option.
- Starting in 0.24, vLLM does not mutate `CUDA_VISIBLE_DEVICES`; use
  `device_ids`. ROCm also began deprecating the environment variable.
- In Hugging Face offline mode, repository IDs for non-cloud models and
  tokenizers resolve to revision-specific local paths. Cloud URIs remain
  unchanged. `EngineArgs(tokens_only=True)` skips tokenizer initialization.
- `--enable-log-requests` logs IDs, parameters, and LoRA requests at INFO,
  plus prompts or token IDs at DEBUG. `VLLM_LOGGING_LEVEL` selects detail,
  `--aggregate-engine-logging` aggregates DP engine statistics, and
  `--fail-on-environ-validation` makes environment validation fatal.
