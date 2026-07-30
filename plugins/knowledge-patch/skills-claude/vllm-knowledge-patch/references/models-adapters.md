# Models, Multimodal Integration, Adapters, and Lifecycle

## Custom model integration

### V1 forward contract (`0.7-0.10`)

Custom model `forward` methods no longer receive `kv_cache` or
`attn_metadata`; attention implementations obtain both through
`forward_context`. During the V0 transition, `SupportsV0Only` allowed a model
definition to declare its dependency on the old engine. V0 is now removed, so
new integrations must implement the V1 contract.

### Multimodal extension path (`0.7-0.10`)

A model that implements the merged multimodal processor and appropriate
`get_*_embeddings` methods is automatically supported by V1. The legacy input
mapper for out-of-tree multimodal models was deprecated in 0.8. By 0.10, VLMs
can run through the Transformers backend and that backend supports multimodal
caching.

V1 preprocesses multimodal data outside the engine loop, shares cached
preprocessed inputs across requests, includes image hashes in prefix caching,
and keeps vision embeddings in an encoder cache
(`v1-architecture-and-batching`).

### Multimodal encoder controls (`engine-and-openai-server`)

Processor caching exposes `--mm-processor-cache-type` plus a shared-memory
object-size cap. Encoder execution has separate tensor-parallel, attention
backend, and attention dtype settings. Encoder FP8 scales can be loaded or
saved with a configurable save margin. Multimodal tensor IPC has its own mode
and GPU-memory budget.

## Pooling, embedding, and task selection

Version 0.10 lets a model advertise multiple tasks and multiple poolers and
select pooling parameters at runtime. Integrations must not assume a single
fixed task/pooler (`0.7-0.10`).

Versions 0.11-0.12 add RADIO and Transformers encoder-only models, BERT token
classification/NER, multimodal pooling, and Qwen3 Omni audio-in-video. Versions
0.13-0.14 add Qwen3-VL embedding and reranking (`0.11-0.14`).

Versions 0.15-0.18 add BGE-M3 sparse and ColBERT embeddings, sparse-embedding
IO, multimodal late-interaction scoring, Cohere Embed v2 compatibility, ColPali
retrieval, and ERNIE pooling models.

Versions 0.25-0.26 add RoBERTa/XLM-RoBERTa token classification
(`0.23-0.26`).

## Runtime lifecycle and weight synchronization

Version 0.7 added `LLM.sleep`, `LLM.wake_up`, `LLM.collective_rpc`, and
`LLM.reset_prefix_cache` for post-training integrations. Version 0.10 extends
RPC with runtime weight reload and configuration updates and adds a logprobs
mode selecting the processing stage that supplies returned logprobs
(`0.7-0.10`).

Version 0.11 adds `LLM.apply_model`; 0.12 adds pause/resume generation for
asynchronous reinforcement-learning training (`0.11-0.14`).

Version 0.16 adds native NCCL weight synchronization, layerwise reload, and
pause/resume that preserves requests. Version 0.17 adds an IPC synchronization
path and sleep level 0 with an enqueue/wait pattern (`0.15-0.18`). Version 0.21
adds `/start_weight_update` and `/finish_weight_update`; version 0.26 adds
runtime draft-weight updates and an `/abort_requests` development endpoint.

Deserialization for reinforcement-learning weight synchronization is gated by
the insecure-serialization setting from 0.18.

## LoRA support

### `0.7-0.10`

Rank-Stabilized LoRA arrived in 0.7. Version 0.8 added V1 LoRA and LoRA for
`TransformersModel`; Multi-LoRA expanded to x86, TPU, and Neuron. Version 0.9
adds a default local-directory resolver and Tensorizer loading for V1 and LoRA.

### `0.11-0.14`

Version 0.12 adds `--fully-sharded-loras` for fused MoE. Version 0.14 supports
LoRA on multimodal towers/connectors for LLaVA, PaliGemma, DotsOCR, and GLM4-V;
adds DeepSeek-OCR, Qwen3-Next, and PLaMo 2/3 paths; caches vision-LoRA processor
work; and loads MoE expert `base_layer` weights.

### `0.15-0.18`

Version 0.16 adds a Hugging Face Hub LoRA resolver. Version 0.17 permits direct
loading of quantized adapters such as QLoRA. Version 0.18 adds Whisper LoRA and
an FP8 dense LoRA kernel.

### `0.19-0.22`

Version 0.19 adds `--lora-target-modules` and H2OVL tower/connector LoRA.
Subsequent releases add adapters for Qwen3-ASR, Gemma 4, DeepSeek V3.2, XPU,
and expert parallelism. Version 0.22 adds simultaneous 2D and 3D MoE LoRA
adapters.

### `0.23-0.26`

Version 0.25 adds language-backbone LoRA for MiniCPM-V 4.6. Version 0.26 adds
FlashInfer MoE LoRA for BF16 models and tower/connector LoRA for
LLaVA-Next-Video. `head_dtype` also applies through the LoRA path.

## Model loading and installation caveats

Gemma 3 on 0.8 requires Transformers from its main branch and should use
`bfloat16` or `float32`; `float16` is numerically unstable. Falcon-H1 on 0.9
also requires a development Transformers installation (`0.7-0.10`).

Version 0.12 accepts `repo_id:quant_type` for GGUF selection, automatically
detects Mistral format, and loads multimodal Gemma 3 GGUF (`0.11-0.14`).

Gemma 4 support in 0.19 includes MoE, multimodal, reasoning, and tools but
requires `transformers>=5.5.0`. The recommended ready-to-run image is
`vllm/vllm-openai:gemma4` (`0.19-0.22`). A Gemma 4 assistant checkpoint used
for speculative decoding must take the MTP path rather than generic draft-model
configuration.

## Model and workload coverage

These lists are compatibility data: confirm that the installed version includes
the named implementation before relying on it.

### `0.7-0.10`

Versions 0.7-0.8 add CogAgent, DeepSeek-VL2, InternLM3, Whisper, Qwen2 PRM,
InternLM2 reward models, Gemma 3, Mistral Small 3.1, Phi-4 multimodal, Grok1,
QwQ-32B tool calling, and Zamba2.

Versions 0.9-0.10 add MiMo-7B, MiniMax-VL-01, Ovis 2, Falcon-H1, LlamaGuard 4,
Llama 4 with EAGLE, EXAONE 4.0, Hunyuan V1, JinaVL Reranker, Arcee, Voxtral,
additional embedding families, attention-free architectures, and hybrid
SSM/attention models.

### `0.11-0.14`

Versions 0.11-0.12 add DeepSeek-V3.2-Exp, Qwen3-VL, OLMo3, Dots OCR, CWM,
PLaMo-3, OpenCUA-7B, Mistral Large 3, Ministral 3, RADIO, Transformers
encoder-only models, BERT token classification/NER, multimodal pooling, and
Qwen3 Omni audio-in-video.

Versions 0.13-0.14 add BAGEL autoregressive support, AudioFlamingo3, latent
MoE, Grok-2, LFM2-VL, MiMo-V2-Flash, openPangu MoE, IQuestCoder, Nemotron Parse
1.1, GLM-ASR, Isaac vision, Kanana 1.5, and K-EXAONE, plus Qwen3-VL embedding
and reranking.

### `0.15-0.18`

Versions 0.15-0.16 add Kimi-K2.5, Molmo2, Step1, Eagle2.5-8B VLM, GLM-OCR,
Qwen3-ASR, Intern-S1-Pro, openPangu7B-VL, MusicFlamingo, and GLM-5.

Versions 0.17-0.18 add Qwen3.5, Ring 2.5, Ovis 2.6, FunASR, FireRedASR2,
Sarvam MoE, OLMo Hybrid, HyperCLOVAX-SEED-Think-14B, ColPali retrieval, and
ERNIE pooling models.

### `0.19-0.22`

Versions 0.19-0.20 add Cohere ASR, ColQwen3.5, Granite 4 Speech and 4.1 Vision,
Qwen3-ForcedAligner, DeepSeek V4, Hunyuan v3 preview, EXAONE-4.5, Phi-4
Reasoning Vision, TeleChat3, Jina Reranker v3, and Nemotron-v3 VL.

Versions 0.21-0.22 add MiMo-V2.5, Laguna XS.2, Moondream3, Cohere MoE and
Eagle, MiniCPM-V 4.6, and InternS2 Preview. DeepSeek V4 also gains ROCm,
pipeline/disaggregated serving, MTP speculation, and NVFP4 MoE support.

### `0.23-0.26`

Version 0.23 adds Step-3.7-Flash, Cosmos3 Reasoner, Mellum v2, Cohere Mini Code,
and encoder-free Gemma 4 Unified. MiniMax-M3 is explicitly unsupported in 0.23
and arrives in 0.24 with DiffusionGemma, HrmText, and OpenMOSS.

Versions 0.25-0.26 add LLaVA-OneVision-2, Unlimited OCR,
MOSS-Transcribe-Diarize, Hy3, Inkling, RoBERTa/XLM-RoBERTa token
classification, Cosmos3 Edge Reasoner, and TranslateGemma.

DiffusionGemma in 0.24 supports CPU execution and structured-output guardrails
for diffusion decoders. Version 0.25 adds tensor parallelism and Hugging Face
stability-window semantics.

## Retired model integrations (`0.23-0.26`)

Version 0.23 deprecates `JAISLMHeadModel`. Version 0.24 removes ERNIE, Xverse,
Bamba, and the InternLM registry alias and deprecates first-generation
Qwen/QwenVL. Version 0.25 removes Baichuan, Aquila, Tarsier, Tarsier2, and
Mantis and moves legacy `api_server.py` to examples. Version 0.26 removes
TeleChat, Persimmon, and Fuyu.
