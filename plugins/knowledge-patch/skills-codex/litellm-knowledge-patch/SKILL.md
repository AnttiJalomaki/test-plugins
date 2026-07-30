---
name: litellm-knowledge-patch
description: LiteLLM
version: 1.93.0
license: MIT
metadata:
  author: Nevaberry
---


# LiteLLM

Use this skill when implementing or operating LiteLLM SDK, Router, or Proxy
features whose exact configuration, fallback, authentication, budget, MCP, or
protocol behavior matters.

## Reference index

| Reference | Topics |
| --- | --- |
| [routing-and-fallbacks.md](references/routing-and-fallbacks.md) | Routing groups, limits, deployment ordering, failover, fallback targets, pre-call checks, health, and affinity |
| [proxy-security-and-runtime.md](references/proxy-security-and-runtime.md) | Authentication, SSRF and request validation, tenant isolation, database topology, HTTP bounds, and hardening |
| [keys-budgets-and-governance.md](references/keys-budgets-and-governance.md) | Spend enforcement, tag budgets, rate limits, virtual keys, rotation, team defaults, and generation policy |
| [mcp-agents-and-guardrails.md](references/mcp-agents-and-guardrails.md) | A2A agents, MCP credential delegation, token exchange, filtering, guardrails, ingress, and grants |
| [models-and-protocols.md](references/models-and-protocols.md) | Providers, model metadata, Chat/Responses bridges, classifiers, credentials, environments, prompts, and tokenizers |
| [operations-and-observability.md](references/operations-and-observability.md) | Telemetry, Redis roles, timeouts, draining, database pools, config discovery, and Python compatibility |

## Working method

1. Identify whether the code uses direct SDK calls, `Router`, or the Proxy;
   similarly named settings do not always apply at every layer.
2. Inspect both static config and relevant environment variables before
   changing defaults or diagnosing behavior.
3. Read the topic reference that owns the behavior. Follow cross-links only
   when a request spans multiple subsystems.
4. Preserve distinct error paths for context-window, content-policy, local
   rate-limit, authentication, and provider failures.
5. In multi-instance deployments, verify that coordination state lives in
   shared Redis wherever the feature requires it.
6. Test routing and security changes with non-production credentials and
   observable failure conditions.

## Breaking behavior to check first

### Budget exhaustion throttles a key

A key over its spend limit is rate-limited rather than revoked. Do not build
cutover or cleanup automation around immediate key revocation. Treat the
result as a throttling state and use explicit rotation or revocation when the
key string must stop working.

### Telemetry attribute names changed

Update queries and processors that expect older LiteLLM error keys. Specific
error details use the `litellm.*` namespace, streaming spans report
`gen_ai.response.time_to_first_chunk`, failed calls emit
`gen_ai.client.operation.exception`, and v2 error spans expose `error.*`
attributes.

### Proxy fallback test flags do nothing

Do not send `mock_testing_fallbacks`,
`mock_testing_context_fallbacks`, or
`mock_testing_content_policy_fallbacks` through the Proxy. They are stripped
from incoming requests. Exercise a real provider failure in an isolated
environment, or use the flags only with direct `Router` calls.

## Routing quick reference

### Enforce deployment RPM and TPM

Deployment `rpm` and `tpm` normally inform selection; they do not hard-block
traffic. Add the pre-call check to reject excess traffic locally:

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

Expect a 429 with `retry-after: 60`. RPM accounting is exact; TPM is
best-effort because actual usage arrives after the response. Use shared Redis
across Proxy instances.

### Order deployments before balancing

Set `litellm_params.order` on sibling deployments. Lower values run first,
the routing strategy balances ties, and each tier consumes its retries before
the next tier. Model-group `fallbacks` begin only after all tiers fail.

```yaml
model_list:
  - model_name: chat
    litellm_params: {model: provider/primary, order: 1}
  - model_name: chat
    litellm_params: {model: provider/secondary, order: 2}
```

### Keep fallback paths distinct

Configure `fallbacks`, `context_window_fallbacks`, and
`content_policy_fallbacks` independently. LiteLLM exhausts `num_retries`
before crossing to a fallback model group. Use `default_fallbacks` only for
groups without a model-specific mapping.

```yaml
litellm_settings:
  num_retries: 3
  request_timeout: 10
  allowed_fails: 3
  cooldown_time: 30
  fallbacks: [{chat: [backup]}]
  context_window_fallbacks: [{chat: [long-context]}]
  content_policy_fallbacks: [{chat: [policy-backup]}]
  default_fallbacks: [emergency]
```

Set `disable_fallbacks: true` in a request or virtual-key metadata when
failover must not occur.

### Reject oversized context before the provider

Enable `router_settings.enable_pre_call_checks`. Supply
`model_info.max_input_tokens` when the deployment-specific limit differs, and
`model_info.base_model` when a deployment name hides the underlying model.
When no deployment fits, `ContextWindowExceededError` can enter the dedicated
context-window fallback route.

### Choose affinity deliberately

Use `encrypted_content_affinity` for encrypted Responses items, which must
return to the deployment key that created them. Give each deployment a unique
`model_info.id`. Use `deployment_affinity` or `session_affinity` for ordinary
stickiness; configure per-group overrides with
`model_group_affinity_config`.

## Proxy safety quick reference

### Keep URL validation enabled

`litellm_settings.user_url_validation` defaults to `true` and blocks
non-global DNS results. Put exact URL hostnames in
`user_url_allowed_hosts`, including the port when present. In split-horizon
DNS, allowlist the public hostname rather than its private resolved address.

### Make authentication checks explicit

`custom_auth_run_common_checks` defaults to `false`. Enable it when custom
authentication must still enforce model allowlists, budgets, and rate limits.
`fail_closed_budget_enforcement` also defaults off; enabling it returns 503
when neither Redis nor the database can establish spend.

Use `allow_requests_on_db_unavailable` only for an intentional
private-network fail-open design. Keep Responses ID user binding and
caller-scoped spend endpoints enabled unless compatibility requires their
legacy switches.

### Protect drain and exposed interfaces

`enable_drain_endpoint` exposes `GET /health/drain`. Configure
`drain_endpoint_token`; otherwise the endpoint is unauthenticated. Use the
independent `DISABLE_ADMIN_UI`, `NO_DOCS`, `NO_OPENAPI`, and `NO_REDOC`
switches to remove interfaces, and enable HSTS only when the service is
actually served over HTTPS.

## Keys and budgets quick reference

Tag budgets require PostgreSQL and are created through `/tag/new`. Attach tags
to a key or request; a multi-tag request is charged to every tag and rejected
if any one is over budget. A key may expose a client alias while retaining an
allowed model group:

```json
{
  "models": ["free-tier"],
  "aliases": {"legacy-chat": "free-tier"},
  "tags": ["engineering"]
}
```

Use `default_key_generate_params` for omitted fields and
`upperbound_key_generate_params` to clamp requested values. Apply
`key_generation_settings` separately to team and personal keys when roles or
required attribution fields differ.

For planned rotation, `/key/{key}/regenerate` supports a `grace_period` in
which old and new strings both work. Automatic rotation is disabled unless
`LITELLM_KEY_ROTATION_ENABLED` is set.

## MCP and protocol quick reference

Choose MCP authentication by trust boundary:

- Use `true_passthrough` or `oauth_delegate` for client-held credentials.
- Use `dcr_bridge` for sealed credential transport and OAuth discovery,
  registration, and token relays with mandatory PKCE S256.
- Use `oauth2_token_exchange` with the `entra_obo` profile for on-behalf-of
  exchange, and persist the selected `oauth2_flow`.

Set `PROXY_BASE_URL` to the exact public origin for MCP OAuth behind ingress;
do not include a path or trailing slash. Require explicit key and end-user MCP
grants when empty grants must not inherit broader access.

When bridging protocols, distinguish
`use_chat_completions_url_for_anthropic_messages` from
`route_all_chat_openai_to_responses`; they reverse different request
directions.

## Deployment review checklist

- Confirm shared Redis for strict limits, shared health, coordination, locks,
  and other cross-worker state used by the selected features.
- Confirm database pool capacity as instances multiplied by workers
  multiplied by `database_connection_pool_limit`.
- Set request, response, pass-through, first-token, idle-token, and total
  stream limits independently.
- Validate hidden aliases, environment-specific model exposure, and team
  public model names through the model-list endpoints expected by clients.
- Check that secret redaction remains enabled and OIDC file credentials stay
  inside approved directories.
- Verify tags, user identity rewriting, and client-supplied metadata cannot
  bypass cost attribution or tenant isolation.
- Exercise fallback order, cooldown, region filtering, and affinity with
  response headers and telemetry enabled.
