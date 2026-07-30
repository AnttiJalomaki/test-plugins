# Routing, rate limits, and fallbacks

## Deployment selection

### Strict deployment limits

Deployment `rpm` and `tpm` are routing hints unless
`router_settings.optional_pre_call_checks` contains
`enforce_model_rate_limits`. With that check, LiteLLM rejects an over-limit
call before the provider with 429 and `retry-after: 60`. RPM enforcement is
exact. TPM enforcement is best-effort because actual token usage is recorded
after the response. Multiple Proxy instances require shared Redis state.

Router deployments can also set separate input and output TPM limits. Local
rate-limit failures are eligible to enter gateway fallback handling (since
1.93.0).

### Model-group aliases

Use the object form of `router_settings.model_group_alias` with `hidden: true`
to accept a legacy or alternate request name without listing it in
`/v1/models`, `/v1/model/info`, or `/v1/model_group/info`:

```yaml
router_settings:
  model_group_alias:
    legacy-chat:
      model: chat
      hidden: true
```

### Per-model routing groups

Use `router_settings.routing_groups` to apply different strategies and
strategy arguments to selected `model_name` sets:

```yaml
router_settings:
  routing_strategy: simple-shuffle
  routing_groups:
    - group_name: latency-sensitive
      models: [premium-chat]
      routing_strategy: latency-based-routing
      routing_strategy_args: {ttl: 60}
```

A model may belong to only one group, and `default` is reserved. Ungrouped
models retain the top-level strategy. Updating `routing_groups` with
`Router.update_settings(...)` or `/config/update` rebuilds group state at
runtime.

### Mirroring

Configure Router traffic mirroring to send a production request to a
secondary model for evaluation. LiteLLM collects the secondary response in
the background; it neither replaces the primary response nor adds secondary
latency to it.

### Ordered deployment tiers

Set `order` inside each deployment's `litellm_params`. Lower values run first,
and the active routing strategy balances deployments tied in a tier. A tier
receives all configured retries before promotion to the next tier.
Model-group fallbacks begin only after every order tier is exhausted.

### Async weighted failover

With `simple-shuffle`, set `enable_weighted_failover: true` to let an async
call exclude a failed deployment and choose another peer in the same model
group using existing weights and rate limits. It does not apply to synchronous
calls. `max_fallbacks` caps these attempts and defaults to 5. Context-window
and content-policy failures continue to use their dedicated routes.

### Team model names

Request team deployments through `model_info.team_public_model_name` so all
sibling deployments participate in ordering and failover. A stale team
`model_aliases` row can rewrite the public name to one internal deployment.
Remove that database alias, or temporarily set
`LITELLM_ENABLE_TEAM_STALE_ALIAS_BYPASS=true` while migrating.

## Fallback processing

### Sequence and cooldown

The Router maintains separate ordered routes for general, context-window, and
content-policy errors. It exhausts `num_retries` before crossing model groups.
`request_timeout` bounds each attempt. `allowed_fails` determines when a
deployment enters cooldown, and `cooldown_time` controls how long it remains
out of selection. A model-specific fallback mapping takes precedence over
`default_fallbacks`.

```python
router = Router(
    model_list=model_list,
    fallbacks=[{"chat": ["backup"]}],
    context_window_fallbacks=[{"chat": ["long-context"]}],
    content_policy_fallbacks=[{"chat": ["policy-backup"]}],
)
```

### Request-scoped fallback parameters

An SDK or Proxy request may supply `fallbacks`. Use an object fallback to
replace the model, messages, temperature, or other parameters on that
attempt. Request-scoped fallback objects also apply to operations such as
embeddings and image generation:

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

### Exact deployment targets

A fallback target may be a deployment's `model_info.id`. This selects only
that deployment and deliberately skips its cooldown check. Inspect
`x-litellm-model-id` to identify the deployment that served the response.

### Wildcard targets

Register a wildcard deployment such as `provider/*` to make concrete
provider-prefixed model names valid fallback targets without listing each
model:

```yaml
model_list:
  - model_name: "provider/*"
    litellm_params: {model: "provider/*"}
litellm_settings:
  fallbacks: [{chat: ["provider/backup-chat"]}]
```

### Disable fallback

Set `disable_fallbacks: true` in the Proxy request body to suppress failover
for one call. Put the same value in virtual-key metadata to suppress failover
for every request authenticated by that key.

### Test fallbacks

Proxy releases since v1.85.0 strip incoming `mock_testing_fallbacks`,
`mock_testing_context_fallbacks`, and
`mock_testing_content_policy_fallbacks`. These flags work only on direct
`Router` calls. Trigger a real provider failure in a non-production
environment when testing Proxy behavior.

## Pre-call filtering

Set `router_settings.enable_pre_call_checks: true` to filter deployments by
input size or region before provider invocation.

For context limits, set `model_info.max_input_tokens` when it must override
known metadata. Also set `model_info.base_model` when a deployment name does
not reveal the underlying model. If no same-group deployment fits, LiteLLM
raises `ContextWindowExceededError`, enabling a `context_window_fallbacks`
route.

For region filtering, set `litellm_params.region_name` when the provider
integration cannot infer location. Integrations with location-bearing
parameters can infer it automatically.

## Health, timeout, and affinity checks

Set `enable_health_check_routing` to exclude unhealthy deployments.
`health_check_staleness_threshold` expires old observations, while
`health_check_ignore_transient_errors` prevents 408 and 429 probes from
affecting routing or cooldown. Set `use_shared_health_check` to keep health
state in Redis across Proxy instances.

Use Router `ttft_timeout` to detect a provider that never emits its first
token; LiteLLM internally streams even a nominally non-streaming request for
this check. Use `stream_idle_timeout` to detect excessive token gaps.
`LITELLM_MAX_STREAMING_DURATION_SECONDS` caps total stream lifetime, while
`LITELLM_STREAM_INACTIVITY_TIMEOUT_SECONDS` detects async streams that send
keepalives but no content chunks.

Encrypted Responses items such as `rs_...` must return through the deployment
key that created them. Give sibling deployments distinct `model_info.id`
values and enable `encrypted_content_affinity`.

For general sticky routing, enable `deployment_affinity` or
`session_affinity`. `deployment_affinity_ttl_seconds` defaults to 3600.
Use `model_group_affinity_config` to select checks per model group; groups not
listed inherit the global checks.
