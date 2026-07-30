# Routing and resilience

## Deployment limit enforcement

Deployment `rpm` and `tpm` normally guide routing rather than hard-blocking traffic. Add `enforce_model_rate_limits` to `router_settings.optional_pre_call_checks` to reject an over-limit call before it reaches the provider. The response is 429 with `retry-after: 60`. RPM is exact; TPM is best-effort because actual usage is recorded after the response. Multiple Proxy instances need shared Redis state.

```yaml
model_list:
  - model_name: chat
    litellm_params:
      model: provider/chat
      rpm: 60
      tpm: 90000
router_settings:
  optional_pre_call_checks: [enforce_model_rate_limits]
```

Virtual keys can also enforce per-tag RPM, and Router deployments can limit input TPM and output TPM separately. Local rate-limit errors are eligible to trigger gateway fallbacks. The v3 limiter reserves TPM before each call by default; set `LITELLM_TPM_TOKEN_RESERVATION_ENABLED=false` to enforce only after actual usage is known.

## Aliases and routing partitions

A hidden alias accepts an alternate request name without appearing in `/v1/models`, `/v1/model/info`, or `/v1/model_group/info`:

```yaml
router_settings:
  model_group_alias:
    legacy-chat:
      model: chat
      hidden: true
```

`routing_groups` applies different routing strategies and arguments to different `model_name` sets. Each model may occur in only one group, `default` is reserved, and ungrouped models use the top-level strategy. Updating `routing_groups` through `Router.update_settings(routing_groups=[...])` or `/config/update` rebuilds the group state at runtime.

```yaml
router_settings:
  routing_strategy: simple-shuffle
  routing_groups:
    - group_name: latency-sensitive
      models: [premium-chat]
      routing_strategy: latency-based-routing
      routing_strategy_args: {ttl: 60}
```

The complexity router supports keyword tier overrides, semantic keyword matching, custom technical keywords, and an optional classifier. It can emit a routing log for each decision.

## Mirroring, order, and same-group failover

Traffic mirroring sends production requests to a secondary model for evaluation. The secondary result is collected in the background and changes neither the primary result nor its latency.

Set `order` inside `litellm_params` to create deployment tiers. Lower values are tried first; the configured strategy balances deployments tied within a tier. Each tier receives its retries before the next tier is promoted, and model-level `fallbacks` start only after every order tier is exhausted.

```yaml
model_list:
  - model_name: chat
    litellm_params: {model: provider/chat-primary, order: 1}
  - model_name: chat
    litellm_params: {model: provider/chat-secondary, order: 2}
```

With `simple-shuffle`, `enable_weighted_failover` lets async calls exclude a failed deployment and re-pick a same-group peer using existing weights and limits before crossing into `fallbacks`. It is capped by `max_fallbacks` (default `5`), does not apply to synchronous calls, and does not intercept context-window or content-policy errors from their dedicated routes.

Team deployments should be requested through `model_info.team_public_model_name` so sibling deployments participate in order and failover. A legacy team `model_aliases` mapping can rewrite the public name to one internal deployment; remove it from the database or use `LITELLM_ENABLE_TEAM_STALE_ALIAS_BYPASS=true` temporarily during migration.

## Retry and fallback sequencing

The Router has separate ordered routes for context-window errors, content-policy violations, and all other errors. It exhausts `num_retries` before crossing to a fallback group. `request_timeout` bounds one attempt. `allowed_fails` and `cooldown_time` determine when and how long a failed deployment is removed. `default_fallbacks` serves groups without a mapping; a model-specific mapping wins.

```python
router = Router(
    model_list=model_list,
    fallbacks=[{"chat": ["backup"]}],
    context_window_fallbacks=[{"chat": ["long-context"]}],
    content_policy_fallbacks=[{"chat": ["policy-backup"]}],
)
```

```yaml
litellm_settings:
  num_retries: 3
  request_timeout: 10
  allowed_fails: 3
  cooldown_time: 30
  default_fallbacks: [emergency]
```

SDK and Proxy requests may provide `fallbacks`. An object entry can replace the model, messages, temperature, and other parameters for that attempt. Request-scoped fallback inputs also apply to embeddings and image generation.

```python
response = client.chat.completions.create(
    model="chat",
    messages=[{"role": "user", "content": "Summarize this."}],
    extra_body={"fallbacks": [{
        "model": "backup",
        "messages": [{"role": "user", "content": "Give a short summary."}],
        "temperature": 0,
    }]},
)
```

Set request-body `disable_fallbacks: true` for one Proxy request, or put the same flag in virtual-key metadata to suppress failover for all calls made with that key.

Since Proxy v1.85.0, incoming `mock_testing_fallbacks`, `mock_testing_context_fallbacks`, and `mock_testing_content_policy_fallbacks` are stripped. They work only on direct `Router` calls. Test Proxy fallback behavior with a real provider error in a non-production environment.

## Context, region, and exact targets

`router_settings.enable_pre_call_checks: true` is required to filter same-group deployments with insufficient context or reject an oversized input before the provider call. `model_info.max_input_tokens` overrides the known limit. If a deployment name obscures the underlying model, also set `model_info.base_model`. When none fit, `ContextWindowExceededError` lets `context_window_fallbacks` choose a larger group.

Pre-call checks can filter by region. Set `litellm_params.region_name` when the integration cannot infer it; location-bearing provider parameters may supply it automatically.

A fallback target can be an exact deployment `model_info.id` instead of a model group. Exact targeting deliberately skips that deployment's cooldown check. The serving deployment is reported by `x-litellm-model-id`.

```yaml
model_list:
  - model_name: chat
    litellm_params:
      model: provider/emergency-chat
    model_info:
      id: emergency-deployment
litellm_settings:
  fallbacks: [{chat: [emergency-deployment]}]
```

A wildcard deployment such as `provider/*` makes concrete prefixed model names valid fallback targets without enumerating them:

```yaml
model_list:
  - model_name: "provider/*"
    litellm_params:
      model: "provider/*"
litellm_settings:
  fallbacks: [{chat: [provider/backup-chat]}]
```

## Affinity and shared health

Encrypted Responses items such as `rs_...` can be continued only through the deployment key that created them. Give deployments distinct `model_info.id` values and add `encrypted_content_affinity` to `optional_pre_call_checks`. Follow-ups return to the origin; ordinary requests remain balanced.

`deployment_affinity` and `session_affinity` provide other sticky-routing modes independently of encrypted-content affinity. `deployment_affinity_ttl_seconds` defaults to `3600`. `model_group_affinity_config` selects checks per group; groups absent from that mapping inherit the global pre-call checks.

`enable_health_check_routing` filters unhealthy deployments. `health_check_staleness_threshold` expires old probe data, and `health_check_ignore_transient_errors` keeps 408 and 429 probes out of routing and cooldown decisions. `use_shared_health_check` moves health state into Redis for multi-instance deployments.

## Stall-specific timeouts

Router `ttft_timeout` catches a provider that never emits its first token; it internally streams even a nominally non-streaming call. `stream_idle_timeout` detects long token gaps. `LITELLM_MAX_STREAMING_DURATION_SECONDS` caps the entire stream, while `LITELLM_STREAM_INACTIVITY_TIMEOUT_SECONDS` catches async providers that emit keepalives but no content chunks.
