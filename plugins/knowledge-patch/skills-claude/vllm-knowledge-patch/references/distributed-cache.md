# Distributed Execution, Parallelism, and Cache Topology

## Offline and multi-process execution

Offline inference added `torchrun` and SPMD execution in 0.7, and `AsyncLLM`
gained a Ray executor. The `LLM` API became compatible with a `torchrun`
launcher in 0.8. Version 0.9 adds multiprocess and `torchrun` pipeline
parallelism plus sequence parallelism combined with pipeline parallelism
(`0.7-0.10`).

## Selecting pipeline versus tensor parallelism

When a model fits within one node but its GPU count does not divide the model
cleanly, set `tensor_parallel_size=1` and set `pipeline_parallel_size` to the
GPU count. Pipeline parallelism is also preferable to tensor parallelism on
GPUs without NVLink, such as L40S, because it avoids heavier cross-GPU
communication (`distributed-parallelism`).

```bash
vllm serve MODEL --tensor-parallel-size 1 --pipeline-parallel-size 4
```

Startup logs expose two useful capacity estimates. `GPU KV cache size` is the
total token capacity of the GPU KV cache. `Maximum concurrency for N tokens per
request` estimates simultaneous requests at `ModelConfig.max_model_len`. Add
GPUs or nodes when these estimates are below the deployment target.

## Native multi-node multiprocessing

`--nnodes/-n` requires the multiprocessing data-parallel backend. The node
count must evenly divide the `DP × PP × TP` world size, and `--node-rank/-r` is
zero-based. Local data-parallel size and rank are derived from those values; the
inferred rank can feed external load balancing (`engine-and-openai-server`).

Rank 0 runs the normal server. Each worker connects to the same head address
with `--headless` (`distributed-parallelism`):

```bash
vllm serve /path/to/model \
  --tensor-parallel-size 8 --pipeline-parallel-size 2 \
  --nnodes 2 --node-rank 1 \
  --master-addr HEAD_NODE_IP --headless
```

## Device placement

From 0.24, vLLM no longer sets `CUDA_VISIBLE_DEVICES` internally; use
`device_ids`. ROCm also begins deprecating `CUDA_VISIBLE_DEVICES`
(`0.23-0.26`). When a visibility environment variable already exists, integer
`--device-ids` index the visible list rather than raw physical devices. UUIDs
are accepted, but integers and UUIDs cannot be mixed, duplicates are rejected,
and the option does not affect Ray executors (`engine-and-openai-server`).

## Data and expert parallelism

Version 0.8 adds expert parallelism for DeepSeek with
`--enable-expert-parallel`, EP/TP MoE with data-parallel attention, and
data-parallel communication. Version 0.10 adds elastic expert parallelism that
can change GPU counts while retaining state, plus the `MOE_DP_CHUNK_SIZE`
environment variable (`0.7-0.10`).

Dynamic expert scaling arrived in 0.17; 0.18 adds NIXL-EP and
`--enable-ep-weight-filter` to avoid loading irrelevant expert weights
(`0.15-0.18`).

Async EPLB becomes the default in 0.23. Version 0.24 rejects NCCL-based EPLB
combined with async EPLB and adds DeepEP v2. Version 0.25 permits sequence
parallelism without data parallelism. Version 0.26 adds hybrid-attention DCP
and NIXL pipeline-parallel prefill in push mode (`0.23-0.26`).

### External balancing and fault tolerance

External data parallelism is MoE-only. It requires a rank and forces local
data-parallel size to one; the rank can be explicit or inferred from native
multi-node launch. `--data-parallel-multi-port-external-lb` instead runs a
node-local supervisor with one API server per local rank and aggregated health.

`--enable-fault-tolerance` requires external data-parallel balancing or an
explicit data-parallel rank; internal balancing is unsupported. A
dictionary-valued `fault_tolerance_config` passed to `EngineArgs` enables fault
tolerance automatically (`engine-and-openai-server`).

Version 0.22 adds a data-parallel supervisor and forwards the selected rank in
the `X-data-parallel-rank` header (`0.19-0.22`). Version 0.23 adds specific-node
targeting for Ray placement groups, per-GPU-worker RDMA NIC selection, a
120-second coordinator startup timeout, and TLS for the data-parallel
supervisor (`0.23-0.26`).

## Ray clusters and transport security

Install the Ray executor with:

```bash
pip install "ray[cgraph]"
```

With `examples/ray_serving/run_cluster.sh`, assign a distinct `VLLM_HOST_IP` to
each head or worker container, leave every launching shell open, and verify the
cluster with `ray status` and `ray list nodes`. Use private addresses that
untrusted hosts cannot reach: Ray cluster traffic is unencrypted and its
payload format may permit arbitrary code execution (`distributed-parallelism`).

For GPUDirect RDMA in Docker, provide host IPC and `/dev/shm`. In Kubernetes,
add `IPC_LOCK` and mount a memory-backed `emptyDir` at `/dev/shm`. With
`NCCL_DEBUG=TRACE`, `[send] via NET/IB/GDRDMA` confirms GPUDirect RDMA;
`[send] via NET/Socket` indicates inefficient TCP transport.

```bash
docker run --gpus all --ipc=host --shm-size=16G \
  -v /dev/shm:/dev/shm vllm/vllm-openai
```

## Disaggregated prefill and KV transport

The LMCache connector in 0.8 supports KV-cache offload, disaggregated prefill,
and chunked prefill. Version 0.9 adds NIXL integration and multiple KV
connectors (`0.7-0.10`).

Version 0.12 adds preparatory Prefill Context Parallelism and cross-layer KV
blocks; 0.13 adds the Mooncake Transfer Engine and external-launcher mode.
Version 0.14 adds XBO, asymmetric-tensor-parallel and heterogeneous-layout
NIXL, cross-layer MultiConnector layouts, and LMCache KV-cache registration
(`0.11-0.14`).

Version 0.20 adds a 3FS connector and heterogeneous-TP prefill/decode for
Mamba2-like models. Version 0.21 adds bidirectional KV transfers between
prefill and decode (`0.19-0.22`).

Version 0.26 adds NIXL pipeline-parallel prefill in push mode
(`0.23-0.26`).

The NIXL `kv_both` role begins deprecation in 0.23, and version 0.24 removes
`P2pNcclConnector` (`0.23-0.26`). Migrate connector topology rather than
carrying either legacy route into new deployment configuration.

## Prefix and encoder caching

V1 preserves hash-based lookup and LRU eviction while making eviction
constant-time and reducing allocation overhead. Prefix caching is enabled by
default because its miss overhead is low (`v1-architecture-and-batching`).

Multimodal preprocessing happens outside the engine loop and preprocessed
inputs can be shared across requests. Image hashes participate in prefix-cache
lookup. A distinct encoder cache keeps vision embeddings, allowing the
scheduler to split text prefill across steps instead of pairing an entire image
and text prompt in one step (`v1-architecture-and-batching`).

At engine-argument resolution, however, hybrid models keep prefix caching
opt-in. Prefix caching defaults on only when the model reports support and is
not hybrid. RISC-V CPU forces prefix caching and chunked prefill off
(`engine-and-openai-server`).

Version 0.9 adds per-request prefix-cache salting. Version 0.10 allows
Completions and Responses to carry `cache_salt` and supports reproducible
SHA-256-plus-CBOR hashing (`0.7-0.10`).

Mamba/hybrid models gain block-aligned prefix caching through
`--mamba-cache-mode align`; speculative decoding supports this mode from 0.17
(`0.15-0.18`). Version 0.25 adds Mamba-hybrid prefix caching to Model Runner V2
(`0.23-0.26`).

## CPU and tiered KV offloading

### Initial policy-aware offloading (`0.11-0.14`, `0.15-0.18`)

Version 0.11 adds CPU KV-cache offload with LRU management. Version 0.18 can
restrict CPU stores to frequently reused blocks, adds FlexKV, and permits
multiple KV groups in one offloading specification.

Version 0.17 changes the KV-connector load-failure default from `recompute` to
`fail`. Configure recomputation explicitly if transparent recovery is required.

### Hybrid and disk tiers (`0.19-0.22`)

Version 0.19 introduces a pluggable CPU-offload `CachePolicy` and hybrid-model
support. Version 0.21 integrates offloading with the Hybrid Memory Allocator and
adds `MooncakeStoreConnector`. Version 0.22 adds tiers beyond CPU RAM: a Python
filesystem secondary tier, Mooncake disk offloading, and `reset_cache`.

### Object stores and per-request policy (`0.23-0.26`)

Version 0.23 adds an object-store secondary tier, enables the Hybrid Memory
Allocator by default for capable connectors, and exposes per-request offloading
policy through `on_new_request`. Later releases add async batched lookup, a
parallelism-independent filesystem tier, workload identity for object storage,
`blocks_per_chunk` for heterogeneous KV groups, and encoder-cache connectors
with CPU offloading.

## Weight offloading

Weight offloading V2 gains prefetch in 0.17, selective CPU offload, and pinned
copies that avoid doubling CPU memory (`0.15-0.18`). This is distinct from KV
offloading and should be sized independently.
