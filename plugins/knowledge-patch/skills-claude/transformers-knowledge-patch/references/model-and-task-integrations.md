# Model and task integrations

Use this catalog to select an architecture and preserve its model-specific
constraints. It is grouped by task; versions are inline compatibility markers.

## Language, reasoning, and embedding models

- **Mistral 3.1** supports vision and a 128K context (4.50.0).
- **Qwen3 and Qwen3MoE** architecture code shipped before weights in 4.51.0.
  Do not assume a checkpoint exists merely because an architecture imports.
  The DeepSeek-V3 contribution was still marked work in progress in that
  release.
- **BitNet** and **GraniteMoeHybrid** join the library in 4.52.1.
- **ModernBERT** gains a question-answering head in 4.51.0.
- **Arcee, T5Gemma, Falcon H1, dots.llm1, and SmolLM3** are added in 4.53.0.
  T5Gemma is an encoder-decoder Gemma family.
- **MiniMax-Text-01** supports inference contexts up to four million tokens in
  4.53.0. **GLM-4.1V** architecture code again precedes a public checkpoint.
- **Ernie 4.5** initially includes its dense 0.3B text model (4.54.0).
- **LFM2** provides 350M, 700M, and 1.2B models aimed at CPU/GPU/NPU edge
  deployment (4.54.0).
- **ModernBERT Decoder** is a causal architecture with sequences through 8192
  tokens (4.54.0). **DeepSeek V2, Doge, and xLSTM** are also added.
- **EXAONE 4.0** supports reasoning and non-reasoning modes, tools, and English,
  Korean, and Spanish (4.54.0).
- **GPT-OSS 20B and 120B** are MoE reasoning integrations with native MXFP4
  weights in 4.55.0; consult loading guidance for memory and kernel constraints.
- **Seed-OSS** joins the language-model set in 4.56.0.
- **Qwen3-Next** combines Gated DeltaNet and attention and includes an 80B-A3B
  checkpoint (4.57.0).
- **VaultGemma** is a 1B differentially private research checkpoint limited to
  1024 tokens (4.57.0).
- **LongCat-Flash** provides a 128K context plus reasoning and tool use
  (4.57.0).
- **FlexOlmo** exposes independently trained domain experts that can be included
  or excluded at inference (4.57.0).
- **BLT** is vocabulary-free: it tokenizes UTF-8 bytes and creates dynamic
  patches (4.57.0). **OLMO3 and Ministral** are also integrated.
- **K-EXAONE** arrives through EXAONE-MoE, and **Youtu-LLM** is a 1.96B model
  with 128K context and a `use_deterministic` option (5.1.0).
- **Moonshine** supports streaming, and **GLM-Image** accepts batch sizes above
  one as of 5.1.0.
- **GLM-5** uses the `GlmMoeDsa` architecture and DeepSeek Sparse Attention
  (5.2.0).
- **Qwen3.5 and Qwen3.5 MoE** arrive in 5.2.0. The first checkpoint is a native
  vision-language model with 397B total/17B active parameters, hybrid Gated
  Delta Networks, and sparse MoE layers.
- **EuroBERT** is a bidirectional Llama-like multilingual encoder with 8192-token
  sequences (5.3.0).
- **OLMo Hybrid** interleaves full attention with Gated DeltaNet and stores both
  KV and recurrent state in its cache (5.3.0). **Nemotron 3** and Qwen3.5
  sequence-classification support are also added.
- **Jina Embeddings v3** provides multilingual, task-specific embeddings with
  five built-in LoRA adapters and 8192-token sequences (5.4.0).
- **Mistral 4** unifies instruction/reasoning operation with text/image input
  and 256K context (5.4.0).
- **Gemma 4** offers pretrained and instruction-tuned 1B, 13B, and 27B variants
  (5.5.0); its image processing has a fixed soft-token budget.
- **NomicBERT** is a native dense embedding model with 8192-token context for
  search, clustering, and classification. Prefix inputs with task-specific
  instructions (5.5.0).
- **OpenAI Privacy Filter** performs bidirectional, on-premises PII detection and
  masking in one forward pass, returning per-token probabilities over eight
  categories and coherent decoded spans (5.6.0).
- **OLMo** gains sequence-classification heads, **Nemotron-H** gains MLP mixers,
  and Qwen Thinker base checkpoints can load without a generative head (5.6.0).
- **Laguna XS.2** is a Poolside MoE decoder whose layers can have different query
  head counts while sharing one KV-cache shape. Its sigmoid router combines
  element-wise gate scores and learned expert bias without an auxiliary
  balancing loss (5.7.0). **SonicMoe** is also supported.
- **DeepSeek-V4-Flash, DeepSeek-V4-Pro, and Base variants** replace Multi-head
  Latent Attention with hybrid local/long-range attention, residual connections
  with Manifold-Constrained Hyper-Connections, and early MoE routing with a
  static token-to-expert hash table (5.8.0).
- **Gemma 4 Assistant** is a small text model for Multi-Token Prediction
  speculative decoding. It reuses the target's KV cache to skip its own prefill
  and cross-attends to target context while drafting (5.8.0).
- **EXAONE 4.5** is a 33B vision-language model with a 1.2B visual encoder,
  153,600-token vocabulary, context through 256K, and Multi-Token Prediction
  (5.8.0).
- **Cohere2Moe / Command A+** uses hybrid sliding/full attention and shared plus
  routed experts with a very large context. Its tensor-parallel plan is
  corrected in 5.9.0.
- **HRM-Text** is a base autoregressive model, not instruction-tuned and without
  chat templates. It uses slow planning and fast computation stacks, PrefixLM
  attention, per-head sigmoid output gates, and parameterless RMSNorm (5.9.0).

## General multimodal and vision-language models

- **Gemma 3** joins in 4.50.0. **Aya Vision** accepts image/text and works across
  23 languages. **SmolVLM2** accepts multiple images and video.
- **Llama 4 Maverick and Scout** load through
  `Llama4ForConditionalGeneration` and accept text plus images (4.51.0). The
  documented environment installs `transformers[hf_xet]`.
- **Phi-4 Multimodal** accepts text, image, and audio, emits text, and has a 128K
  context (4.51.0).
- **Qwen2.5-Omni** accepts text, images, audio, and video and can stream text and
  speech (4.52.1).
- **Janus and Janus-Pro** accept image/text and can emit text or images, but the
  caller must select one output mode rather than requesting interleaved output
  (4.52.1). **InternVL3** is also added.
- **Llama Guard 4** provides multimodal moderation (4.52.1).
- **Gemma 3n** accepts text, image, video, and audio with text output (4.53.0).
- **DeepSeek VL** accepts image/text and responds with text (4.54.0).
- **PerceptionLM** handles image and video understanding (4.54.0).
- **Command A Vision** supports captioning, visual QA, and document/chart
  understanding through Cohere2 Vision (4.55.0).
- **Ovis2, HunYuan, and GLM-4.5V** are integrated in 4.56.0.
- **Qwen3-VL** provides dense and MoE variants in Instruct and Thinking forms;
  **Qwen3 Omni MoE** provides unified multimodal generation (4.57.0).
- **LFM2-VL** accepts text and variable-resolution images. It preserves native
  resolution through 512×512 and tiles larger images; the 1.6B version also
  receives a global thumbnail (4.57.0).
- **Ernie 4.5 VL MoE** support appears in 5.2.0. Its class/config names change in
  5.3.0 to match vLLM and SGLang conventions.
- **ModernVBert** combines ModernBERT and SigLIP for visual-document
  understanding and retrieval; **ColModernVBert** emits ColPali-style
  multi-vector document-image embeddings (5.3.0).
- **Mistral 4** and **PI0** arrive in 5.4.0. PI0 is a vision-language-action
  model that predicts robot actions from visual observations and instructions.
- **Gemma 4** needs its own patch-budget preprocessing and must not receive
  standard ImageNet normalization (5.5.0).
- **Granite 4.1 Vision** targets chart, table, and semantic key-value extraction.
  It combines SigLIP2 with Window Q-Former projectors and injects vision features
  at eight language-model locations (5.8.0).

## Detection, segmentation, depth, and feature matching

- **ShieldGemma 2** classifies image-safety categories (4.50.0).
- **SigLIP2** adds NaFlex variable aspect ratios and resolutions (4.50.0).
- **Prompt Depth Anything** produces metric depth from an iPhone LiDAR prompt
  (4.50.0). **Distill Any Depth** follows in 4.51.0.
- **SAM-HQ** provides promptable high-quality segmentation but does not yet
  support fine-tuning in 4.52.1.
- **D-FINE** adds object detection in 4.52.1 and computes correct auxiliary
  losses with denoising disabled in 5.7.0.
- **MLCD** is integrated in 4.52.1.
- **V-JEPA 2** provides a video encoder and action-conditioned world model
  (4.53.0).
- **LightGlue** matches local features between image pairs and can pair with
  SuperPoint for pose or homography estimation (4.53.0).
- **EoMT** provides pipeline-compatible image segmentation (4.54.0) and can use
  a DINOv3 backbone in 5.1.0.
- **AIMv2** provides vision encoders, and **EfficientLoFTR** finds image
  correspondences for pose/homography estimation (4.54.0).
- **MM Grounding DINO** performs zero-shot grounding/detection with original MM
  Grounding DINO or LLMDet checkpoints (4.55.0).
- **DINOv3** produces dense visual features; **MetaCLIP 2** provides multilingual
  image/text representations; **Florence-2** supports prompted captioning,
  detection, and segmentation (4.56.0).
- **SAM 2** supports point/box-prompted segmentation for images and videos
  (4.56.0). **EdgeTAM** adds mobile-oriented real-time video segmentation in
  4.57.0.
- **VidEoMT** supports online video segmentation (5.4.0).
- **SAM3-LiteText** combines SAM3's ViT-H image encoder with a compact distilled
  MobileCLIP text encoder (5.6.0).
- **DEIMv2** provides real-time detection in eight sizes from X to Atto. Larger
  models use a Spatial Tuning Adapter to turn single-scale DINOv3 features into
  multiple scales; the smallest use pruned HGNetv2 (5.7.0).
- SAM3, EdgeTAM, and SAM3-Lite-Text require full rather than pooled
  `text_embeds` as of 5.9.0.

## Documents, OCR, and structured vision

- **ColQwen2** treats pages as images and emits multi-vector embeddings for
  late-interaction retrieval (4.53.0).
- **Kosmos-2.5** provides spatially grounded OCR and image-to-Markdown (4.56.0).
- **PP-DocLayoutV3** predicts instance-segmented layouts and reading order;
  **GLM-OCR** recognizes complex documents. They can form a two-stage layout
  then parallel-recognition pipeline (5.1.0).
- **PP-DocLayoutV2** combines RT-DETR element detection/classification with a
  pointer network for reading order (5.3.0).
- **UVDoc** rectifies single images or batches (5.4.0).
- **SLANeXt** recognizes table structure, while PP-OCRv5 mobile/server detector
  and recognizer families handle multilingual documents and scene text
  (5.4.0).
- **PPLCNet** supplies document-orientation, table, and text-line-orientation
  classifiers; **PPLCNetV3** is a CPU-oriented vision backbone (5.4.0).
- **Qianfan-OCR** is a 4B end-to-end image-to-text document model for structured
  parsing, tables, charts, document QA, and key-information extraction. Its
  Layout-as-Thought mode emits structured layout before the final result
  (5.6.0).
- **SLANet and SLANet_plus** provide lightweight CPU-oriented table structure
  recognition for documents and natural scenes (5.6.0).
- **PP-FormulaNet-L and PP-FormulaNet_plus-L** recognize formulas and table
  structures in documents and natural scenes (5.8.0).

## Speech, audio, and acoustic tokenization

- **CSM** provides contextual text-to-speech (4.52.1).
- **Dia** offers dialogue-oriented TTS with nonverbal cues and audio
  conditioning (4.53.0).
- **Kyutai STT** provides streaming-codec speech recognition through bilingual
  `kyutai/stt-1b-en_fr` and English-only `kyutai/stt-2.6b-en` (4.53.0).
- **Voxtral** adds audio to Ministral for transcription, translation, audio QA,
  summarization, and voice-driven function calls (4.54.0). The Mini 3B and Small
  24B checkpoints have 32K context and accept up to 30 minutes for transcription
  or 40 minutes for general audio understanding.
- **X-Codec** supports semantic-aware audio tokenization, music continuation,
  and text-to-sound synthesis (4.56.0).
- **Parakeet** adds CTC speech recognition through `ParakeetForCTC` (4.57.0).
- **VoxtralRealtime** performs incremental low-latency ASR over arriving chunks
  rather than requiring a complete file (5.2.0).
- **VibeVoice Acoustic Tokenizer** supports the continuous audio-tokenization
  design used by long-form multi-speaker speech synthesis (5.2.0).
- **VibeVoice ASR** handles up to 60 minutes of 24 kHz audio, hotwords, joint
  transcription/diarization/timestamps, more than 50 languages, and
  code-switching (5.3.0).
- **Higgs Audio V2** supports single/multi-speaker generation and zero-shot voice
  cloning. Its separate 24 kHz tokenizer encodes speech, music, and sound events
  at 25 fps without diffusion (5.3.0).
- **Music Flamingo** reasons across speech, sound, and music with audio sequences
  through 20 minutes (5.5.0).
- Audio models gain vLLM compatibility in 5.6.0.
- **Granite Speech Plus** provides prompted speech-to-text, speaker annotations,
  and word timestamps. Its projector concatenates the speech encoder's final
  hidden state with a configurable subset of intermediate states (5.8.0).
- **Parakeet TDT** is a separate Parakeet integration from CTC, and
  **AudioFlamingoNext** checkpoints are supported in 5.9.0.

## Time series, science, and earth observation

- **TimesFM** adds time-series forecasting in 4.52.1.
- **TimesFM 2.5** is a decoder-only zero-shot forecaster with continuous
  quantile prediction (5.3.0).
- **EVOLLA** generates protein language over sequences, structures, and user
  queries (4.54.0).
- **CHMv2** estimates forest-canopy height from high-resolution optical
  satellite imagery (5.4.0).

## Result-affecting integration fixes

- Dinov2 image classification handles register tokens correctly in 4.52.1.
- PerceptionLM video/non-tiled-image preprocessing, Fuyu inference, Qwen-VL
  video beam search, LLaVA-OneVision batches, and Idefics2/Idefics3/SmolVLM
  device placement are corrected in 4.56.0.
- Qwen2.5-VL still-image outputs may change in 5.6.0 because temporal RoPE
  scaling is no longer applied incorrectly.
- T5Gemma2 long cross-attention and Qwen3.5 cached linear attention are corrected
  in 5.7.0; GraniteMoeHybrid attention-only configs also stop crashing on a
  nonexistent Mamba mask.
