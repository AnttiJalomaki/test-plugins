# Identity, budgets, and virtual keys

## Spend and tag budgets

Since 1.93.0, a key that exceeds its budget is rate-limited rather than revoked. Workflows needing invalid credentials must revoke explicitly instead of treating budget exhaustion as a hard block.

Tag budgets require PostgreSQL and are created independently with `/tag/new`. Attach them to a virtual key through top-level `tags` or `metadata.tags`; requests may supply `metadata.tags` or `x-litellm-tags`. A multi-tag request is charged to every tag and is rejected if any tag is over budget.

```http
POST /tag/new
Authorization: Bearer <master-key>
Content-Type: application/json

{"name":"engineering","max_budget":500,"soft_budget":400,"budget_duration":"30d"}
```

`default_team_params` fills only fields omitted from `/team/new`, including SSO-created teams. Its `models` field applies only to SSO-created teams. In releases after v1.94.0rc1, quoted `budget_reset_time` values set wall-clock reset time in the configured timezone. Malformed values fail startup, and sub-day budgets ignore the setting.

```yaml
litellm_settings:
  default_team_params:
    max_budget: 100
    budget_duration: 30d
    models: [chat]
  budget_reset_time: "09:00"
```

`/key/update` can grant a temporary increase without changing the base budget. Supply both `temp_budget_increase` and `temp_budget_expiry`.

## Key aliases and authentication headers

A virtual key may expose a client-facing alias while limiting access to a configured model group. Pass both its allowed `models` and key-specific `aliases` when creating the key.

```json
{
  "models": ["free-tier"],
  "aliases": {"legacy-chat": "free-tier"},
  "duration": "30min"
}
```

`general_settings.litellm_key_header_name` moves virtual-key authentication out of `Authorization`, leaving that standard header for an upstream gateway. The alternate header value still has the form `Bearer <key>`.

```yaml
general_settings:
  master_key: sk-admin
  litellm_key_header_name: X-Litellm-Key
```

From v1.95.0-rc.1, `overwrite_user_with_key_hash: true` replaces the outgoing Chat Completions `user` with the virtual key SHA-256 hash, or a fixed master-key alias, even when the caller supplied a value. Custom-auth and JWT requests are unchanged.

## Key creation policy

`general_settings.custom_key_generate` loads an async callback before `/key/generate`. It receives `GenerateKeyRequest` and must return `{"decision": true}` to permit creation or `{"decision": false, "message": "..."}` to veto it.

`default_key_generate_params` fills omitted request fields. `upperbound_key_generate_params` clamps requested values to administrative ceilings instead of rejecting the request.

```yaml
litellm_settings:
  default_key_generate_params:
    max_budget: 1.5
    models: [standard-chat]
    team_id: core-infra
  upperbound_key_generate_params:
    max_budget: 100
    budget_duration: 10d
    duration: 30d
    max_parallel_requests: 1000
    tpm_limit: 1000
    rpm_limit: 1000
```

`key_generation_settings` independently governs team and personal key creation. Each scope can allow roles and require request fields such as `tags` for cost attribution.

```yaml
litellm_settings:
  key_generation_settings:
    team_key_generation:
      allowed_team_member_roles: [admin]
      required_params: [tags]
    personal_key_generation:
      allowed_user_roles: [proxy_admin]
      required_params: [tags]
```

## Key rotation and regeneration

The enterprise `POST /key/{key}/regenerate` endpoint rotates a key and may update its parameters. `grace_period` keeps old and new strings valid during cutover; omitting it or passing an empty value revokes the old value immediately.

Automatic rotation is controlled by `LITELLM_KEY_ROTATION_ENABLED` and is off by default. When enabled, the job checks every 86,400 seconds and uses a 600-second distributed lock by default. `LITELLM_KEY_ROTATION_GRACE_PERIOD` accepts a duration such as `24h`; an empty value revokes immediately.

## Authentication, teams, and membership

`custom_auth_run_common_checks` defaults to `false`. Enable it to apply Proxy model allowlists, budgets, and rate limits after custom authentication.

`fail_closed_budget_enforcement` also defaults off. When enabled, it checks every budgeted call against the database and returns 503 if neither Redis nor the database can establish spend. `allow_requests_on_db_unavailable` instead permits virtual-key calls that cannot be checked; reserve it for private-network deployments.

If a JWT has no team claims, authentication falls back to the user's database-backed team memberships.

Organization administrators can update teams with `PATCH /team/{team_id}` using JSON merge-patch semantics. `GET /key/list` accepts an `expires` filter. `lite auth print-token` emits a token suitable for Claude Code's `apiKeyHelper`, and the Microsoft Graph endpoint can be changed for GCC High.

## Auth caching and TPM accounting

`litellm_settings.enable_redis_auth_cache` shares virtual-key auth payloads across workers by using the response-cache Redis client. It requires `cache: true` with `cache_params.type: redis`. Set `general_settings.user_api_key_cache_ttl` when memory and Redis entries should have the same TTL.

A single key can enforce per-tag RPM. Router deployments can cap input TPM and output TPM independently. The v3 limiter reserves TPM before a call by default; `LITELLM_TPM_TOKEN_RESERVATION_ENABLED=false` skips the reservation and enforces from actual usage after the call.
