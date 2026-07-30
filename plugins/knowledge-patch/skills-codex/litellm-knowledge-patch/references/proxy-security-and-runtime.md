# Proxy security and runtime controls

## Authentication and cache behavior

Set `litellm_settings.enable_redis_auth_cache` to share virtual-key
authentication payloads across workers through the response-cache Redis
client. This requires `cache: true` and `cache_params.type: redis`.
`general_settings.user_api_key_cache_ttl` can align the in-memory and Redis
cache TTLs.

When a JWT has no team claims, Proxy authentication falls back to the user's
database-backed team memberships (since 1.93.0).

`custom_auth_run_common_checks` defaults to `false`. Enable it to run Proxy
model allowlists, budgets, and rate limits after custom authentication.
`fail_closed_budget_enforcement` also defaults off. When enabled, every
budgeted request is verified through the database and receives 503 if neither
Redis nor the database can establish spend.

`allow_requests_on_db_unavailable` deliberately admits requests whose virtual
key cannot be checked. Restrict that fail-open design to controlled
private-network deployments.

## URL, request, and upload validation

`litellm_settings.user_url_validation` defaults to `true`. It rejects fetched
URLs whose DNS answer is private, loopback, link-local, or otherwise
non-global. Entries in `user_url_allowed_hosts` must exactly match the URL
hostname, including a port. For split-horizon DNS, allowlist the public
hostname rather than the resolved private address.

Set `enable_json_schema_validation` to validate all requests. Set
`enable_key_alias_format_validation` to require 2–255-character aliases that
start and end with an alphanumeric character and otherwise contain only
`a-zA-Z0-9_-/.@`.

Set `require_managed_files: true` to make `POST /v1/files` require
`target_model_names`; classic provider uploads then fail with 400.

## Identity and tenant isolation

From v1.95.0-rc.1, `overwrite_user_with_key_hash: true` replaces the outgoing
Chat Completions `user` value with the virtual key's SHA-256 hash, or a fixed
alias for the master key. It overrides a client value. Custom-auth and JWT
requests are unchanged.

Responses IDs are bound to user information by default.
`disable_responses_id_security` removes this cross-user protection.
Non-admin `/spend/keys` and `/spend/users` responses are caller-scoped by
default; `legacy_unscoped_spend_list_endpoints` restores the global view.
Set `reject_clientside_metadata_tags` to prevent request metadata tags from
altering budget attribution.

## Request handling

`max_request_size_mb` rejects oversized requests, while
`max_response_size_mb` prevents an oversized model response from being sent.
`pass_through_request_timeout` independently bounds custom and native-provider
pass-through calls and defaults to 600 seconds. An endpoint-specific timeout
takes precedence.

Set `cancel_on_disconnect: true` to cancel a non-streaming upstream request
when its client disconnects. LiteLLM records the cancellation as status 499.

## Database topology and convergence

Set `DATABASE_URL_READ_REPLICA` to send read-only Prisma operations to a
reader while writes continue through `DATABASE_URL`. With
`IAM_TOKEN_DB_AUTH=true`, LiteLLM refreshes tokens for both connections.

`database_disable_prepared_statements` adds `pgbouncer=true`; values in
`database_extra_connection_params` take precedence. Use
`supported_db_objects` to constrain which stored object classes load, and
`proxy_config_reload_interval_seconds` to control cross-pod refresh from the
database. The reload interval defaults to 30 seconds.

## Outbound HTTP

The aiohttp transport ignores `HTTP_PROXY` and `HTTPS_PROXY` by default. Set
`AIOHTTP_TRUST_ENV=true` to honor them. Connector limits default to unlimited
(`0`). Socket keepalive is disabled unless `AIOHTTP_SO_KEEPALIVE` is enabled;
its idle time, interval, and probe-count defaults are 60 seconds, 30 seconds,
and 5 probes.

## Hardening

HSTS is opt-in through `LITELLM_ENABLE_HSTS` and applies only over HTTPS.
`DISABLE_ADMIN_UI`, `NO_DOCS`, `NO_OPENAPI`, and `NO_REDOC` independently
remove public interfaces.

Secret redaction is enabled by default. Do not set
`LITELLM_DISABLE_REDACT_SECRETS=true` unless exposure is intentional.
`LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS` limits file-backed OIDC tokens and
defaults to `/var/run/secrets,/run/secrets`.
