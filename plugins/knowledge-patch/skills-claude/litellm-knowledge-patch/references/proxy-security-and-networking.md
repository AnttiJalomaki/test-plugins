# Proxy security and networking

## URL, request, and upload validation

`litellm_settings.user_url_validation` defaults to `true`. It rejects fetched URLs whose DNS answer is private, loopback, link-local, or otherwise non-global. `user_url_allowed_hosts` must match the hostname exactly as written in the URL, including a port. With split-horizon DNS, allowlist the public hostname rather than its private answer.

`enable_json_schema_validation` validates all requests. `enable_key_alias_format_validation` restricts aliases to 2–255 characters, requires alphanumeric first and last characters, and permits only `a-zA-Z0-9_-/.@` between them. `require_managed_files: true` makes `POST /v1/files` require `target_model_names` and reject classic provider uploads with 400.

Set `max_request_size_mb` to reject oversized requests and `max_response_size_mb` to withhold oversized model responses. `pass_through_request_timeout` separately bounds custom and native-provider pass-through requests at 600 seconds by default; an endpoint-specific timeout wins.

## Authentication and tenant isolation

`custom_auth_run_common_checks` defaults to `false`; enable it to run model allowlists, budget checks, and rate limits after custom authentication.

Responses IDs are bound to user information by default. `disable_responses_id_security` removes this cross-user protection. Non-admin `/spend/keys` and `/spend/users` results are caller-scoped by default; `legacy_unscoped_spend_list_endpoints` restores the old global view. `reject_clientside_metadata_tags` stops request-supplied tags from changing budget attribution.

`fail_closed_budget_enforcement` checks every budgeted request against the database and returns 503 if neither Redis nor the database can establish spend. `allow_requests_on_db_unavailable` deliberately permits unchecked virtual-key requests and is intended only for private-network deployments.

## MCP public origins, proxies, and grants

For MCP OAuth behind ingress, set `PROXY_BASE_URL` to the exact public origin, with no path or trailing slash. It takes precedence over forwarded headers. Without it, `use_x_forwarded_for` is honored only when the immediate peer lies inside `mcp_trusted_proxy_ranges`.

`require_key_mcp_access_defined` prevents an empty key grant from inheriting the team's MCP servers. `require_end_user_mcp_access_defined` requires an explicit end-user grant.

## Outbound HTTP controls

The aiohttp transport ignores `HTTP_PROXY` and `HTTPS_PROXY` by default. Set `AIOHTTP_TRUST_ENV=true` to honor them. Connector limits default to unlimited (`0`). Socket keepalive is off unless `AIOHTTP_SO_KEEPALIVE` is set; its idle, interval, and probe-count defaults are 60 seconds, 30 seconds, and 5 probes.

## Drain, disconnect, and hardening switches

`enable_drain_endpoint` exposes `GET /health/drain` for pre-stop hooks and is off by default. With no `drain_endpoint_token`, the endpoint is unauthenticated; when a token is set, callers must send the matching `X-Drain-Token`.

`cancel_on_disconnect: true` cancels a non-streaming upstream request after its client leaves and records the cancellation as 499.

HSTS is opt-in through `LITELLM_ENABLE_HSTS` and applies only when served over HTTPS. `DISABLE_ADMIN_UI`, `NO_DOCS`, `NO_OPENAPI`, and `NO_REDOC` remove exposed interfaces independently. Log secret redaction is enabled unless `LITELLM_DISABLE_REDACT_SECRETS=true`. `LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS` restricts file-backed OIDC tokens; the default roots are `/var/run/secrets,/run/secrets`.

## MCP-aware content inspection

Model Armor can scan MCP tool calls in `pre_mcp_call` and `during_mcp_call` modes. Content Filter supports `pre_mcp_call`. With `skip_unscannable_attachments`, Model Armor passes through reference-only attachments and does not impose an attachment-count cap.
