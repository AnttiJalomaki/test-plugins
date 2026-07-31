# Authorization

## Choose authorization by transport

For the HTTP authorization profile introduced in compatibility batch
`2025-03-26-compat`, authorization remains optional, but an HTTP transport that
implements it should use OAuth 2.1. A stdio implementation should obtain its
credentials from the environment instead.

Require PKCE for every client. Use HTTP 401 when authorization is required or a
token is invalid. A server may use authorization-code grants for human users or
client-credentials grants for applications.

## Discover authorization metadata

### Initial profile behavior

In the initial profile, clients try RFC 8414 authorization-server metadata and
should send `MCP-Protocol-Version`. The authorization base is the MCP URL's
origin: discard the entire MCP path. If discovery fails, use `/authorize`,
`/token`, and `/register` at that origin. Dynamic registration was the preferred
path, with a server-specific client ID or user-entered registration details as
fallbacks.

```text
MCP URL:  https://api.example.com/v1/mcp
Metadata: https://api.example.com/.well-known/oauth-authorization-server
Fallback: https://api.example.com/authorize
          https://api.example.com/token
          https://api.example.com/register
```

### Protected-resource discovery

Compatibility batch `2025-06-18-compat` makes an authorized MCP server an OAuth
protected resource. The server must publish RFC 9728 metadata containing at
least one `authorization_servers` entry. A 401 response points to it through
the `resource_metadata` parameter of `WWW-Authenticate`. The client parses that
document, chooses an authorization server when several are advertised, and
then loads that server's RFC 8414 metadata.

### Current fallback order

Compatibility batch `2025-11-25-compat` requires clients to support both
`resource_metadata` in the 401 challenge and protected-resource well-known
discovery. If the challenge omits `resource_metadata`, try the MCP-path form
first, then the origin root:

```text
https://mcp.example.com/.well-known/oauth-protected-resource/public/mcp
https://mcp.example.com/.well-known/oauth-protected-resource
```

For an authorization-server issuer containing a path, try these in order:

1. OAuth metadata with the issuer path inserted after the well-known name.
2. OIDC discovery with the issuer path inserted after the well-known name.
3. OIDC discovery with the well-known name appended to the issuer path.

```text
https://auth.example.com/.well-known/oauth-authorization-server/tenant1
https://auth.example.com/.well-known/openid-configuration/tenant1
https://auth.example.com/tenant1/.well-known/openid-configuration
```

Do not conflate the initial authorization-server lookup, which removes the MCP
path, with protected-resource discovery, whose path form deliberately
distinguishes MCP resources.

## Register clients

For clients and authorization servers with no prior relationship, prefer a
Client ID Metadata Document after pre-registration when
`client_id_metadata_document_supported` is true. The `client_id` must be an
HTTPS URL with a path. Its JSON document must contain:

- a `client_id` exactly equal to that URL;
- `client_name`;
- `redirect_uris`.

```json
{
  "client_id": "https://app.example.com/oauth/client.json",
  "client_name": "Example MCP Client",
  "redirect_uris": ["http://127.0.0.1:3000/callback"]
}
```

A supporting authorization server should fetch the document and must validate
both the document and the redirect URI requested by the client. Dynamic Client
Registration is optional and remains a backwards-compatible fallback;
user-entered credentials are another fallback.

## Bind tokens to the MCP resource

Every authorization and token request must include the RFC 8707 `resource`
parameter, even if the authorization server does not support it. Use the most
specific canonical absolute MCP URI. Include a distinguishing path when
needed, and never include a fragment.

```text
resource=https%3A%2F%2Fmcp.example.com%2Fserver%2Fmcp
```

The MCP server must reject a token not issued for that resource. It must not
pass the inbound token through to an upstream API.

## Send and validate access tokens

Send `Authorization: Bearer <access-token>` on every HTTP request, even after
an MCP session has been established. Never put the token in a query string.
Return 401 for an invalid or expired token and 403 for insufficient scope.

When an MCP server delegates to a third-party authorization server, it must
issue its own token bound to the upstream session. Keep the MCP token and
upstream token validity and lifecycle synchronized; do not expose the upstream
credential as the MCP access token.

## Choose scopes and perform step-up authorization

For initial authorization:

1. Use the 401 challenge's `scope` when present. It is authoritative for that
   request even when it differs from protected-resource metadata.
2. Otherwise request every value in protected-resource
   `scopes_supported`.
3. If `scopes_supported` is absent, omit `scope`.

At runtime, insufficient permission should produce HTTP 403 with a challenge
containing all of the following:

```http
WWW-Authenticate: Bearer error="insufficient_scope", scope="required scopes", resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource"
```

A user-facing client should reauthorize with the increased scope set and retry
the original operation. Apply a small retry limit to avoid an authorization
loop.
