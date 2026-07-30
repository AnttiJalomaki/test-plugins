---
name: vllm-knowledge-patch
description: vLLM
version: 0.26.0
license: MIT
metadata:
  author: Nevaberry
---


# vLLM Knowledge Patch

Load this skill when building, upgrading, serving, or troubleshooting vLLM.
Use it for engine configuration, OpenAI-compatible APIs, distributed execution,
cache topology, quantization, model integration, and speculative decoding.

## Reference index

| Reference | Topics |
| --- | --- |
| [engine-runtime.md](references/engine-runtime.md) | V1 architecture, scheduling, compilation, sampling, Model Runner V2, runtime dependencies, configuration, removals |
| [serving-apis.md](references/serving-apis.md) | OpenAI-, Anthropic-, gRPC-, and Rust-facing APIs; speech, embeddings, operations, security, validation, logging |
| [distributed-cache.md](references/distributed-cache.md) | Parallelism, multi-node launch, disaggregated serving, KV connectors, offloading, prefix caching, RDMA |
| [models-adapters.md](references/models-adapters.md) | Model and task coverage, multimodal integration, LoRA, pooling, RLHF lifecycle, installation caveats |
| [quantization-hardware.md](references/quantization-hardware.md) | Quantized formats, hardware support, KV-cache formats, online quantization, plugin contracts |
| [speculative-decoding.md](references/speculative-decoding.md) | Speculation methods, configuration keys, proposers, acceptance, compatibility, vocabulary handling |

## Upgrade triage

### Treat V1 as the only engine

V1 became the supported default and V0 was subsequently removed. Do not build
new integrations around `AsyncLLMEngine`, `LLMEngine`, `MQLLMEngine`, V0
attention backends, V0 executors, or V0-only environment switches. Custom
model forward methods must obtain KV cache and attention metadata from
`forward_context`, not positional `kv_cache` or `attn_metadata` arguments.

### Recheck defaults instead of assuming continuity

- A model's `generation_config` can supply sampling and chat-template defaults.
  Pin temperature, token limits, and other behavior-critical values explicitly.
- V1 uses a deterministic default seed. Avoid assuming independent process runs
  produce different samples at nonzero temperature.
- `top_k=0` disables top-k sampling; do not generate new configuration with the
  transitional `-1` sentinel.
- Async scheduling moved from experimental and unsafe in some preemption cases
  to broadly enabled. Use `--no-async-scheduling` only after checking workload
  compatibility, especially speculation, CPU, and pipeline-parallel paths.
- `generation_config.max_tokens` is a request default, not a hard upper bound.
- KV-connector load failures default to failing the request. Configure
  recomputation explicitly if that is the intended recovery policy.

### Replace retired options and interfaces

Use current configuration surfaces:

| Retired or deprecated | Current direction |
| --- | --- |
| `VLLM_ATTENTION_BACKEND` | `--attention-backend` or `AttentionConfig` |
| `--convert reward` | `--convert embed` |
| `guided_*` request fields | Structured-output configuration |
| `--calculate-kv-scales` | Online or explicit KV-scale configuration |
| `VLLM_ALL2ALL_BACKEND` | Current expert-parallel backend configuration |
| backend environment variables | `--moe-backend` and `--linear-backend` |
| `--disable-frontend-multiprocessing` | Current frontend/executor selection |
| `score` task | Pooling or current scoring interfaces |
| `swap_space` | KV offloading configuration |
| legacy tokenizer/chat-template imports | Current exported import locations |

Also remove `num_lookahead_slots`, `best_of`, LoRA extra vocabulary,
per-request logits processors, and old compilation forms. Consult
[engine-runtime.md](references/engine-runtime.md) before carrying configuration
forward across a large upgrade.

## Engine quick reference

### Understand the V1 scheduling model

`EngineCore` isolates scheduling and model execution. Tokenization,
multimodal preprocessing, detokenization, and response streaming overlap around
that hot path. Scheduling allocates token counts per request rather than
separate prefill and decode phases, so one token budget covers chunked prefill,
prefix hits, ordinary decode, and speculative tokens.

V1 prefix caching is hash-based, LRU-evicted, and inexpensive enough to enable
by default for supported non-hybrid models. Hybrid models retain opt-in
behavior. Do not force chunked prefill or prefix caching contrary to a model's
support declaration; unsupported combinations can crash or corrupt output.

### Configure compilation intentionally

V1 integrates `torch.compile`. Current optimization levels use `-O0` through
`-O3`; the old `-O.xx` spelling is gone. CUDA graph capture and standalone
Inductor behavior changed over time, and `compile_ranges` permits selective
compilation. When debugging numerical or graph issues, record the optimization
level, capture mode, model runner, attention backend, and hardware generation.

### Account for Model Runner V2 selection

Model Runner V2 progressed from experimental to the default for dense models,
including quantized dense models. Unsupported features can fall back to the
older runner on intermediate versions, so inspect startup logs rather than
inferring the runner only from model family. The newer runner covers VLMs,
pooling, pipeline and decode-context parallelism, hybrid/Mamba paths,
speculation, logprobs, and multiple CUDA-graph capture modes.

### Use structured configuration

Dataclass-backed CLI options accept a JSON object or dotted keys. For example:

```bash
vllm serve MODEL --attention-config.flash_attn_version=2
```

Config JSON accepts decimal `k/m/g/t` and binary `K/M/G/T` size suffixes.
Nested YAML configuration is supported. Keep engine, scheduler, cache,
attention, and speculative settings in their proper nested objects so removals
and validation errors are visible.

## Serving quick reference

### Know compatibility boundaries

The OpenAI-compatible server does not implement Completions `suffix` or Chat
`image_url.detail`; Chat `user` is accepted but ignored. Setting
`parallel_tool_calls=false` enforces at most one tool call, while `true` merely
allows multiple calls. Request validation rejects invalid token IDs,
non-positive parallelism or scheduling values, non-finite sampling values, and
degenerate structured-output constraints.

Responses can be created, retrieved, and cancelled:

```text
POST /v1/responses
GET  /v1/responses/{response_id}
POST /v1/responses/{response_id}/cancel
```

Strict tool calling, reasoning/tool streaming, structured outputs, request
timings, token IDs, prompt embeddings, cache keys, and namespace tools are
version- and parser-sensitive. Read [serving-apis.md](references/serving-apis.md)
before mirroring OpenAI or Anthropic client behavior exactly.

### Separate rendering when useful

Prompt preprocessing can run apart from GPU inference. Use render endpoints or
`vllm launch render` for a GPU-less preprocessing tier, especially when
centralizing chat templates or multimodal processing.

### Treat private transports as private

The gRPC interface is explicitly a private-use surface even when TLS support is
available elsewhere. Ray cluster traffic is also unsafe to expose to untrusted
networks. Bind cluster addresses privately and place authentication and network
policy at the deployment boundary.

## Distributed and cache quick reference

### Choose parallelism for the topology

Use pipeline parallelism with tensor parallel size one when GPU count does not
divide the model cleanly or when GPUs lack a fast interconnect such as NVLink.
Native multi-node launch requires multiprocessing data parallelism and a node
count that evenly divides `DP × PP × TP`. External data-parallel balancing is
MoE-only, and fault tolerance requires an external balancer or explicit rank.

```bash
vllm serve MODEL --tensor-parallel-size 1 --pipeline-parallel-size 4
```

Startup log lines for GPU KV-cache size and maximum concurrency are capacity
estimates. Use them to validate sizing rather than relying only on GPU count.

### Make device placement explicit

Current vLLM does not rewrite `CUDA_VISIBLE_DEVICES`; use `device_ids`. Integer
IDs index an existing visibility mask, UUIDs are allowed, forms cannot be mixed,
and the option does not affect Ray executors.

### Design offloading as a tiered cache

KV offloading supports policies, hybrid allocators, multiple cache groups,
CPU, filesystem, disk, and object-store tiers. Some connectors enable the
Hybrid Memory Allocator automatically. Per-request policy, async batched lookup,
chunk sizing, workload identity, and encoder-cache offloading are available but
backend-dependent. See [distributed-cache.md](references/distributed-cache.md).

## Quantization quick reference

Quantization names do not imply universal hardware support. Verify GPU compute
capability, CPU architecture, runtime backend, linear versus MoE coverage, and
KV-cache compatibility. TurboQuant KV cache currently forces FlashAttention 2;
set that version explicitly to make the constraint clear.

Use `--quantization-config` for per-layer-kind `linear` and `moe` specs and
ignore patterns. Online shorthand can populate the structured config. Loading a
deprecated scheme requires `--allow-deprecated-quantization`.

For out-of-tree methods, register a `QuantizationConfig`, dispatch by layer
type, and provide distinct method contracts for `LinearBase` and `FusedMoE`.
Import the registration module before selecting its quantization name. Full
hardware and plugin details are in
[quantization-hardware.md](references/quantization-hardware.md).

## Speculative-decoding quick reference

Use the component shorthands `--spec-method`, `--spec-model`, and
`--spec-tokens`, or use `--speculative-config`; do not set the same field in
both. Inside the object, draft execution uses `draft_tensor_parallel_size` and
draft `max_model_len`, while temperature and top-p remain sampling parameters.

Select methods deliberately:

- `suffix` uses bounded shared state and no draft model.
- `ngram` couples omitted min/max lookup bounds.
- `mtp` is required for Gemma 4 assistant checkpoints.
- `custom_class` loads a fully qualified proposer class through `model`.
- `draft_model` is required for heterogeneous vocabularies, which are
  greedy-only and limited to tokens shared by normalized token text.

Speculation is distribution-preserving up to numerical precision, not a promise
of stable token log probabilities or bit-identical output across hardware and
batch shapes. Consult
[speculative-decoding.md](references/speculative-decoding.md) for method limits,
acceptance modes, dynamic speculation, DFlash, TLI, DSpark, and weight updates.

## Working procedure

1. Inspect the installed vLLM version, model manifest, hardware, runner choice,
   and server frontend.
2. Identify whether the task concerns engine/runtime, serving, distribution,
   adapters/models, quantization, or speculation.
3. Load the matching reference files from the index; multiple areas often
   interact through attention, cache, and parallelism settings.
4. Prefer manifest, runtime help, startup validation, code, and tests over
   assumptions when a deployment differs from the documented path.
5. Make defaults explicit in production configuration and preserve startup logs
   with the deployed configuration.
6. Test representative requests: long context, preemption, structured output,
   tool streaming, multimodal data, adapter loading, and failure recovery as
   applicable.
