---
name: litellm-knowledge-patch
description: LiteLLM
version: 1.93.0
license: MIT
metadata:
  author: Nevaberry
---


# LiteLLM Knowledge Patch

Use this patch when implementing, reviewing, operating, or upgrading LiteLLM SDK, Router, or Proxy deployments.
Start with the compatibility and security checks below, then load only the topic references needed for the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [routing-and-resilience.md](references/routing-and-resilience.md) | Rate-limit enforcement, routing groups, ordered tiers, affinity, retries, fallbacks, health checks, and timeouts |
| [identity-budgets-and-keys.md](references/identity-budgets-and-keys.md) | Spend enforcement, tag budgets, key defaults and rotation, team behavior, JWTs, and rate limits |
| [proxy-security-and-networking.md](references/proxy-security-and-networking.md) | Validation, tenant isolation, authentication checks, MCP ingress, HTTP transport, hardening, and request bounds |
| [providers-protocols-and-mcp.md](references/providers-protocols-and-mcp.md) | Provider/model support, protocol bridges, Claude context handling, A2A, MCP authentication, filtering, and guardrails |
| [deployment-configuration.md](references/deployment-configuration.md) | Credentials, environment exposure, prompt framing, tokenizers, Redis separation, and config convergence |
| [operations-and-observability.md](references/operations-and-observability.md) | OpenTelemetry, drain behavior, database topology, Redis protection, Python runtime, and operational sizing |

## Apply the patch

1. Identify whether the code uses the SDK, `Router`, Proxy configuration, or management APIs.
2. Check the breaking and security-sensitive behaviors before changing configuration.
3. Read the routing reference when retries, fallbacks, limits, health, or sticky routing are involved.
4. Read the identity reference before changing budgets, teams, virtual keys, or key generation.
5. Read the protocol reference for Chat, Responses, Messages, MCP, A2A, or provider capability mapping.
6. Validate defaults explicitly; several protections and enforcement modes are opt-in.
7. Test multi-worker or multi-pod behavior with shared Redis state where the feature requires it.

## Breaking-change checks

### Spend limits throttle rather than revoke

Crossing a key budget now rate-limits the key instead of revoking it. Do not rely on spend exhaustion to invalidate credentials or to create an irreversible hard stop. Treat revocation and budget enforcement as separate controls.

### Proxy fallback test flags no longer work

The Proxy strips these incoming fields:

- `mock_testing_fallbacks`
- `mock_testing_context_fallbacks`
- `mock_testing_content_policy_fallbacks`

They remain usable only with direct `Router` calls. Exercise a real provider failure in a non-production environment when testing Proxy fallbacks.

### OpenTelemetry attribute names changed

Update queries and dashboards that used old LiteLLM error keys. LiteLLM-specific details are under `litellm.*`; streaming spans use `gen_ai.response.time_to_first_chunk`; failed calls emit `gen_ai.client.operation.exception`; v2 error spans expose `error.*` again.

### Team aliases can bypass ordering and failover

Request team deployments through `model_info.team_public_model_name`. A stale team `model_aliases` mapping can collapse the public name to one internal deployment. Remove that database alias, or temporarily set `LITELLM_ENABLE_TEAM_STALE_ALIAS_BYPASS=true` while upgrading.

### SSRF validation is on by default

`litellm_settings.user_url_validation` rejects URLs resolving to private, loopback, link-local, or otherwise non-global addresses. `user_url_allowed_hosts` matches the URL hostname exactly, including its port; for split-horizon DNS, allowlist the public hostname.

### Security boundaries have explicit escape hatches

- Responses IDs are user-bound unless `disable_responses_id_security` removes that protection.
- Non-admin spend-list results are caller-scoped unless `legacy_unscoped_spend_list_endpoints` restores global results.
- Client metadata tags remain usable unless `reject_clientside_metadata_tags` is enabled.
- Custom authentication does not run common model, budget, and rate-limit checks unless `custom_auth_run_common_checks` is enabled.

## Routing quick reference

### Enforce deployment RPM and TPM

Configured deployment `rpm` and `tpm` normally inform routing. Add the pre-call check to reject over-limit calls with 429 and `retry-after: 60`:

```yaml
router_settings:
  optional_pre_call_checks: [enforce_model_rate_limits]
```

RPM enforcement is exact. TPM is best-effort because actual token use is recorded after the response. Share Redis state across Proxy instances.

### Partition routing strategies

Use `routing_groups` to apply separate strategies to disjoint model-name sets:

```yaml
router_settings:
  routing_strategy: simple-shuffle
  routing_groups:
    - group_name: latency-sensitive
      models: [premium-chat]
      routing_strategy: latency-based-routing
      routing_strategy_args: {ttl: 60}
```

A model can appear in only one group, `default` is reserved, and ungrouped models inherit the top-level strategy. Runtime updates through `Router.update_settings()` or `/config/update` rebuild group state.

### Build ordered deployment tiers

Set `litellm_params.order`; lower values run first. The routing strategy balances deployments tied within a tier, retries are exhausted inside each tier, and model-level fallbacks run only after all tiers fail.

### Enable async weighted failover

For `simple-shuffle`, `enable_weighted_failover: true` re-picks a weighted, rate-limited same-group peer after an async failure before crossing to `fallbacks`. It does not affect synchronous calls, is capped by `max_fallbacks` (default `5`), and leaves context-window and content-policy errors on their own routes.

### Route encrypted Responses continuations

Assign every deployment a distinct `model_info.id` and enable:

```yaml
router_settings:
  optional_pre_call_checks: [encrypted_content_affinity]
```

Encrypted `rs_...` items then return to the deployment key that created them while ordinary traffic remains balanced.

### Select retry and fallback routes

The Router exhausts `num_retries` before crossing model groups and maintains distinct ordered routes for context-window, content-policy, and remaining errors. `request_timeout` bounds each attempt; `allowed_fails` and `cooldown_time` decide when a deployment leaves selection. A model-specific fallback mapping overrides `default_fallbacks`.

Request-scoped fallbacks may replace `model`, `messages`, `temperature`, and other inputs for an attempt, including embeddings and image generation. `disable_fallbacks: true` in the request or virtual-key metadata disables failover.

### Enforce context windows before provider calls

Set `router_settings.enable_pre_call_checks: true`. Use `model_info.max_input_tokens` to override a known limit and `model_info.base_model` when a deployment name hides the underlying model. If no deployment fits, `ContextWindowExceededError` can enter `context_window_fallbacks`.

## Security and operations quick reference

### Choose fail-open or fail-closed budgets

`fail_closed_budget_enforcement` is off by default. Enabling it checks every budgeted request against the database and returns 503 when neither Redis nor the database can establish spend. `allow_requests_on_db_unavailable` intentionally permits unchecked virtual keys and is suitable only for private-network deployments.

### Protect optional endpoints and interfaces

- `enable_drain_endpoint` is off by default. Without `drain_endpoint_token`, `/health/drain` is unauthenticated; otherwise callers need `X-Drain-Token`.
- `LITELLM_ENABLE_HSTS` is opt-in and applies only over HTTPS.
- `DISABLE_ADMIN_UI`, `NO_DOCS`, `NO_OPENAPI`, and `NO_REDOC` disable their respective interfaces independently.
- Secret redaction is on unless `LITELLM_DISABLE_REDACT_SECRETS=true`.
- File-backed OIDC credentials default to `/var/run/secrets,/run/secrets`; override with `LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS`.

### Bound work and disconnects

Set `max_request_size_mb` and `max_response_size_mb` to reject oversized traffic. `pass_through_request_timeout` defaults to 600 seconds, with endpoint-specific values taking precedence. `cancel_on_disconnect: true` cancels a non-streaming upstream request and records 499 after the client leaves.

### Share health and affinity state deliberately

`enable_health_check_routing` filters unhealthy deployments. Set a staleness threshold and optionally ignore transient 408/429 probe failures. `use_shared_health_check` stores health state in Redis. Sticky routing uses `deployment_affinity` or `session_affinity`; global affinity checks may be overridden per group with `model_group_affinity_config`.

## High-use feature quick reference

### Use per-tag and directional limits

A virtual key can enforce per-tag RPM. Router deployments can limit input TPM and output TPM independently, and local rate-limit errors can trigger gateway fallbacks. The v3 limiter reserves TPM before calls unless `LITELLM_TPM_TOKEN_RESERVATION_ENABLED=false` switches enforcement to actual post-call usage.

### Hide compatibility aliases

Use the object form to accept an alias without advertising it from model discovery endpoints:

```yaml
router_settings:
  model_group_alias:
    legacy-chat:
      model: chat
      hidden: true
```

### Mirror traffic silently

Router traffic mirroring sends a production request to a secondary model for evaluation. Its response is collected in the background and does not change the primary response or its latency.

### Separate response and coordination Redis

Coordination Redis may be configured independently from the response cache. The usage cache can be built from `REDIS_*` variables, and the `general_settings` request allowlist is applied to LiteLLM globals.

### Use the new gateway and protocol features

The gateway can register and invoke A2A agents alongside model and MCP routes. Chat Completions forwards `verbosity`; protocol bridges preserve custom-tool round trips, allowlists, and translated `reasoning_tokens`. MCP supports delegated credentials, OAuth token exchange, semantic tool filtering, and pre-call guardrail scanning.

For syntax, defaults, constraints, and management endpoints, use the corresponding topic reference rather than inferring behavior from this overview.
