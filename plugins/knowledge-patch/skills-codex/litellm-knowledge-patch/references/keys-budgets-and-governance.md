# Keys, budgets, and governance

## Spend and rate-limit behavior

When a key exceeds its spend limit, LiteLLM rate-limits it rather than
revoking it (since 1.93.0). Use explicit revocation or key rotation when the
old credential must become invalid.

A single key can enforce separate per-tag RPM limits. Router deployments can
enforce input TPM and output TPM independently. Local rate-limit errors can
trigger gateway fallbacks.

The v3 rate limiter reserves TPM before a call by default. Set
`LITELLM_TPM_TOKEN_RESERVATION_ENABLED=false` to skip the reservation and
enforce TPM after the call using actual usage.

## Team defaults and reset times

`litellm_settings.default_team_params` fills only fields omitted from
`/team/new`, including SSO-created teams. Its `models` value applies only to
teams created through SSO:

```yaml
litellm_settings:
  default_team_params:
    max_budget: 100
    budget_duration: 30d
    models: [chat]
  budget_reset_time: "09:00"
```

In releases after v1.94.0rc1, a quoted `budget_reset_time` sets the wall-clock
reset time in the configured timezone. Malformed values stop startup.
Sub-day budget durations ignore the setting.

## Tag budgets

Tag budgets require PostgreSQL and are created independently with `/tag/new`.
Attach tags to a virtual key through top-level `tags` or `metadata.tags`.
Attach tags per request through `metadata.tags` or `x-litellm-tags`.

A request with multiple tags is charged to every tag. LiteLLM rejects the
request if any attached tag is over budget.

```http
POST /tag/new
Authorization: Bearer <master-key>
Content-Type: application/json

{"name":"engineering","max_budget":500,"soft_budget":400,"budget_duration":"30d"}
```

## Key aliases and authentication header

Pass both `models` and an `aliases` mapping to `/key/generate` when a virtual
key should expose a client-facing name but remain restricted to a configured
model group:

```json
{
  "models": ["free-tier"],
  "aliases": {"legacy-chat": "free-tier"},
  "duration": "30min"
}
```

Set `general_settings.litellm_key_header_name` to move virtual-key
authentication out of `Authorization`, leaving that header available to an
upstream gateway. The custom header value still has the
`Bearer <key>` form.

## Key-generation policy

`general_settings.custom_key_generate` loads an async callback before
`/key/generate`. It receives `GenerateKeyRequest` and returns
`{"decision": true}` to proceed or
`{"decision": false, "message": "..."}` to veto generation.

Use `litellm_settings.default_key_generate_params` to fill omitted fields.
Use `upperbound_key_generate_params` to clamp requests to administrative
ceilings rather than reject them:

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

Use `key_generation_settings` to govern team and personal keys separately.
Each scope can allow selected roles and require parameters such as `tags`:

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

## Rotation and temporary budgets

The enterprise `POST /key/{key}/regenerate` endpoint can rotate a key and
change its parameters in one operation. `grace_period` keeps old and new
strings valid during cutover. Omitting it or sending an empty value revokes
the old string immediately.

Set both `temp_budget_increase` and `temp_budget_expiry` through `/key/update`
to grant a time-limited increase without changing the base budget.

Automatic rotation is disabled by default. Set
`LITELLM_KEY_ROTATION_ENABLED` to enable it. The job checks every 86,400
seconds and uses a 600-second distributed lock by default.
`LITELLM_KEY_ROTATION_GRACE_PERIOD` accepts a duration such as `24h`; an empty
value revokes the old key immediately.

## Team, key, and CLI administration

Organization admins can update teams with `PATCH /team/{team_id}` using JSON
merge-patch semantics. `GET /key/list` accepts an `expires` filter (since
1.93.0).

Use `lite auth print-token` to supply tokens to Claude Code's `apiKeyHelper`.
The Microsoft Graph endpoint used by the authentication flow is configurable
for GCC High environments.
