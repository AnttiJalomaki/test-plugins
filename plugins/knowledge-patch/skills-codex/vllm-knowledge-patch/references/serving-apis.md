# Serving APIs

## Endpoint growth and core compatibility

- In `0.7-0.10`, serving added Jina- and Cohere-compatible reranking,
  `/score`, streaming transcription, and the Responses API. The request
  `model` field became optional.
- Operational routes added `/load`, `/is_sleeping`, and tokenizer/chat-template
  information through `get_tokenizer_info`. The HTTP server also gained SSL
  key rotation.
- Completions and Responses accept `cache_salt`. Requests gained
  `return_tokens_as_token_id`, image embeddings, `allowed_token_ids`,
  `mm_processor_kwargs`, `bad_words`, and chunking for audio longer than 30
  seconds.
- `LLM.chat` gained `chat_template_kwargs`; embeddings gained truncation and
  `tokenization_kwargs`; usage can report `cached_tokens`.
- V1 added structured output and reasoning output, backend-specific guided
  decoding, compatibility among structured output, thinking, and speculation,
  XGrammar `tool_choice: required`, Guidance structural tags, and `$defs`.
  `--enable-reasoning` was deprecated.

## Responses, Chat, and Completions

- Across `0.11-0.14`, Responses added MCP tools, multi-turn non-Harmony
  requests, reasoning-item input, parsed tool arguments,
  `parallel_tool_calls` compliance, tool filtering, Browser/Container MCP
  tools, extra body parameters, and MCP streaming.
- In `0.15-0.18`, Responses added partial-message generation,
  `include_stop_str_in_output`, `prompt_cache_key`, sampling parameters,
  returned token IDs, structured outputs, streamed reasoning-part events,
  and streaming tool/function calls.
- In `0.19-0.22`, `/v1/chat/completions/batch` arrived and Responses or
  Completions gained streamed required/named tool choices,
  `system_fingerprint`, `prompt_embeds` content parts,
  `chat_template_kwargs`, configurable truncation side, and
  `thinking_token_budget`; `reasoning_effort` maps to `enable_thinking`.
- In `0.23-0.26`, strict tool calling became available in Chat Completions
  and Responses for supported parsers. Responses added namespace tools,
  per-request timing metrics, and `return_loss_mask`; Python-backed endpoints
  expose `logprob_token_ids`, and `/v1/completions` accepts `bad_words`.
- Responses are created at `/v1/responses`, retrieved at
  `/v1/responses/{response_id}`, and canceled at
  `/v1/responses/{response_id}/cancel`.
- OpenAI-style Completions does not support `suffix`; Chat does not support
  `image_url.detail`. Chat accepts but ignores `user`.
- `parallel_tool_calls=false` guarantees at most one tool call. The default
  `true` permits multiple calls but cannot force a model to emit them.

## Reasoning, structured output, and parsers

- Version 0.14 added `reasoning_effort` and
  `--default-chat-template-kwargs`. Parser additions included SeedOSS,
  Qwen3-Coder XML, DeepSeek-V3.2, Gigachat 3, Holo2, FunctionGemma, and
  GLM-4.7.
- Versions 0.23-0.25 unified reasoning and tool parsing behind
  `Parser.parse`, then added the Streaming Parser Engine and parsers for
  Qwen3, MiniMax-M2, Nemotron V3, Kimi K2.5-K2.7, Seed-OSS, and DeepSeek V4.
- Anthropic-compatible Messages added thinking blocks, token counting,
  `tool_choice=none`, and redacted thinking blocks in `0.15-0.18`.
- In `0.23-0.26`, Anthropic-compatible Messages added structured output,
  effort, system-role messages inside the message array, cache-usage
  reporting, and system messages in the middle of a conversation.

## Embeddings, scoring, and reranking

- Version 0.13 introduced embedding `encoding_format=bytes_only` and multiple
  image/audio items per request. Version 0.14 added
  `continue_final_message` to `/embeddings`.
- Version 0.15 added BGE-M3 sparse and ColBERT embeddings. `/score` accepts
  either `data_1`/`data_2` or `queries`/`documents`.
- Later releases added sparse-embedding IO, multimodal late-interaction
  scoring, and Cohere Embed v2 compatibility.
- Version 0.19 permits multiple embedding types in one call.
- Version 0.24 lets `/v1/embeddings` accept messages and
  `chat_template_kwargs`.

## Speech and realtime interfaces

- Whisper `verbose_json` gained timestamps, then per-segment `avg_logprob`
  and `compression_ratio`.
- In `0.15-0.18`, speech APIs added batch transcription and translation,
  automatic language detection, and beam search for offline and online
  transcription.
- Version 0.16 added WebSocket realtime audio through the Voxtral
  infrastructure; 0.17 extended realtime streaming to Qwen3-ASR.
- A GPU-less prompt preprocessing path began with render endpoints and became
  `vllm launch render`, allowing rendering to be separated from GPU inference.

## Health, lifecycle, and administration

- From 0.11, health returns HTTP 503 when the engine is dead.
- Version 0.13 added `/reset_prefix_cache` for KV connectors; 0.14 added
  `/server_info`.
- Serving controls in `0.15-0.18` include deriving `api_server_count` from DP
  size, `--ssl-ciphers`, nested YAML,
  `--disable-access-log-for-endpoints`, multimodal/encoder-cache clearing,
  `--distributed-timeout-seconds`, and a graceful-shutdown timeout for
  in-flight requests.
- Version 0.19 added `/v1/chat/completions/batch`; 0.21 added
  `/start_weight_update` and `/finish_weight_update` for RLHF.
- Version 0.26 introduced endpoint plugins and `/abort_requests` on the RLHF
  development router.

## Python, Rust, and gRPC frontends

- Version 0.14 added a binary HTTP/2-multiplexed gRPC entrypoint alongside
  REST.
- Version 0.22 added an experimental in-tree Rust frontend with
  data-parallel integration and API-key authorization for `/v2`.
- In `0.23-0.26`, the Rust frontend added streaming `generate`, dynamic LoRA
  routes, `/server_info`, `--enable-request-id-headers`, API-key auth, CORS,
  `/pause`, `/resume`, `/is_paused`, `/get_world_size`, static HTTPS,
  HTTP/gRPC mTLS, video/audio, native `vllm-bench`, and TorchCodec video
  decoding.
- Despite its TLS work, the 0.25 gRPC notes classify the interface as insecure
  and suitable only for private use.

## Request validation and safety

- Versions 0.23-0.26 reject out-of-vocabulary token IDs, non-positive
  parallelism/scheduling values, non-finite temperature or repetition
  penalties, degenerate structured-output configurations, and request-level
  GPU video-backend selection.
- Invalid image URLs return HTTP 422. Regex and derender work are
  resource-bounded.
- Version 0.14 fixed invalid UTF-8 tokens, tool-call stream completion,
  tools-plus-`response_format` crashes, Voxtral transcription, CPU RoPE under
  `--enforce-eager`, and a stuck CPU scheduler caused by encoder-cache leaks.
