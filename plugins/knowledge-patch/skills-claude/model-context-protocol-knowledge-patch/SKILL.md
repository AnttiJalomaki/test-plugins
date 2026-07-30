---
name: model-context-protocol-knowledge-patch
description: Model Context Protocol (MCP)
version: 2026-07-28 RC
license: MIT
metadata:
  author: Nevaberry
---


# Model Context Protocol

Use this skill when designing, migrating, debugging, or reviewing MCP clients,
servers, transports, authorization, or the TypeScript and Python SDKs.

First identify all three compatibility dimensions:

1. The negotiated protocol revision and whether the connection uses the legacy
   handshake era or the modern discovery era.
2. The SDK major version and runtime-specific adapter.
3. The peer's advertised capabilities and the transport actually in use.

Do not infer one dimension from another. An SDK can support several protocol
revisions, and modern behavior can require an explicit negotiation option or
entry point.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/protocol-core.md](references/protocol-core.md) | Lifecycle, discovery, message framing, schemas, metadata, caching, errors, and deprecations |
| [references/http-and-subscriptions.md](references/http-and-subscriptions.md) | Streamable HTTP, sessions, SSE, cancellation, routing headers, security, and subscriptions |
| [references/authorization.md](references/authorization.md) | OAuth profile, discovery, registration, token binding, scopes, issuer validation, and redirect safety |
| [references/interaction-patterns.md](references/interaction-patterns.md) | Elicitation, sampling, structured tool results, tasks, and multi-round requests |
| [references/typescript-sdk-v2.md](references/typescript-sdk-v2.md) | TypeScript package split, codemod, handlers, transports, errors, negotiation, and modern serving |
| [references/python-sdk-v2.md](references/python-sdk-v2.md) | Python dependencies, renamed APIs, server/client migration, transports, errors, OAuth, and subscriptions |

## Breaking changes first

### Separate the legacy and modern protocol eras

The modern revision is stateless at protocol level:

- call `server/discover` to select a supported revision;
- omit `initialize`, `notifications/initialized`, and `Mcp-Session-Id`;
- carry protocol version, client capabilities, and preferably client identity in
  request `_meta`;
- return server identity and a required result discriminator;
- pass application state explicitly, such as a handle in tool arguments.

Legacy initialization and session behavior remain relevant when a client or
server negotiates a 2025 revision. Never send methods from one era after the
connection has selected the other.

### Do not batch current MCP JSON-RPC messages

The 2025-03-26 protocol briefly allowed a top-level JSON-RPC array. The
2025-06-18 revision removed batching, and Streamable HTTP accepts one request,
notification, or response per POST. Send each message separately unless
interoperating with a peer explicitly pinned to the earlier revision.

### Replace server-to-client pushes with multi-round results

Modern servers do not directly issue elicitation, sampling, roots, or ping
requests. Return an input-required result containing named `inputRequests`;
the client fulfills them and retries the original operation with
`inputResponses`.

Carry prior answers and flow phase in integrity-protected `requestState`.
Treat that state as untrusted, bind it to principal, operation, and expiry, and
remember that signing does not encrypt it.

### Rebuild subscriptions around `subscriptions/listen`

Modern change notifications use a long-lived POST response opened by
`subscriptions/listen`. The older GET event stream and
`resources/subscribe`/`resources/unsubscribe` do not carry forward.
Subscription notifications carry a subscription ID; request-scoped progress
and logging stay on the response stream for their request.

### Treat 2025 tasks as legacy interop

The experimental 2025 task shape is not the modern task design. Modern tasks
are an official extension, omit `tasks/list` and `tasks/result`, poll with
`tasks/get`, and supply client input with `tasks/update`. SDK v2 task helpers
may be removed even where explicit custom-method wire interop remains possible.

## TypeScript v2 migration

Run the codemod from each package root, then inspect every
`@mcp-codemod-error` marker and format the result:

```bash
npx @modelcontextprotocol/codemod@latest v1-to-v2 .
grep -rn '@mcp-codemod-error' .
```

Use Node.js 20 or newer. Replace the monolithic package with the client,
server, core schema, and runtime adapter packages that the code actually uses.
Never import the unpublished internal core package, and do not pass v1 class
instances or nominal types into v2 code.

Replace variadic `.tool()`, `.prompt()`, and `.resource()` registration with
`registerTool`, `registerPrompt`, and `registerResource`. Use Standard Schema
objects—Zod 4.2 or newer is supported—and register low-level handlers by
method string. Request details now live under `ctx.mcpReq`; HTTP-only state is
under `ctx.http`.

Choose the serving entry deliberately:

- `createMcpHandler()` for per-request HTTP serving;
- `serveStdio()` to select an era for a stdio connection;
- a Node adapter for Node request/response APIs;
- the Web-standard transport for fetch-shaped runtimes.

A plain client still uses the legacy handshake unless `versionNegotiation` is
configured. `auto` probes discovery and performs only allowed fallback;
pinning the modern revision fails closed when negotiation cannot succeed.

Read [references/typescript-sdk-v2.md](references/typescript-sdk-v2.md) before
changing imports, registration, error handling, authorization, tests, cache
stores, or transport setup.

## Python v2 migration

The Python v2 line is prerelease. Pin an explicit v2 prerelease or use
`mcp>=2,<3`, allow it to exact-pin `mcp-types`, and update raised dependency
floors. Libraries that require v1 behavior should cap below v2.

Replace:

- `httpx` and `httpx-sse` with the non-interchangeable `httpx2` types;
- `FastMCP` with `MCPServer`;
- ambient `get_context()` access with an injected `Context`;
- low-level decorators with constructor `on_*` callbacks;
- `McpError(ErrorData(...))` with `MCPError(code, message, data)`;
- camel-case Python attributes with snake-case attributes, while preserving
  wire aliases during serialization;
- RootModel-style message unions with concrete members and exported adapters.

Use `model_dump(by_alias=True, mode="json")` for wire JSON. Put extensions in
`_meta`, because unknown protocol-model fields are discarded.

The high-level `Client` defaults to automatic era negotiation. On a modern
connection, push-style back-channel calls fail; use resolver-driven
input-required flows or pin legacy behavior temporarily. Read
[references/python-sdk-v2.md](references/python-sdk-v2.md) before changing
dependencies, resource handling, timeouts, OAuth, stdio, or HTTP clients.

## Authorization quick reference

For authorized HTTP servers:

- use OAuth 2.1 and PKCE; stdio credentials come from the environment;
- discover protected-resource metadata, then authorization-server metadata;
- include the canonical MCP resource URI in authorization and token requests;
- send the bearer token on every HTTP request and never in a query string;
- validate token audience/resource and never forward the inbound token to an
  upstream API;
- validate authorization-response issuer and keep stored credentials separated
  by issuer;
- prefer Client ID Metadata Documents, with dynamic registration only as a
  compatibility fallback;
- handle 401 for missing or invalid authorization and 403 plus an
  `insufficient_scope` challenge for step-up authorization.

See [references/authorization.md](references/authorization.md) for discovery
ordering, exact URLs, registration requirements, and SDK-specific safety rules.

## Transport and security quick reference

For a legacy Streamable HTTP request, send `Accept: application/json,
text/event-stream`; support either a single JSON response or an SSE response.
After initialization, send the negotiated `MCP-Protocol-Version`.

For modern HTTP, send the routing headers required for the method and tool-like
name. Schema properties marked `x-mcp-header` mirror selected arguments into
`Mcp-Param-*`, and the server must reject header/body mismatches.

Validate every present `Origin`. Local servers should bind to loopback, protect
against DNS rebinding, authenticate connections, and return HTTP 403 for an
invalid origin. Use HTTPS for non-loopback authorization and redirect
endpoints.

Do not apply old SSE retry assumptions to modern response streams. A broken
modern stream loses that in-flight request; retry with a new request ID.

## Results, schemas, and errors

Use JSON Schema 2020-12 by default. Tool input and output schemas can use the
full dialect, so resolve references and enforce resource limits for composition
keywords. Validate `structuredContent` against declared output schemas and
retain a text serialization when older clients need compatibility.

Return tool argument validation failures as tool execution errors where the
caller can correct the input. Reserve protocol errors for protocol failures.
In the modern allocation:

- `-32020` is header mismatch;
- `-32021` is missing required client capability;
- `-32022` is unsupported protocol version;
- implementation-defined errors use `-32000` through `-32019`;
- MCP-reserved errors use `-32020` through `-32099`.

Cache only when the negotiated revision and result shape support it. Honor
`ttlMs` and `cacheScope`, partition private caches correctly, and never treat
self-reported server identity as a security decision.

## Implementation checklist

1. Pin or negotiate the protocol revision before choosing methods and codecs.
2. Confirm advertised capabilities before elicitation, sampling, completion,
   tasks, list calls, or subscriptions.
3. Validate transport headers, content type, origin, authorization resource,
   and issuer at the HTTP boundary.
4. Keep request cancellation, progress, logging, and subscription traffic on
   the stream defined by the selected era.
5. Validate tool inputs and structured outputs with the selected JSON Schema
   dialect.
6. Test the actual runtime entry point; in-memory legacy transports do not
   prove modern HTTP behavior.
7. Exercise stale-cache, step-up authorization, broken-stream, and
   multi-round retry paths explicitly.
