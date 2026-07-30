# Providers, protocol bridges, and MCP

## Provider and model support

The OpenAI-compatible `meta` provider serves `meta/muse-spark-1.1` through Chat Completions, `/v1/messages`, and Responses.

The model map includes:

- `gpt-5.6`, `gpt-5.6-sol`, `gpt-5.6-terra`, and `gpt-5.6-luna` for OpenAI and Azure
- `gpt-realtime-2.1` and `gpt-realtime-2.1-mini`
- `xai/grok-4.5` and `xai/grok-4.5-latest`
- `meta/muse-spark-1.1`
- `jp.anthropic.claude-opus-4-8`
- `vertex_ai/chirp_3`

GPT-5.6 metadata accounts for priority, flex, batch, and over-272K-token pricing tiers. Bedrock regional inference profiles resolve their regional price through `get_model_info`.

## Chat, Responses, and Messages bridges

Chat Completions forwards `verbosity` to providers. The Chat-to-Responses bridge preserves Codex CLI custom-tool round trips and allowlists and retains `reasoning_tokens` in translated usage.

`litellm_settings.use_chat_completions_url_for_anthropic_messages` sends OpenAI-compatible `/v1/messages` through Chat Completions instead of Responses. `route_all_chat_openai_to_responses` sends OpenAI Chat Completions through the Responses bridge. Both settings have corresponding `LITELLM_*` environment variables.

## Claude context capabilities

Bedrock Claude Invoke retains `clear_tool_uses_20250919` context edits and emits the `context-management-2025-06-27` beta. Mapped Claude 4.8 and later models advertise `supports_mid_conversation_system`. Adaptive thinking and effort are translated for pre-4.6 Anthropic models, including Vertex model names ending in `@default`.

## A2A gateway

The gateway can add and invoke A2A agents alongside model and MCP routes, avoiding a separate agent gateway.

## Client-held MCP credentials

MCP servers support `true_passthrough` and `oauth_delegate` authentication. Upstream OAuth discovery is bound to each server. The `dcr_bridge` path carries client-held credentials in a sealed envelope and exposes discovery plus registration and token relays; PKCE S256 is mandatory.

MCP configuration also accepts `oauth2_token_exchange` and the `entra_obo` exchange profile through the REST API and dashboard. The chosen `oauth2_flow` is stored explicitly, legacy null values are backfilled at startup, and outbound concurrency limits cover on-behalf-of tool calls.

## MCP semantic filtering

Before applying the semantic filter, LiteLLM expands `litellm_proxy` tools. The result reports how many tools were removed and preserves complete tool names in its response header. Context-window failures are surfaced and filtering fails closed.

## MCP guardrails

Model Armor supports `pre_mcp_call` and `during_mcp_call`; Content Filter supports `pre_mcp_call`. `skip_unscannable_attachments` lets Model Armor pass reference-only attachments through and removes the attachment-count cap.
