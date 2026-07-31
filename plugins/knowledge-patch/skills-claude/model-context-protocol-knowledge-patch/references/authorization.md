# Authorization, Registration, and Security

## Transport authorization profile (`2025-03-26-compat`)

Authorization is optional. HTTP transports that implement it use OAuth 2.1;
stdio deployments obtain credentials from the environment instead. Require
PKCE for every client. Authorization-required and invalid-token responses use
HTTP 401. Use authorization-code grants for user authorization or
client-credentials grants for application authorization as appropriate.

Send `Authorization: Bearer <access-token>` on every HTTP request, including
requests made after an MCP session is established. Never put the token in the
query string. Invalid or expired tokens receive 401; insufficient scope
receives 403.

When an MCP server delegates authorization to a third-party authorization
server, issue an MCP-specific token bound to the upstream session. Keep the
upstream and MCP token validity and lifecycle synchronized; do not expose the
upstream token to the client.

## Original authorization discovery and registration (`2025-03-26-compat`)

Clients try RFC 8414 authorization-server metadata and should include
`MCP-Protocol-Version`. For this discovery profile, derive the authorization
origin by discarding the entire path of the MCP URL:

```text
MCP URL:  https://api.example.com/v1/mcp
Metadata: https://api.example.com/.well-known/oauth-authorization-server
Fallback: https://api.example.com/authorize
          https://api.example.com/token
          https://api.example.com/register
```

If discovery fails, use `/authorize`, `/token`, and `/register` at that origin.
Dynamic Client Registration was the recommended first choice in this profile;
a server-specific client ID or user-entered registration details were the
fallback. Later revisions refine both discovery and registration preference.

## Protected-resource discovery and audience binding (`2025-06-18-compat`)

Treat an authorized MCP server as an OAuth protected resource. It publishes
RFC 9728 metadata containing at least one `authorization_servers` entry and
points to that document from a 401 `WWW-Authenticate` header. The client parses
the metadata, selects an authorization server when several are advertised, and
then reads that server's RFC 8414 metadata.

Every authorization and token request includes the RFC 8707 `resource`
parameter even if the authorization server ignores it:

```text
resource=https%3A%2F%2Fmcp.example.com%2Fserver%2Fmcp
```

Choose the most specific canonical absolute MCP URI. Preserve a path that
distinguishes this server from other resources and omit fragments. The MCP
server rejects a token not issued for that resource. It must never pass the
inbound token through to an upstream API.

## Client ID Metadata Documents (`2025-11-25-compat`)

For a client and authorization server without a prior relationship, perform
pre-registration and then use a Client ID Metadata Document when
`client_id_metadata_document_supported` is true. Fall back first to optional
Dynamic Client Registration and then to user-entered credentials.

The `client_id` is an HTTPS URL containing a path. That URL serves JSON whose
`client_id` exactly matches the URL and which also declares `client_name` and
`redirect_uris`:

```json
{
  "client_id": "https://app.example.com/oauth/client.json",
  "client_name": "Example MCP Client",
  "redirect_uris": ["http://127.0.0.1:3000/callback"]
}
```

An authorization server that supports this mechanism fetches and validates
the document and verifies that the authorization request's redirect URI is one
of its declared values.

## Current discovery fallback order (`2025-11-25-compat`)

Support a `resource_metadata` URL in the 401 bearer challenge as well as
protected-resource well-known discovery. If the challenge omits the URL, try
the MCP-path form first and then the origin root:

```text
https://mcp.example.com/.well-known/oauth-protected-resource/public/mcp
https://mcp.example.com/.well-known/oauth-protected-resource
```

When the authorization-server issuer contains a path, try these metadata forms
in order: OAuth path insertion, OIDC path insertion, then OIDC path appending.

```text
https://auth.example.com/.well-known/oauth-authorization-server/tenant1
https://auth.example.com/.well-known/openid-configuration/tenant1
https://auth.example.com/tenant1/.well-known/openid-configuration
```

## Scope selection and step-up (`2025-11-25-compat`)

For initial authorization, request the 401 challenge's `scope` when present.
Otherwise request every `scopes_supported` value from protected-resource
metadata. Omit `scope` if that metadata field is absent. A scope set supplied
by a challenge is authoritative for its request even when it differs from
`scopes_supported`.

For an operation that needs more permission, return HTTP 403 with a bearer
challenge containing `error="insufficient_scope"`, the required `scope`, and
`resource_metadata`. A user-facing client reauthorizes for the increased scope
set and retries the original operation. Apply a small retry limit so repeated
challenges cannot create an authorization loop.

## HTTP endpoint security

Validate `Origin` on every incoming Streamable HTTP connection to prevent DNS
rebinding. Bind local servers to `127.0.0.1`, not `0.0.0.0`; authenticate all
connections; serve authorization endpoints over HTTPS; and accept only
localhost or HTTPS redirect URIs. Since `2025-11-25`, rejecting an invalid
`Origin` specifically returns HTTP 403 Forbidden.
