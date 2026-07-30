# Operations and Migrations

## CLI, configuration, and observability

- In `0.7-0.10`, `vllm bench` consolidated benchmark commands and
  `--dataset` in `benchmark_serving.py` was deprecated. Paged help is
  available as `--help=page`; the CLI default model changed to Qwen3-0.6B.
- Metrics deprecated in that batch include
  `vllm:time_in_queue_requests`, `vllm:model_forward_time_milliseconds`, and
  `vllm:model_execute_time_milliseconds`.
  `--show-hidden-metrics-for-version` exposed transition metrics.
- Later additions included `vllm:cache_config_info`, KV-event publishing, and
  in-memory Prometheus metric access; the `gpu_` prefix was deprecated for
  metrics not specific to GPUs.
- Version 0.15 removed `vllm:time_per_output_token_seconds`; use
  `vllm:inter_token_latency_seconds`.
- Environment and timeout controls in `0.19-0.22` include
  `VLLM_MAX_N_SEQUENCES`, opt-in `VLLM_MEDIA_CACHE`,
  `VLLM_SKIP_MODEL_NAME_VALIDATION`, and
  `--cpu-distributed-timeout-seconds`.
- `tcmalloc` became the CPU default in 0.19.

## Removed engine, request, and extension interfaces

- Version 0.12 removed `num_lookahead_slots`, `best_of`, and LoRA extra
  vocabulary; `xformers` was deprecated.
- Version 0.13 removed deprecated plugin, compilation, task, seed, and
  multimodal settings; `embed_input_ids`/`embed_multimodal` fallbacks; and the
  tokenizer setter. Replace `--convert reward` with `--convert embed`.
- Version 0.14 deprecated `seed_everything`.
- Direct-child EPLB fields on `ParallelConfig`, `guided_*`,
  `override_pooler_config`, `disable_log_requests`,
  `CompilationConfig.use_inductor`, and the old metrics were scheduled for
  removal beginning in 0.12.
- In `0.15-0.18`, removals included DeepSpeedFp8, RTN, BitBlas, Marlin 24,
  `reasoning_content`, `VLLM_ALL2ALL_BACKEND`, per-request logits processors,
  and `swap_space`.
- Version 0.19 deprecated `--calculate-kv-scales`, the `score` task, virtual
  engines, and `--disable-frontend-multiprocessing`; it removed
  per-tensor/per-channel FP8 and Sparse24 integration.
- Version 0.22 removed old imports for `get_tokenizer` and
  `resolve_hf_chat_template`, plus deprecated MLA-prefill arguments.
  Replace backend environment variables with `--moe-backend` and
  `--linear-backend`.
- Version 0.23 began deprecating the NIXL `kv_both` role. Version 0.24 removed
  `P2pNcclConnector`.
- Version 0.25 dropped `gptq_marlin` on ROCm, moved legacy `api_server.py` to
  examples, and deprecated the old online FP8 MoE class.

## Wheel and dependency transitions

- In `0.7-0.10`, wheels moved from PyTorch 2.6/CUDA 12.4 to PyTorch 2.7 with
  CUDA 12.8 by default; CUDA 12.4 support ended and a CUDA 12.6 artifact was
  offered. Version 0.10 used PyTorch 2.7.1. The CUDA 12.8 flow accepts
  `--torch-backend=auto`.
- In `0.11-0.14`, CPU builds moved to PyTorch 2.8 and ROCm to 7.0; then the
  baseline became PyTorch 2.9.0/CUDA 12.9/Transformers 4.57.3 and finally
  PyTorch 2.9.1 with `cu129` default wheels.
- In `0.15-0.18`, the baseline moved to PyTorch 2.10.0; use the updated wheel
  that fixed CUDA 12.9+ library mismatches. XPU moved from IPEX to
  `vllm-xpu-kernels`, ROCm renamed `aiter` to `amd-aiter`, and Ray left the
  default dependencies.
- Version 0.20 switched default PyPI wheels and the compatible server image to
  CUDA 13.0, upgraded CUDA/XPU builds to PyTorch 2.11, added Python 3.14, and
  moved to `transformers>=5`. CUDA 12.9 users were directed to `uv` with
  `--torch-backend=cu129`.
- Version 0.21 deprecated Transformers v4 and required a C++20 compiler.
- CUDA 13.0 wheels in 0.21 and CUDA 12.9 wheels in 0.22 use
  `manylinux_2_28`. Version 0.22 added a non-root `vllm-openai` image target
  and optional Python-only installation.
- Version 0.24 moved ROCm to PyTorch 2.11, XPU to torch-xpu 2.12, CUDA
  containers to GCC 12, and made `mistral_common` optional. Starlette must be
  at least 1.0.1 for its security fix.
- Version 0.26 moved the Transformers integration to 5.13.0.

## Security-sensitive behavior

- In `0.7-0.10`, `VLLM_ALLOW_INSECURE_SERIALIZATION` and a V1 switch to
  disable pickle fallback made serialization trust explicit.
- Version 0.11 addressed GHSA-wr9h-g72x-mwhm; 0.13 added protection for
  CVE-2025-62164.
- Version 0.14 prevents token leakage in crash logs and loads PyTorch weights
  with `weights_only=True`.
- Version 0.18 makes NemotronVL and KimiK25 honor `trust_remote_code` and
  places RLHF weight-sync deserialization behind the insecure-serialization
  setting.
- Private Ray cluster traffic is unencrypted and may deserialize
  code-executing payloads. Keep it on trusted private addresses.
- The gRPC entrypoint is documented for private use even where TLS/mTLS
  support exists.
- Invalid image URLs, token IDs, numeric values, and structured-output
  configurations now fail validation rather than reaching unsafe or undefined
  paths.

## Upgrade checklist

1. Match the wheel or container to CUDA/ROCm/XPU, PyTorch, Python, compiler,
   libc, and Transformers requirements.
2. Install optional Ray or Python components explicitly.
3. Replace V0 engine imports and removed request/configuration fields.
4. Replace deprecated backend environment variables and import paths.
5. Reconcile sampling values inherited from `generation_config`.
6. Verify model-specific dependencies, `trust_remote_code`, and weight-loading
   policy.
7. Compare old dashboards against renamed or removed metrics.
8. Exercise health, cache reset, pause/resume, cancellation, shutdown, and
   weight-update routes under failure.
9. Validate stricter request errors and private-network assumptions.
