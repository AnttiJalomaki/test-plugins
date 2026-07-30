# Authorization

## HTTP authorization profile

Authorization is optional. When an HTTP transport implements it, use OAuth
2.1; stdio implementations should obtain credentials from the environment.
PKCE is required for every client. Servers may use authorization-code grants
for users or client-credentials grants for applications.

Send `Authorization: Bearer <access-token>` on every HTTP request, including
requests inside an established MCP session. Never place access tokens in query
strings. Use:

- HTTP 401 when authorization is required or a token is invalid or expired.
- HTTP 403 when the token is valid but lacks the required scope.

An MCP server that delegates to a third-party authorization server must issue
its own token bound to the upstream session, synchronize their validity and
lifecycle, and never pass the inbound token through to the upstream API.

The earliest HTTP authorization profile used RFC 8414 discovery at the
authorization origin, discarding the MCP URL's full path. It recommended
dynamic registration and fell back to `/authorize`, `/token`, and `/register`
at that origin. Keep this behavior only for peers targeting
`2025-03-26-compat`; later protected-resource discovery takes precedence.

```text
MCP URL:  https://api.example.com/v1/mcp
Metadata: https://api.example.com/.well-known/oauth-authorization-server
Fallback: https://api.example.com/authorize
          https://api.example.com/token
          https://api.example.com/register
```

Discovery requests should send `MCP-Protocol-Version`.

## Protected-resource discovery

Since `2025-06-18-compat`, an authorized MCP server is an OAuth protected
resource. It must publish RFC 9728 metadata with at least one
`authorization_servers` entry and advertise the metadata URL in the 401
`WWW-Authenticate` challenge. The client parses the protected-resource
metadata, chooses an authorization server when several are offered, and then
loads that server's RFC 8414 metadata.

Clients must support both `resource_metadata` in the challenge and well-known
protected-resource discovery. Since `2025-11-25-compat`, if the challenge omits
the URL, try:

1. The well-known URI that retains the MCP resource path.
2. The well-known URI at the origin root.

```text
https://mcp.example.com/.well-known/oauth-protected-resource/public/mcp
https://mcp.example.com/.well-known/oauth-protected-resource
```

For an authorization-server issuer containing a path, try these metadata
forms in order:

```text
https://auth.example.com/.well-known/oauth-authorization-server/tenant1
https://auth.example.com/.well-known/openid-configuration/tenant1
https://auth.example.com/tenant1/.well-known/openid-configuration
```

## Resource indicators and token binding

Include the RFC 8707 `resource` parameter in every authorization and token
request, even when the selected authorization server does not advertise
support. Use the most specific canonical absolute MCP URI:

- Include a distinguishing path when needed.
- Do not include a fragment.
- Preserve exact protected-resource and issuer strings.

```text
resource=https%3A%2F%2Fmcp.example.com%2Fserver%2Fmcp
```

The MCP server must reject a token not issued for its resource and must not
forward that token upstream.

## Client registration

For a client and authorization server without a prior relationship, use this
order:

1. Pre-registered client information, when available.
2. A Client ID Metadata Document when
   `client_id_metadata_document_supported` is true.
3. Dynamic Client Registration as a compatibility fallback.
4. User-entered credentials.

A Client ID Metadata Document uses an HTTPS URL with a path as the `client_id`.
The JSON at that URL must contain an exactly matching `client_id`, plus
`client_name` and `redirect_uris`. Supporting authorization servers should
fetch the document and must validate it and the requested redirect URI.

```json
{
  "client_id": "https://app.example.com/oauth/client.json",
  "client_name": "Example MCP Client",
  "redirect_uris": ["http://127.0.0.1:3000/callback"]
}
```

Dynamic Client Registration is deprecated in the modern RC and remains only
for authorization servers that do not support Client ID Metadata Documents.
Its request must set a suitable `application_type` to avoid OpenID Connect
redirect-URI conflicts.

## Scope selection and step-up

For initial authorization:

- Use the 401 challenge's `scope` when present.
- Otherwise request every entry in protected-resource metadata's
  `scopes_supported`.
- Omit `scope` when `scopes_supported` is absent.

A challenged scope set is authoritative for that request, whether or not it is
a subset of `scopes_supported`.

For insufficient runtime permission, return HTTP 403 with a Bearer challenge
containing `error="insufficient_scope"`, the required `scope`, and
`resource_metadata`. A user-facing client should reauthorize with the expanded
scope and retry the original operation, with a small retry limit.

## Authorization response issuer binding

Authorization servers should include `iss` in authorization responses. Before
the code exchange, a client must compare any returned issuer with the issuer
recorded for the flow.

Persisted client credentials must be keyed by issuer and never reused with a
different authorization server. Re-register when the authorization server
changes.

## TypeScript SDK v2 authorization migration

### Packages and server responsibilities

Express resource-server helpers live in `@modelcontextprotocol/express`;
Web-standard resource-server helpers live in `@modelcontextprotocol/server`.
Authorization-server helpers moved to the deprecated, frozen
`@modelcontextprotocol/server-legacy/auth`. Treat that package only as a
migration bridge and move authorization-server responsibilities to a dedicated
OAuth provider or library.

### Error and bearer-provider behavior

The separate OAuth exception classes are replaced by `OAuthError` and
`OAuthErrorCode`. A verifier used by v2 bearer middleware must throw the v2
error. A legacy or generic invalid-token error is treated as an internal
failure and becomes HTTP 500 rather than a 401 challenge.

The minimal `AuthProvider` supports non-OAuth bearer tokens with `token()` and
optional `onUnauthorized()`. It retries once after a 401.

### Callback and credential safety

Validate `state` in the host, then pass the callback's `URLSearchParams` to
`finishAuth()` so the SDK can validate `iss`. Non-loopback token endpoints must
use HTTPS.

Persist token and client-information objects verbatim so their `issuer` stamp
survives. In a multi-issuer store:

- Key records by `ctx.issuer`.
- When `ctx` is absent, return the most recently saved token set.
- Implement `discoveryState()` and `saveDiscoveryState()`.
- Use `onInsufficientScope` to choose whether insufficient scope reauthorizes
  by default or throws.

## Python SDK v2 authorization migration

`OAuthClientProvider.callback_handler` returns
`AuthorizationCodeResult(code, state, iss)`, enabling authorization-response
issuer validation. Client-credentials providers rename `scopes=` to `scope=`.

The removed `RFC7523OAuthClientProvider` and `JWTParameters` have no direct
replacement. Choose the provider matching the actual grant:

- `ClientCredentialsOAuthProvider`
- `PrivateKeyJWTOAuthProvider`
- `IdentityAssertionOAuthProvider`

When refresh tokens are supported and enabled, the client automatically asks
for `offline_access` and adds `prompt=consent`. Restrict `grant_types` to
`["authorization_code"]` to preserve no-refresh behavior.

Pathless OAuth metadata and redirect URLs no longer receive a trailing slash.
Re-register persisted clients if their exact redirect URI changes, and require
exact equality for protected-resource and issuer strings.
