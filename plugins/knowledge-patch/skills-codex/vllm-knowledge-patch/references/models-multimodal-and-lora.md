# Models, Multimodal Processing, and LoRA

## Multimodal integration contract

- In `0.7-0.10`, a model implementing the merged multimodal processor plus the
  appropriate `get_*_embeddings` methods became automatically supported by
  V1. The legacy out-of-tree input mapper was deprecated.
- VLMs can use the Transformers backend, including multimodal caching.
- Request inputs include image embeddings, multiple image/audio items, and
  long-audio chunking. Processor settings can be passed through
  `mm_processor_kwargs`.
- V1 keeps reusable preprocessed inputs, prefix hashes, and encoder embeddings
  in three distinct cache paths. See
  [distributed execution and caching](distributed-and-caching.md).
- Prompt embeddings, audio embeddings, and incremental `StreamingInput`
  provide non-token or session-oriented integration paths.

## Pooling, embeddings, and tasks

- Version 0.10 allows a model to advertise several tasks and poolers, with
  pooling parameters selected at runtime instead of one fixed task/pooler.
- Versions 0.11-0.14 added Transformers encoder-only models, BERT token
  classification/NER, multimodal pooling, RADIO, Qwen3 Omni audio-in-video,
  and Qwen3-VL embedding/reranking.
- Versions 0.15-0.18 added BGE-M3 sparse, ColBERT, ColPali retrieval, sparse
  IO, multimodal late interaction, Cohere Embed v2, and ERNIE pooling.
- Versions 0.23-0.26 added RoBERTa and XLM-RoBERTa token classification.
- Diffusion language models became a workload in 0.24 with DiffusionGemma,
  CPU execution, and structured-output guardrails. Version 0.25 added tensor
  parallelism and Hugging Face stability-window semantics.

## LoRA progression

- In `0.7-0.10`, Rank-Stabilized LoRA, V1 LoRA, `TransformersModel` LoRA, and
  a default local-directory resolver arrived. Multi-LoRA expanded to x86,
  TPU, and Neuron; Tensorizer loading supports V1 and LoRA.
- In `0.11-0.14`, `--fully-sharded-loras` added fused-MoE sharding. LoRA
  reached the towers/connectors of LLaVA, PaliGemma, DotsOCR, and GLM4-V, plus
  DeepSeek-OCR, Qwen3-Next, PLaMo 2/3, vision-processor caching, and MoE expert
  `base_layer` loading.
- In `0.15-0.18`, a Hugging Face Hub resolver arrived, directly loaded
  quantized adapters such as QLoRA became valid, and Whisper plus FP8 dense
  LoRA paths were added.
- In `0.19-0.22`, `--lora-target-modules` can restrict adapted modules.
  Adapters expanded to H2OVL towers/connectors, Qwen3-ASR, Gemma 4,
  DeepSeek V3.2, XPU, expert parallelism, and simultaneous 2D/3D MoE.
- In `0.23-0.26`, MiniCPM-V 4.6 gained language-backbone LoRA; BF16 models
  gained FlashInfer MoE LoRA; LLaVA-Next-Video gained tower/connector LoRA.
- `head_dtype` applies through LoRA when the generation head must stay FP32.

## Model families added in early V1

- Versions 0.7-0.8 added CogAgent, DeepSeek-VL2, InternLM3, Whisper, Qwen2
  PRM, InternLM2 reward models, Gemma 3, Mistral Small 3.1, Phi-4 multimodal,
  Grok1, QwQ-32B tool calling, and Zamba2.
- Versions 0.9-0.10 added MiMo-7B, MiniMax-VL-01, Ovis 2, Falcon-H1,
  LlamaGuard 4, Llama 4 with EAGLE, EXAONE 4.0, Hunyuan V1, JinaVL Reranker,
  Arcee, Voxtral, more embedding families, attention-free architectures, and
  hybrid SSM/attention models.
- Gemma 3 on 0.8 requires Transformers main and should use `bfloat16` or
  `float32`; `float16` is numerically unstable. Falcon-H1 on 0.9 also needs a
  development Transformers install.

## Model families added with later V1 capabilities

- Versions 0.11-0.12 added DeepSeek-V3.2-Exp, Qwen3-VL, OLMo3, Dots OCR, CWM,
  PLaMo-3, OpenCUA-7B, Mistral Large 3, and Ministral 3.
- Versions 0.13-0.14 added autoregressive-only BAGEL, AudioFlamingo3, latent
  MoE, Grok-2, LFM2-VL, MiMo-V2-Flash, openPangu MoE, IQuestCoder,
  Nemotron Parse 1.1, GLM-ASR, Isaac vision, Kanana 1.5, and K-EXAONE.
- Versions 0.15-0.16 added Kimi-K2.5, Molmo2, Step1, Eagle2.5-8B VLM,
  GLM-OCR, Qwen3-ASR, Intern-S1-Pro, openPangu7B-VL, MusicFlamingo, and GLM-5.
- Versions 0.17-0.18 added Qwen3.5, Ring 2.5, Ovis 2.6, FunASR, FireRedASR2,
  Sarvam MoE, OLMo Hybrid, HyperCLOVAX-SEED-Think-14B, ColPali, and ERNIE
  pooling models.
- Versions 0.19-0.20 added Cohere ASR, ColQwen3.5, Granite 4 Speech/4.1
  Vision, Qwen3-ForcedAligner, DeepSeek V4, Hunyuan v3 preview, EXAONE-4.5,
  Phi-4 Reasoning Vision, TeleChat3, Jina Reranker v3, and Nemotron-v3 VL.
- Versions 0.21-0.22 added MiMo-V2.5, Laguna XS.2, Moondream3, Cohere MoE and
  Eagle, MiniCPM-V 4.6, and InternS2 Preview. DeepSeek V4 gained ROCm,
  pipeline/disaggregated serving, MTP speculation, and NVFP4 MoE.
- Gemma 4 support includes MoE, multimodal, reasoning, and tools but requires
  `transformers>=5.5.0`; the `vllm/vllm-openai:gemma4` image is the
  preconfigured installation path.

## Latest model families and retirement

- Version 0.23 added Step-3.7-Flash, Cosmos3 Reasoner, Mellum v2, Cohere Mini
  Code, and encoder-free Gemma 4 Unified. MiniMax-M3 was explicitly
  unsupported there.
- Version 0.24 added MiniMax-M3, DiffusionGemma, HrmText, and OpenMOSS.
- Versions 0.25-0.26 added LLaVA-OneVision-2, Unlimited OCR,
  MOSS-Transcribe-Diarize, Hy3, Inkling, Cosmos3 Edge Reasoner, and
  TranslateGemma.
- Version 0.23 deprecated `JAISLMHeadModel`; 0.24 removed ERNIE, Xverse,
  Bamba, and the InternLM alias while deprecating first-generation Qwen and
  QwenVL.
- Version 0.25 removed Baichuan, Aquila, Tarsier, Tarsier2, and Mantis.
  Version 0.26 removed TeleChat, Persimmon, and Fuyu.
