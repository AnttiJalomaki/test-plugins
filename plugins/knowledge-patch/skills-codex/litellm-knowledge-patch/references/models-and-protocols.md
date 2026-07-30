# Models, providers, and protocol bridges

## Provider and model support

The OpenAI-compatible `meta` provider serves `meta/muse-spark-1.1` through
Chat Completions, `/v1/messages`, and Responses (since 1.93.0).

The model map includes:

- `gpt-5.6`, `gpt-5.6-sol`, `gpt-5.6-terra`, and `gpt-5.6-luna` for OpenAI
  and Azure
- `gpt-realtime-2.1` and `gpt-realtime-2.1-mini`
- `xai/grok-4.5` and `xai/grok-4.5-latest`
- `meta/muse-spark-1.1`
- `jp.anthropic.claude-opus-4-8`
- `vertex_ai/chirp_3`

GPT-5.6 metadata includes priority, flex, batch, and over-272K-token pricing
tiers. Bedrock regional inference profiles resolve regional pricing through
`get_model_info`.

## Chat and Responses translation

Chat Completions forwards the `verbosity` parameter to providers. The
chat-to-Responses bridge preserves Codex CLI custom-tool round trips and
allowlists, as well as `reasoning_tokens` in translated usage (since 1.93.0).

Use `litellm_settings.use_chat_completions_url_for_anthropic_messages` to send
OpenAI-compatible `/v1/messages` through Chat Completions instead of
Responses. Use `route_all_chat_openai_to_responses` to send OpenAI Chat
Completions through the Responses bridge. Corresponding `LITELLM_*`
environment variables exist for both switches.

## Claude capability mapping

Bedrock Claude Invoke retains `clear_tool_uses_20250919` context edits and
emits the `context-management-2025-06-27` beta. Mapped Claude 4.8 and later
models advertise `supports_mid_conversation_system`.

Adaptive thinking and effort translate for pre-4.6 Anthropic models,
including Vertex model names with an `@default` suffix.

## Complexity routing

The complexity auto-router supports keyword tier overrides, semantic keyword
matching, custom technical keywords, and an optional LLM classifier. Enable
its routing log when decisions must be audited.

## Reusable named credentials

Define credentials once in top-level `credential_list`, then reference the
entry through `litellm_credential_name`. Every credential requires a
`credential_info` mapping, even when empty:

```yaml
model_list:
  - model_name: chat
    litellm_params:
      model: azure/gpt-4o
      litellm_credential_name: azure-prod
credential_list:
  - credential_name: azure-prod
    credential_values:
      api_key: os.environ/AZURE_API_KEY
      api_base: os.environ/AZURE_API_BASE
    credential_info: {}
```

## Environment-scoped exposure

Set `LITELLM_ENVIRONMENT` to `production`, `staging`, or `development`. Put
the allowed values in each deployment's
`model_info.supported_environments`; LiteLLM exposes that model only in the
selected environments.

## Prompt framing

Override automatically detected prompt formatting directly in a Proxy
model's `litellm_params`. A template can provide initial and final text,
per-role `pre_message` and `post_message` strings, and `bos_token` and
`eos_token`:

```yaml
litellm_params:
  model: huggingface/example/instruct
  initial_prompt_value: "\n"
  roles:
    user: {pre_message: "<|im_start|>user\n", post_message: "<|im_end|>"}
    assistant: {pre_message: "<|im_start|>assistant\n", post_message: "<|im_end|>"}
  final_prompt_value: "\n"
```

## Custom token counting

Set `model_info.custom_tokenizer` to make `/utils/token_counter` use a
specified Hugging Face tokenizer for a Proxy model. Provide `identifier` and
optionally `revision`; a private tokenizer can read its credential through
`auth_token`.
