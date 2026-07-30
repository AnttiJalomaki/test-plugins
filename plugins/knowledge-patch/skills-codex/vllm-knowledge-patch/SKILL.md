---
name: vllm-knowledge-patch
description: vLLM
version: 0.26.0
license: MIT
metadata:
  author: Nevaberry
---


# vLLM Knowledge Patch

Use this patch when changing a vLLM engine, server, deployment, model
integration, or extension. Start with the quick reference, then read every
topic file relevant to the change.

## Reference index

| Reference | Topics |
|---|---|
| [Engine and runtime](references/engine-and-runtime.md) | V1, Model Runner V2, compilation, scheduling, sampling, configuration, custom models |
| [Serving APIs](references/serving-apis.md) | HTTP, Responses, Chat, Completions, embeddings, speech, parsers, Rust and gRPC frontends |
| [Distributed execution and caching](references/distributed-and-caching.md) | DP/TP/PP/EP, Ray, multiprocessing, KV connectors, prefix caching, offloading, RDMA |
| [Quantization and hardware](references/quantization-and-hardware.md) | Checkpoint formats, online quantization, KV cache, hardware boundaries, custom plugins |
| [Speculative decoding](references/speculative-decoding.md) | Draft models, suffix and n-gram methods, heterogeneous vocabularies, acceptance behavior |
| [Models, multimodal, and LoRA](references/models-multimodal-and-lora.md) | Model/task support, processors, pooling, adapters, model-specific caveats |
| [Operations and migrations](references/operations-and-migrations.md) | Dependencies, wheels, CLI, metrics, security, removals, validation |

## Breaking changes first

### Treat V1 as the only engine

- Do not import or design around `AsyncLLMEngine`, `LLMEngine`,
  `MQLLMEngine`, V0 executors, or V0 attention backends.
- Do not pass `kv_cache` or `attn_metadata` to a custom model's forward
  method. Read both from `forward_context`.
- V1 schedules token budgets, not distinct prefill and decode phases.
  Extensions must tolerate chunked prefill, cached tokens, and speculative
  tokens in the same scheduling model.
- Prefix caching is normally inexpensive and enabled, but hybrid models keep
  it opt-in. Respect the model's support declarations for both prefix caching
  and chunked prefill.

### Audit removed parameters and interfaces

- Remove `num_lookahead_slots`, `best_of`, LoRA extra vocabulary,
  `swap_space`, per-request logits processors, and the old tokenizer setter.
- Replace `--convert reward` with `--convert embed`.
- Replace backend environment variables with configuration or CLI fields,
  including `--attention-backend`, `--moe-backend`, and `--linear-backend`.
- Use current import locations for `get_tokenizer` and
  `resolve_hf_chat_template`.
- Do not rely on the legacy PagedAttention implementation, deprecated model
  registry entries, retired connectors, or removed quantization schemes.

### Make sampling explicit

- A model's `generation_config` can supply chat-template and sampling
  defaults. Pin temperature, token limits, and related settings when stable
  behavior matters.
- Use `top_k=0` to disable top-k sampling.
- Avoid `seed=None`; V1 uses deterministic seed behavior, with a caller-RNG
  caveat when V1 multiprocessing is disabled.
- `generation_config.max_tokens` is a default, not a hard ceiling.
- Unsupported speculative sampling options are rejected rather than ignored.
- Speculative decoding preserves the target distribution within numerical
  precision but does not promise stable token log probabilities.

### Check deployment and dependency assumptions

- Ray is not a default dependency. Install its executor extras explicitly.
- CUDA, PyTorch, Transformers, compiler, and manylinux requirements changed
  repeatedly; use the exact wheel or container matching the deployment.
- vLLM no longer rewrites `CUDA_VISIBLE_DEVICES`; use `device_ids`.
- Treat Ray cluster traffic and the gRPC interface as private-network
  protocols. Configure authentication and TLS on supported public frontends.
- Weight loading defaults are stricter. Preserve `weights_only=True`,
  `trust_remote_code` checks, and insecure-serialization gates.

## High-value engine controls

### Compilation and graph capture

- V1 integrates `torch.compile`; `-O0` through `-O3` select
  startup/performance tradeoffs.
- Use `compile_ranges` for selective compilation. Do not use the removed
  `-O.xx` spelling.
- `FULL_AND_PIECEWISE` is the CUDA graph default. Model Runner V2 supports
  broader piecewise, mixed, and full-graph workloads.
- Use `head_dtype` when an accuracy-sensitive generation head must remain
  FP32.

### Scheduling and sizing

- Async scheduling is supported with pipeline parallelism in current
  configurations, but historical releases had preemption corruption and
  excluded several backends. Keep explicit opt-out available during upgrades.
- `--max-model-len auto` fits context length to available GPU memory.
- `--performance-mode` provides balanced, interactivity, and throughput
  presets; explicit values still win.
- Hardware, offline/server use, parallelism, model context, and multimodal
  shape all affect automatic sequence and batched-token limits.
- Use startup KV-cache capacity and maximum-concurrency logs as deployment
  sizing signals.

### Nested configuration

- Dataclass options accept whole JSON objects or dotted keys such as
  `--attention-config.flash_attn_version=2`.
- JSON config values accept decimal `k/m/g/t` and binary `K/M/G/T` size
  suffixes where the option supports human-readable integers.
- Speculation shorthand flags and a JSON speculative configuration may not
  set the same field.
- Nested YAML configuration is supported.

## High-value serving behavior

### OpenAI-style compatibility

- The request `model` field may be omitted.
- Completions `suffix` and Chat `image_url.detail` are unsupported.
- Chat `user` is accepted but ignored.
- `parallel_tool_calls=false` limits output to one tool call; `true` permits
  but does not force parallel calls.
- Strict tool calling depends on a supported parser.
- Invalid token IDs, non-finite sampling values, non-positive scheduling
  knobs, and degenerate structured-output settings are rejected.

### Responses and operational routes

- Responses support creation, retrieval, cancellation, structured output,
  reasoning events, tools, MCP integration, token IDs, and request controls.
- Health returns HTTP 503 when the engine is dead.
- Use `/load`, `/is_sleeping`, `/server_info`, tokenizer information, cache
  reset, pause/resume, and weight-update routes according to the active
  frontend.
- Endpoint plugins can extend serving, and the RLHF development router
  exposes request abortion.

### Speech, embeddings, and scoring

- Speech paths include streaming and batch transcription, translation,
  timestamps, language detection, beam search, and realtime WebSockets.
- Embeddings support truncation, multiple media items, byte-only encoding,
  sparse and late-interaction representations, and chat-shaped inputs on
  supported endpoints.
- `/score` accepts pair fields or query/document collections and supports
  multimodal and reranking use cases.

## High-value distributed and cache guidance

### Pick parallelism for the topology

- Prefer pipeline parallelism when the GPU count does not evenly divide the
  model or when GPUs lack a fast peer interconnect.
- Native multi-node launch requires the multiprocessing data-parallel
  backend and a node count that evenly divides the `DP × PP × TP` world size.
- External data-parallel balancing is MoE-only. Fault tolerance requires an
  external balancer or explicit data-parallel rank.
- Sequence, decode-context, pipeline, tensor, data, and expert parallelism
  have distinct compatibility constraints; verify the workload-specific
  combination.

### Secure and size cluster plumbing

- Give each containerized Ray node a private `VLLM_HOST_IP`; cluster traffic
  is unencrypted and unsafe on an untrusted network.
- Multiprocessing workers use `--headless` and the same head address.
- For GPUDirect RDMA, provide locked memory and a sufficiently large shared
  memory mount, then verify `NET/IB/GDRDMA` rather than socket transport.
- Integer `device_ids` index an existing visibility mask. UUID and integer
  forms cannot be mixed, and Ray ignores this option.

### Configure cache failure and offload policy deliberately

- KV connector load failures fail requests by default; set recomputation
  explicitly if that is desired.
- CPU, filesystem, disk, and object-store tiers have connector-specific
  policy, identity, chunking, reset, and request hooks.
- Prefix cache salting isolates tenants. Use reproducible SHA-256 plus CBOR
  hashes only when cross-process stability is required.
- Multimodal processing, encoder embeddings, and token prefixes use separate
  reuse paths and memory controls.

## High-value quantization and speculation guidance

### Validate hardware before selecting a format

- Quantization support is method-, architecture-, and backend-specific.
  Check the hardware table before selecting AWQ, GPTQ, Marlin,
  compressed-tensors, FP8, FP4, or GGUF.
- TurboQuant KV cache currently forces FlashAttention 2.
- Pre-SM100 FP4 can fall back to Marlin; unsupported combinations must not be
  inferred from a nearby format.
- Qwen3.5 with an FP8 KV cache has a B200 accuracy caveat.

### Use the correct speculative configuration

- Use `draft_tensor_parallel_size` inside `speculative_config`.
- Use `method="mtp"` for Gemma 4 assistant checkpoints.
- Heterogeneous vocabulary mapping is draft-model-only and greedy-only.
- Custom proposer classes are named in the speculative `model` field.
- Suffix decoding is draft-model-free; n-gram lookup copies a supplied bound
  to the omitted minimum or maximum.
- Rejection sampling modes are `strict`, `probabilistic`, and `synthetic`;
  validate synthetic acceptance rates.

## Before finishing a change

1. Confirm the installed vLLM version and frontend.
2. Check removed and deprecated fields in
   [operations and migrations](references/operations-and-migrations.md).
3. Check engine, scheduling, sampling, and model-runner compatibility.
4. Validate parallelism, cache, connector, and network assumptions.
5. Validate quantization against exact hardware and checkpoint format.
6. Make request defaults explicit and test invalid-input behavior.
7. Exercise health, cancellation, shutdown, and cache-reset paths.
8. Re-run representative outputs without speculation when diagnosing
   probability or reproducibility differences.
