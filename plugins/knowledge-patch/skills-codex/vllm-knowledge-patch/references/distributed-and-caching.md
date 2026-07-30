# Distributed Execution and Caching

## Offline and executor evolution

- In `0.7-0.10`, offline inference added `torchrun` and SPMD execution;
  `AsyncLLM` gained a Ray executor; `LLM` became `torchrun` compatible; and
  multiprocess/`torchrun` pipeline parallelism plus pipeline-combined sequence
  parallelism arrived.
- RayExecutorV2 became the default in 0.21.
- Version 0.25 permits sequence parallelism without data parallelism. Version
  0.26 adds hybrid-attention decode-context parallelism and NIXL
  pipeline-parallel prefill in push mode.

## Choosing and launching parallelism

- Use pipeline parallelism with `tensor_parallel_size=1` when the GPU count
  does not divide the model evenly, or when GPUs such as L40S lack NVLink and
  tensor-parallel communication would dominate. Set `pipeline_parallel_size`
  to the GPU count:

```bash
vllm serve MODEL --tensor-parallel-size 1 --pipeline-parallel-size 4
```

- `--nnodes/-n` requires the multiprocessing DP backend. Node count must
  evenly divide `DP × PP × TP`; `--node-rank/-r` is zero-based. The engine
  derives local DP size/rank and may use the rank for external balancing.
- External DP is rejected for non-MoE models, requires a rank, and forces
  local DP size to one. A rank can be explicit or inferred.
  `--data-parallel-multi-port-external-lb` instead starts a node-local
  supervisor with one API server per local rank and aggregate health.
- `--enable-fault-tolerance` requires external DP balancing or an explicit DP
  rank; internal balancing is unsupported. A dictionary-valued
  `fault_tolerance_config` passed to `EngineArgs` enables it automatically.
- Version 0.22 added a DP supervisor and forwards the selected rank in
  `X-data-parallel-rank`.
- Version 0.23 added node-targeted Ray placement groups, per-GPU-worker RDMA
  NIC selection, a 120-second coordinator startup timeout, and supervisor TLS.

## Ray and multiprocessing network safety

- Install the Ray executor with:

```bash
pip install "ray[cgraph]"
```

- For `examples/ray_serving/run_cluster.sh`, assign every head/worker container
  its own private `VLLM_HOST_IP`, keep each launcher shell open, and verify
  `ray status` plus `ray list nodes`.
- Ray cluster traffic is unencrypted and its payload format can allow
  arbitrary code execution. Never expose cluster addresses to untrusted
  hosts.
- For native multiprocessing, rank 0 runs the server and workers join the same
  head address with `--headless`:

```bash
vllm serve /path/to/model \
  --tensor-parallel-size 8 --pipeline-parallel-size 2 \
  --nnodes 2 --node-rank 1 \
  --master-addr HEAD_NODE_IP --headless
```

## RDMA and capacity checks

- `GPU KV cache size` in startup logs is total GPU KV token capacity.
  `Maximum concurrency for N tokens per request` estimates simultaneous
  requests at `ModelConfig.max_model_len`; add GPUs/nodes if these miss the
  deployment target.
- Docker should provide host IPC and `/dev/shm`; Kubernetes should add
  `IPC_LOCK` and mount a memory-backed `emptyDir` at `/dev/shm`.
- With `NCCL_DEBUG=TRACE`, `[send] via NET/IB/GDRDMA` confirms GPUDirect RDMA;
  `[send] via NET/Socket` shows TCP transport.

```bash
docker run --gpus all --ipc=host --shm-size=16G \
  -v /dev/shm:/dev/shm vllm/vllm-openai
```

## Data and expert parallelism

- In `0.7-0.10`, DeepSeek MoE gained `--enable-expert-parallel`, EP/TP MoE
  with DP attention, and DP communication. Elastic EP added GPU-count changes
  while retaining state; `MOE_DP_CHUNK_SIZE` became configurable.
- In `0.15-0.18`, elastic EP reached dynamic serving, NIXL-EP integration,
  and `--enable-ep-weight-filter` for skipping irrelevant expert weights.
- MRV2 later gained EPLB. Async EPLB became the default in 0.23; 0.24 rejects
  NCCL-based EPLB combined with async EPLB and adds DeepEP v2.

## Disaggregated serving and KV transfer

- In `0.7-0.10`, LMCache supported KV offload, disaggregated and chunked
  prefill; NIXL and multiple simultaneous KV connectors followed.
- In `0.11-0.14`, layouts added preparatory Prefill Context Parallelism,
  cross-layer KV blocks, Mooncake Transfer Engine, external launch, XBO,
  asymmetric TP, heterogeneous NIXL, cross-layer MultiConnector, and LMCache
  cache registration.
- In `0.19-0.22`, a 3FS connector, heterogeneous-TP prefill/decode for
  Mamba2-like models, and bidirectional prefill/decode transfers arrived.
- Version 0.23 enabled HMA by default for capable connectors. Later changes
  include NIXL pipeline-parallel prefill in push mode.

## Prefix and multimodal caching

- V1 processes multimodal inputs outside the engine loop and caches
  preprocessed inputs across requests. Image hashes participate in prefix
  lookup.
- A separate encoder cache stores vision embeddings, letting the scheduler
  split accompanying text prefill across steps.
- Prefix-cache salting arrived in `0.7-0.10` for tenant isolation.
  SHA-256 plus CBOR can make hashes reproducible.
- Mamba/hybrid models can use block-aligned state caching with
  `--enable-prefix-caching --mamba-cache-mode align`; speculative decoding
  gained compatibility in `0.15-0.18`.
- Processor caching has `--mm-processor-cache-type` and a shared-memory object
  size limit. The encoder independently controls tensor parallelism, attention
  backend/dtype, FP8 scale load/save and margin, plus tensor-IPC mode and GPU
  memory.

## Offloading policy and failure semantics

- Version 0.11 added CPU KV offload with LRU management.
- In `0.15-0.18`, CPU stores can be restricted to frequently reused blocks,
  FlexKV became a backend, and one specification can contain multiple KV
  groups.
- Version 0.17 changed connector load failure from `recompute` to `fail`.
  Configure recomputation explicitly if request failure is not desired.
- In `0.19-0.22`, CPU offload gained a pluggable `CachePolicy`, hybrid-model
  support, Hybrid Memory Allocator integration, `MooncakeStoreConnector`,
  filesystem and Mooncake disk secondary tiers, and `reset_cache`.
- In `0.23-0.26`, storage expanded to object-store tiers, async batched
  lookup, parallelism-independent filesystem storage, workload identity,
  heterogeneous-group `blocks_per_chunk`, and encoder-cache connectors with
  CPU offload.
- Per-request `on_new_request` hooks can select offloading policy.
