# MCP, agents, and guardrails

## A2A agent gateway

Add and invoke A2A agents through the same gateway that serves model and MCP
routes. A deployment does not need a separate agent gateway.

## Client-held MCP credentials

MCP server configuration supports `true_passthrough` and `oauth_delegate`
authentication modes, with upstream OAuth discovery bound to each server
(since 1.93.0).

Use the `dcr_bridge` path to carry client-held credentials in a sealed
envelope. It exposes discovery and registration/token relays and requires
PKCE S256.

## OAuth token exchange

Set the MCP server `auth_type` to `oauth2_token_exchange` and select the
`entra_obo` token-exchange profile for on-behalf-of calls. The REST API and
dashboard support this configuration. Persist the selected `oauth2_flow`
explicitly; startup backfills older missing values. Outbound concurrency
limits also apply to on-behalf-of tool calls.

## Semantic filtering

The MCP semantic filter expands `litellm_proxy` tools before it filters them.
It reports the number of tools removed and preserves full tool names in its
response header. Context-window failures are surfaced and fail closed rather
than bypassing the filter.

## Guardrails

Model Armor supports MCP tool-call scanning in `pre_mcp_call` and
`during_mcp_call` modes. Content Filter supports `pre_mcp_call`.

With `skip_unscannable_attachments`, Model Armor passes reference-only
attachments through and does not apply an attachment-count cap.

## Ingress origin and trusted proxies

For MCP OAuth behind ingress, set `PROXY_BASE_URL` to the exact public origin
without a path or trailing slash. It takes precedence over forwarded headers.

Without `PROXY_BASE_URL`, `use_x_forwarded_for` is honored only when the
immediate peer belongs to `mcp_trusted_proxy_ranges`.

## Explicit MCP grants

Set `require_key_mcp_access_defined` to prevent an empty virtual-key grant
from inheriting the team's MCP servers. Set
`require_end_user_mcp_access_defined` to require an explicit end-user grant.
