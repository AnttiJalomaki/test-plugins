# Authorization, registration, and security

Use this reference for HTTP authorization, metadata discovery, client
registration, token validation, scopes, issuer safety, and SDK migration.

Relevant protocol attributions: `2025-03-26-compat`,
`2025-06-18-compat`, `2025-11-25-compat`, and `2026-07-28-rc`.

## Choose the authorization profile

Authorization is optional. When an HTTP transport implements it, use OAuth 2.1;
stdio implementations should obtain credentials from the environment. PKCE is
required for every client. Servers may use authorization-code grants for user
delegation or client-credentials grants for applications.

Return HTTP 401 when authorization is required or a token is invalid or
expired. Return HTTP 403 when a valid token lacks scope.

Send `Authorization: Bearer <access-token>` on every HTTP request, even during
an established legacy session. Never put an access token in the query string.

If the MCP server delegates to another service, issue an MCP access token bound
to the upstream session and synchronize the two lifecycles. Never pass the
inbound MCP token directly to an upstream API.

## Discover the protected resource

Authorized servers publish RFC 9728 protected-resource metadata containing at
least one `authorization_servers` entry. A 401 response points to it with
`resource_metadata` in `WWW-Authenticate`.

Clients must support both that challenge parameter and well-known discovery.
Without the header, try the MCP-path form before the origin root:

```text
https://mcp.example.com/.well-known/oauth-protected-resource/public/mcp
https://mcp.example.com/.well-known/oauth-protected-resource
```

If several authorization servers are advertised, let the client choose one,
then discover that issuer's authorization-server metadata.

## Discover the authorization server

For the older authorization profile, try RFC 8414 metadata and send
`MCP-Protocol-Version` where applicable. Its authorization base is the MCP
URL's origin—the entire MCP path is discarded:

```text
MCP URL:  https://api.example.com/v1/mcp
Metadata: https://api.example.com/.well-known/oauth-authorization-server
Fallback: https://api.example.com/authorize
          https://api.example.com/token
          https://api.example.com/register
```

When an issuer includes a path, use this discovery order:

1. OAuth metadata with well-known path insertion.
2. OIDC discovery with well-known path insertion.
3. OIDC discovery with well-known path appending.

```text
https://auth.example.com/.well-known/oauth-authorization-server/tenant1
https://auth.example.com/.well-known/openid-configuration/tenant1
https://auth.example.com/tenant1/.well-known/openid-configuration
```

Only fall back to the origin's `/authorize`, `/token`, and `/register` endpoints
when the metadata flow for the older profile fails.

## Bind tokens to the MCP resource

Every authorization and token request includes the RFC 8707 `resource`
parameter, even if the authorization server ignores it. Use the most specific
canonical absolute MCP URI, retain a distinguishing path, and omit fragments:

```text
resource=https%3A%2F%2Fmcp.example.com%2Fserver%2Fmcp
```

The resource server must reject a token not issued for that exact resource.
Keep protected-resource and issuer strings exact; path normalization or an
added trailing slash can change identity.

## Register clients

For clients and authorization servers without a prior relationship, prefer
Client ID Metadata Documents after pre-registration when
`client_id_metadata_document_supported` is true.

The client ID is an HTTPS URL with a path. Its JSON document must contain an
exactly matching `client_id`, plus `client_name` and `redirect_uris`:

```json
{
  "client_id": "https://app.example.com/oauth/client.json",
  "client_name": "Example MCP Client",
  "redirect_uris": ["http://127.0.0.1:3000/callback"]
}
```

An authorization server that supports this mechanism should fetch and validate
the document and the requested redirect URI.

Dynamic Client Registration is optional and deprecated. Retain it only as a
compatibility fallback, followed by server-specific or user-entered
credentials. A dynamic-registration request must set an appropriate
`application_type` to avoid OIDC redirect-URI conflicts.

## Select and increase scopes

For initial authorization, choose scopes in this order:

1. Use the 401 challenge's `scope` when present.
2. Otherwise request all `scopes_supported` from protected-resource metadata.
3. If that field is absent, omit `scope`.

A challenged scope set is authoritative for that request even when it differs
from `scopes_supported`.

For runtime step-up, return:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope", scope="files.write", resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource"
```

A user-facing client should reauthorize for the increased set, retry the
original operation, and enforce a small retry limit.

## Validate issuer and stored credentials

Authorization servers should return `iss`. Record the selected issuer before
redirecting, validate any returned `iss` before code exchange, and key
persisted client credentials by issuer. Never reuse credentials with another
authorization server; re-register when the server changes.

Non-loopback token endpoints must use HTTPS. Validate host-managed OAuth
`state` before handing callback parameters to an SDK. Accept only localhost or
HTTPS redirect URIs, and re-register a client if an SDK migration changes its
exact redirect URI by removing a trailing slash.

## TypeScript SDK v2 authorization details

- Resource-server middleware lives in the Express adapter for Express or the
  Web-standard server package for fetch-shaped hosts.
- Authorization-server helpers survive only in the deprecated frozen legacy
  auth package. Move that responsibility to a dedicated OAuth provider.
- OAuth exception classes collapse into `OAuthError` and `OAuthErrorCode`.
  Bearer middleware must receive the v2 error; a generic or legacy invalid-token
  exception can become HTTP 500 instead of a 401 challenge.
- The minimal `AuthProvider` supports arbitrary bearer tokens through
  `token()` and optional `onUnauthorized()`, with one retry after 401.
- Pass callback `URLSearchParams` to `finishAuth()` for `iss` validation after
  the host validates `state`.
- Persist token and client-information objects verbatim so the issuer stamp is
  retained. For multi-issuer stores, key by `ctx.issuer`, return the latest
  token set when context is absent, and implement
  `discoveryState()`/`saveDiscoveryState()`.
- Decide explicitly whether `onInsufficientScope` reauthorizes or throws.

## Python SDK v2 authorization details

- `OAuthClientProvider.callback_handler` returns
  `AuthorizationCodeResult(code, state, iss)` so issuer validation can run.
- Client-credentials providers rename `scopes=` to singular `scope=`.
- `RFC7523OAuthClientProvider` and `JWTParameters` are removed. Choose
  `ClientCredentialsOAuthProvider`, `PrivateKeyJWTOAuthProvider`, or
  `IdentityAssertionOAuthProvider` for the actual grant.
- When refresh tokens are enabled and supported, the client requests
  `offline_access` and adds `prompt=consent`. Restrict `grant_types` to
  `["authorization_code"]` to keep no-refresh behavior.
- Pathless metadata and redirect URLs no longer gain a trailing slash. Make
  persisted registration, resource, issuer, and redirect values match exactly.

## HTTP boundary safety

Validate `Origin` on every incoming HTTP connection to prevent DNS rebinding.
Local servers should bind to `127.0.0.1`, not `0.0.0.0`, authenticate
connections, and serve authorization endpoints over HTTPS. An invalid present
origin must receive HTTP 403.
