---
name: model-context-protocol-knowledge-patch
description: Model Context Protocol (MCP)
version: 2026-07-28 RC
license: MIT
metadata:
  author: Nevaberry
---


# Model Context Protocol

Use this skill when implementing, migrating, or reviewing MCP clients, servers,
transports, authorization, or SDK integrations. Establish the negotiated
protocol revision and SDK major version before applying advice: several
features changed shape or disappeared across revisions.

## Reference index

| Reference | Topics |
|---|---|
| [authorization.md](references/authorization.md) | OAuth protected-resource discovery, registration, scopes, issuer binding, and SDK auth migration |
| [interactive-operations.md](references/interactive-operations.md) | Elicitation, sampling, tasks, tool results, completions, metadata, and multi-round trips |
| [protocol-revisions-and-schemas.md](references/protocol-revisions-and-schemas.md) | Era negotiation, discovery, lifecycle, removed operations, schemas, caching, tracing, and errors |
| [transport-sessions-and-subscriptions.md](references/transport-sessions-and-subscriptions.md) | Streamable HTTP, sessions, SSE, cancellation, routing headers, subscriptions, and security |
| [typescript-sdk-v2.md](references/typescript-sdk-v2.md) | TypeScript v2 packages, migration, handlers, transports, validation, caching, and modern-era APIs |
| [python-sdk-v2.md](references/python-sdk-v2.md) | Python v2 dependencies, models, servers, clients, transports, OAuth, tracing, and subscriptions |

## Start with the protocol era

There are two materially different protocol shapes:

- The 2025 era uses `initialize` followed by
  `notifications/initialized`, may use protocol-level HTTP sessions, and
  supports server-to-client requests on a live back-channel.
- The modern era uses `server/discover`, carries protocol version and client
  capabilities in every request's `_meta`, has no protocol session or
  initialization handshake, and represents back-channel work as explicit
  multi-round results.

Do not infer the modern era merely from installing an SDK v2. TypeScript
clients require `versionNegotiation`; Python's high-level client defaults to
automatic probing, but can still select a legacy connection.

On modern requests:

- Require `resultType` on the wire. Ordinary results use `"complete"` and
  embedded client work uses `"input_required"`.
- Expect the SDK to consume wire-only fields before application handlers run.
- Return `InputRequiredResult` and accept `inputResponses` on a retry instead
  of pushing elicitation, sampling, or roots requests.
- Preserve prior answers and flow phase in integrity-protected
  `requestState`; each retry carries only that round's responses.
- Treat missing `resultType` as complete only when reading an older peer.

## Breaking protocol changes

### Sessions and lifecycle

Modern MCP removes `Mcp-Session-Id`, `initialize`, and
`notifications/initialized`. Cross-call state must be explicit in tool
arguments or application handles. Do not key state by a transport connection
when implementing modern behavior.

Legacy implementations still must perform the lifecycle operation. If an HTTP
server creates a legacy session, clients repeat its session ID on every later
request and reinitialize after an expired-session 404.

### Batching

Top-level JSON-RPC batching appeared in the 2025-03-26 revision and was removed
in 2025-06-18. Send one JSON-RPC message per Streamable HTTP POST. Never use a
top-level array when targeting the later revision.

### Back-channel operations

Modern servers do not directly issue `roots/list`,
`sampling/createMessage`, or `elicitation/create`. They return an input-required
result and the client retries the original request with answers.

The core 2025 Tasks experiment was redesigned as the
`io.modelcontextprotocol/tasks` extension. Modern tasks remove `tasks/list` and
`tasks/result`, poll through `tasks/get`, and deliver client input through
`tasks/update`.

### Removed and deprecated operations

Modern MCP removes `ping`, `logging/setLevel`,
`notifications/roots/list_changed`, HTTP GET event streams,
`resources/subscribe`, and `resources/unsubscribe`.

Roots, Sampling, Logging, HTTP+SSE, and non-`none` Sampling
`includeContext` values are deprecated. Prefer tool parameters or resource
URIs, provider APIs, stderr or OpenTelemetry, and Streamable HTTP.

### Stream recovery

Modern Streamable HTTP has no SSE event IDs, `Last-Event-ID`, or redelivery.
A broken response stream loses the in-flight request; retry with a new
JSON-RPC request ID.

Older pollable SSE streams behave differently: reconnect with GET and
`Last-Event-ID`, honor the server's SSE `retry`, and remember that dropping a
stream does not cancel its request.

## Streamable HTTP essentials

Use one MCP endpoint for POST requests and, for older revisions only, an
optional GET event stream. A request-bearing POST can return either JSON or an
SSE stream; clients must accept both.

For 2025-era POSTs:

```http
Accept: application/json, text/event-stream
Content-Type: application/json
MCP-Protocol-Version: 2025-11-25
```

For modern POSTs add routing headers:

```http
Mcp-Method: tools/call
Mcp-Name: weather
```

Modern tool arguments marked `x-mcp-header` can also be mirrored into
`Mcp-Param-*` headers. Servers cross-check those values against the body and
return header-mismatch error `-32020` on disagreement.

Validate every present `Origin`. Reject invalid origins with HTTP 403, bind
local services to loopback, authenticate connections, and configure an
explicit allowlist for browser origins. Opaque `Origin: null` is not trusted.

## Authorization essentials

Treat an authorized MCP endpoint as an OAuth protected resource:

1. Read its RFC 9728 metadata from the 401 challenge or well-known location.
2. Select an advertised authorization server and read its metadata.
3. Include the canonical MCP resource URI in every authorization and token
   request.
4. Send the bearer token on every HTTP request, never in the query string.
5. Reject tokens issued for a different resource and never forward the
   inbound MCP token to an upstream API.

All clients use PKCE. Use 401 for absent, invalid, or expired credentials and
403 plus an `insufficient_scope` challenge for a valid token lacking scope.
On step-up, reauthorize with the challenged scope and retry with a small bound.

Prefer Client ID Metadata Documents when supported. Dynamic Client
Registration is a compatibility fallback and should set `application_type`.
Validate an authorization response's `iss`, bind persisted credentials to that
issuer, and re-register if the authorization server changes.

## Tool and schema essentials

Use JSON Schema 2020-12 by default. Modern tool input and output schemas may
use the full dialect, including `$ref` and composition keywords; validators
must impose resource bounds.

A tool with `outputSchema` returns matching JSON in `structuredContent`.
For older clients, also serialize the same value into a text content item.
Tool results may include `resource_link` entries even when their URIs are
absent from `resources/list`.

Return argument validation failures as tool execution errors that the caller
can inspect and correct. Reserve top-level protocol errors for malformed RPC or
deliberate wire failures.

Tool annotations can describe read-only or destructive behavior. Presentation
metadata includes human-readable `title`, icons on tools/resources/templates/
prompts, and implementation `description`; protocol dispatch continues to use
the programmatic `name`.

Modern cacheable list/read results carry `ttlMs` and `cacheScope`. Keep
`tools/list` deterministic, do not assume missing hints imply caching, and
partition private caches appropriately.

## Elicitation and sampling

Negotiate form and URL elicitation modes explicitly. Form schemas are flat and
support primitive fields, titled single-select choices, and titled
multi-select string arrays. Do not request secrets in a form.

URL elicitation is for sensitive or third-party interaction outside the
client. In modern multi-round flows it has neither `elicitationId` nor a
completion notification; retry the original request and use application
`requestState` when correlation is needed.

For 2025 tool-enabled sampling, advertise `sampling: {tools: {}}`. Every
assistant `tool_use` must be followed immediately by exactly one matching
`tool_result`, and a tool-result message contains only tool results.

## TypeScript v2 migration priorities

- Run `@modelcontextprotocol/codemod` at each package root, inspect every
  `@mcp-codemod-error`, then format and test.
- Replace the monolithic package with client, server, and core packages; use
  runtime adapters and Node.js 20 or newer.
- Keep v1 and v2 objects on opposite process or transport boundaries during a
  staged migration.
- Replace variadic registration with `registerTool`, `registerPrompt`, and
  `registerResource`; use Standard Schema and Zod 4.2 or newer.
- Select `createMcpHandler()` or `serveStdio()` for revision-aware serving.
  Direct `Server.connect()` remains legacy-era.
- Configure `versionNegotiation` when a TypeScript client must probe or pin the
  modern era.

Consult [typescript-sdk-v2.md](references/typescript-sdk-v2.md) before changing
imports, handlers, errors, authentication, validators, transports, caching, or
tests.

## Python v2 migration priorities

- Install an explicit prerelease `mcp>=2,<3` version and let it pin
  `mcp-types`; unqualified installation still selects stable v1 during the
  release-candidate period.
- Replace all SDK-facing `httpx` and `httpx-sse` values with `httpx2`; the
  types are not interchangeable.
- Rename `FastMCP` to `MCPServer`, inject `Context` explicitly, and move
  transport options to `run()` or the app factory.
- Use snake_case Python attributes and `model_dump(by_alias=True, mode="json")`
  for self-produced wire data.
- Replace low-level decorators with constructor `on_*` callbacks returning
  complete protocol result objects.
- Use `Resolve(...)` for dependencies and multi-round interactive inputs that
  must work across both protocol eras.

Consult [python-sdk-v2.md](references/python-sdk-v2.md) before changing
dependencies, protocol models, server construction, client calls, OAuth,
transport behavior, or low-level handlers.
