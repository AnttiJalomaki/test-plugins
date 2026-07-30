# TypeScript SDK v2 migration and modern serving

Use this reference when migrating a TypeScript SDK v1 application, selecting v2
packages and adapters, registering handlers, negotiating protocol eras, or
serving the modern revision.

## Run and review the codemod

Run the codemod at a package root:

```bash
npx @modelcontextprotocol/codemod@latest v1-to-v2 .
grep -rn '@mcp-codemod-error' .
```

It updates fixed import, registration, mock, and `package.json` mappings. It
marks judgment calls with `@mcp-codemod-error` and does not format the result.
Review every marker, run the formatter, and then type-check and test.

The codemod edits only the nearest manifest and reports other workspace
members. Run it for each member and declare the v2 packages used by that
member's own imports.

V1 and v2 can coexist during a staged migration, but objects, nominal types,
and class instances cannot cross the boundary. Add v2 first, migrate along
process or transport boundaries, and remove v1 last.

## Install the split packages and adapters

V2 requires Node.js 20 or newer. Replace `@modelcontextprotocol/sdk` with:

- `@modelcontextprotocol/client`;
- `@modelcontextprotocol/server`;
- public Zod schemas from `@modelcontextprotocol/core`;
- the adapter package for the actual runtime or framework.

Never import unpublished `@modelcontextprotocol/core-internal`.

The packages ship native ESM and CommonJS builds. Node, Express, Hono, and
Fastify integrations live in separate packages. Express, Hono, and Fastify
adapters require their framework peer dependency; the Node adapter does not
require Hono.

Authorization-server helpers remain only in the deprecated frozen
`@modelcontextprotocol/server-legacy/auth` bridge. Move authorization-server
duties to a dedicated OAuth provider or library. See
[authorization.md](authorization.md) for resource-server packages, v2 OAuth
errors, issuer-aware storage, callback validation, and bearer providers.

## Select runtime-specific transports

Use:

- `NodeStreamableHTTPServerTransport` for Node request/response handlers;
- `WebStandardStreamableHTTPServerTransport` for Web `Request`/`Response`;
- stdio transports from their client or server package subpaths.

```typescript
import { NodeStreamableHTTPServerTransport } from '@modelcontextprotocol/node';
import { WebStandardStreamableHTTPServerTransport } from '@modelcontextprotocol/server';
import { StdioClientTransport } from '@modelcontextprotocol/client/stdio';
```

`SSEServerTransport` survives only in
`@modelcontextprotocol/server-legacy/sse` as a temporary bridge.
`WebSocketClientTransport` is removed. Both halves of an `InMemoryTransport`
pair must come from the same client or server package.

Both stdio transports accept `maxBufferSize`, defaulting to 10 MB. A message
that exceeds the limit closes the transport through `onerror`.
`ReadBuffer.readMessage()` skips non-JSON stdout lines but still rejects JSON
that fails JSON-RPC validation.

## Migrate handler context

The flat `RequestHandlerExtra` becomes `ServerContext` or `ClientContext`.
Request state, cancellation, nested requests, and notifications are under
`ctx.mcpReq`; optional HTTP state is under `ctx.http`:

```typescript
const { signal, id, _meta } = ctx.mcpReq;
await ctx.mcpReq.send(request);
await ctx.mcpReq.notify(notification);
const token = ctx.http?.req?.headers.get('authorization');
```

Closing a transport aborts in-flight handlers through `ctx.mcpReq.signal`.

Reserved protocol envelope and retry fields are lifted before a handler runs:

- `ctx.mcpReq.envelope`;
- `ctx.mcpReq.inputResponses`;
- `ctx.mcpReq.requestState()`.

This lift also applies to a 2025 custom request whose bare parameters are named
`inputResponses` or `requestState`.

## Register low-level handlers by method

Register a specification request handler by method string:

```typescript
server.setRequestHandler('tools/call', async (request, ctx) => result);
```

For a custom method, provide Standard Schema objects for parameters and,
optionally, result. The callback receives parsed parameters directly after
reserved envelope keys are removed:

```typescript
server.setRequestHandler(
  'acme/search',
  { params: SearchParams, result: SearchResult },
  async (params, ctx) => result
);
```

Specification notification handlers still receive the complete
`{ method, params }` envelope. A strict custom parameter schema must allow an
optional `_meta`.

The negotiated revision selects one codec and method registry for the whole
`Client` or `Server`. `MessageExtraInfo.classification` cannot change eras per
message. Sending a specification method absent from the selected era fails
locally with `SdkError(MethodNotSupportedByProtocolVersion)`; an inbound
deleted method returns `-32601` even if application code registered a handler.

## Register high-level primitives

Replace variadic `.tool()`, `.prompt()`, and `.resource()` with
`registerTool`, `registerPrompt`, and `registerResource`, using configuration
objects and Standard Schema instances. `registerResource` always requires a
metadata argument, even `{}`.

```typescript
server.registerTool('greet', {
  description: 'Greet a user',
  inputSchema: z.object({ name: z.string() })
}, async ({ name }) => ({
  content: [{ type: 'text', text: `Hello, ${name}!` }]
}));
```

For a tool or prompt without an input schema, the callback's only argument is
`ctx`, not an empty argument object.

Raw Zod shapes remain only through deprecated overloads. Zod 3 is unsupported;
use Zod 4.2 or newer, another Standard Schema implementation, or
`fromJsonSchema()`. Make a completable optional field with
`completable(z.string(), callback).optional()`, wrapping the completable result
rather than its inner string.

## Send typed outbound requests

For specification methods, `request()`, `callTool()`, and
`ctx.mcpReq.send()` infer and validate the result from the method name. Remove
the explicit result-schema argument:

```typescript
await client.request({ method: 'tools/list', params: {} });
```

Custom methods and arbitrary forwarded methods still need a result schema. A
gateway can use `ResultSchema` from `@modelcontextprotocol/core` as a
passthrough:

```typescript
await client.request({ method, params }, ResultSchema);
```

## Import schemas and discriminate types

Import raw Zod `*Schema` constants only from
`@modelcontextprotocol/core`. Client and server package roots expose Zod-free
`isSpecType` guards and synchronous `specTypeSchemas` Standard Schemas.

```typescript
import { CallToolResultSchema } from '@modelcontextprotocol/core';
import { isSpecType } from '@modelcontextprotocol/client';
```

Inferred v2 result objects carry index signatures, so property-`in` checks do
not discriminate their unions. Use the provided guards.

The v1 result-only `JSONRPCResponse*` names become
`JSONRPCResultResponse*`. V2's `JSONRPCResponse*` names include result and error
responses. The wire type is `ResourceTemplateType`; the URI-template helper
class remains `ResourceTemplate`.

## Validate JSON Schema by runtime

The default validator is AJV on Node and `@cfworker/json-schema` on Workers. It
dispatches from a schema's declared dialect, defaulting a missing `$schema` to
2020-12.

Importing an adapter from an explicit `/validators/ajv` or
`/validators/cf-worker` subpath makes that backend a direct peer dependency.

## Handle headers and content types

Inbound headers use Web `Headers`; read them with `.get()`.
`requestInit.headers` still accepts any `HeadersInit`. A custom `Accept` value
is appended to required MCP media types rather than replacing them.

Server POST entries parse `Content-Type` and return HTTP 415 unless its media
type is `application/json`.

On modern HTTP, `Client.callTool()` mirrors arguments marked `x-mcp-header`
into `Mcp-Param-*`. If no prior `tools/list` provided the definition, pass
`CallToolRequestOptions.toolDefinition`.

`createMcpHandler()` validates routing and parameter headers and returns HTTP
400 with `-32020` on mismatch. A well-formed error body addressed to the
request is delivered as `ProtocolError`. Browser clients skip dynamic
parameter-header mirroring.

## Use the v2 error hierarchy

Choose the error class by failure layer:

| Layer | Type |
| --- | --- |
| Wire-visible protocol failure | `ProtocolError`, `ProtocolErrorCode` |
| Local SDK failure | `SdkError`, string-valued `SdkErrorCode` |
| HTTP transport failure | `SdkHttpError` |

An `SdkHttpError` exposes HTTP status as `.status`, not `.code`. Timeouts and
closed connections are SDK errors. Custom transports and tests construct an
HTTP failure with `(sdkCode, message, { status, statusText })`.

```typescript
if (error instanceof SdkHttpError && error.status === 401) reauthorize();
if (error instanceof SdkError &&
    error.code === SdkErrorCode.RequestTimeout) retry();
```

Unknown or disabled tool calls reject with
`ProtocolError(InvalidParams)`, not a resolved `isError` tool result. Relayed
protocol messages no longer carry an `MCP error <code>:` prefix. A gateway
recreates an inbound wire failure with
`ProtocolError.fromError(code, message, data)` before throwing it.

## Understand client list behavior

`listPrompts()`, `listResources()`, `listResourceTemplates()`, and
`listTools()` return empty results when the peer did not advertise the
capability, unless `enforceStrictCapabilities` is enabled.

Without a cursor they aggregate pages up to `listMaxPages`, whose default is
64. Supplying a cursor fetches exactly one page.

Tool output validators compile lazily on the first `callTool()`. An invalid
output schema therefore fails before that call is sent, not during
`listTools()`.

## Configure response caching

On a modern connection, cacheable typed verbs reuse results according to
`ttlMs` and `cacheScope`. Override one call with `cacheMode: 'refresh'` or
`'bypass'`; configure `cachePartition` and `defaultCacheTtlMs` when needed.

A custom `ResponseCacheStore` persists `CacheEntry.value` as the JSON-serialized
result string and implements `delete(key)`.

Modern serving always emits cache fields. Conservative defaults are
`ttlMs: 0` and `cacheScope: 'private'`; configure
`ServerOptions.cacheHints` per operation or resource metadata `cacheHint`.
Legacy responses never receive those fields.

## Understand server defaults and validation

`McpServer` eagerly installs handlers for each declared primitive capability.
An empty declared capability returns an empty list and is advertised with
`listChanged: true` unless explicitly disabled.

Localhost app factories validate present `Origin` headers by default.
Configure `allowedOrigins` for trusted browser clients. Opaque
`Origin: null` is always rejected with HTTP 403.

## Configure version selection

`ProtocolOptions.supportedProtocolVersions` pins and orders versions offered
during legacy initialization. Modern probing is controlled separately by
`versionNegotiation`.

A client can persist `client.getDiscoverResult()` and reconnect with
`{ prior: { kind: 'modern', discover } }`, or specify
`{ prior: { kind: 'legacy' } }` to skip probing. The host owns freshness of
that discovery cache.

`DiscoverResult` does not publicly expose `serverInfo` or the response
`ttlMs`/`cacheScope`; use `client.getServerVersion()` for identity and allow
the response-cache layer to consume freshness hints.

## Opt into the modern era

A v2 `Client` performs the 2025 initialization exchange unless
`versionNegotiation` is configured.

With mode `auto`, it probes `server/discover` and falls back only when a legacy
revision remains allowed. Authentication responses, HTTP timeouts, and 5xx
probe failures do not select legacy. An official stdio probe timeout or child
exit may permit fallback.

Pinning the modern revision with `{ pin: '2026-07-28' }` fails with
`SdkError(EraNegotiationFailed)` rather than falling back:

```typescript
const client = new Client(
  { name: 'my-client', version: '1.0.0' },
  {
    versionNegotiation: {
      mode: 'auto',
      probe: { timeoutMs: 10_000, maxRetries: 0 }
    }
  }
);
```

## Choose revision-aware server entry points

Directly connecting `Server` or `McpServer` to stdio remains 2025-era.

- Use `createMcpHandler()` for per-request HTTP serving. It supports modern and
  stateless legacy behavior by default.
- Pass `legacy: 'reject'` for a modern-only entry.
- Use `serveStdio()` to pin an era for each stdio connection.
- Wrap a fetch-shaped handler with `toNodeHandler()` on Node. The alpha-era
  `.node()` method is removed.

```typescript
const handler = createMcpHandler(() => buildServer());
app.all('/mcp', toNodeHandler(handler));
serveStdio(() => buildServer());
```

## Test the actual modern entry

`InMemoryTransport.createLinkedPair()` exercises 2025-era instances only. Test
modern HTTP without a socket by injecting the handler's `.fetch` into
`StreamableHTTPClientTransport`:

```typescript
const handler = createMcpHandler(buildServer);
const transport = new StreamableHTTPClientTransport(
  new URL('http://test.local/mcp'),
  { fetch: (url, init) => handler.fetch(new Request(url, init)) }
);
```

For a modern stdio test, spawn a child that runs `serveStdio()`.

## Map modern HTTP failures

After dispatch, `createMcpHandler()` maps
`MissingRequiredClientCapabilityError` (`-32021`) to HTTP 400. Other
handler-produced protocol errors remain in-band on HTTP 200.

If the SSE response is already committed, the error stays in that HTTP 200
stream. A proxy translates downstream `-32021` for its own hop rather than
rethrowing unchanged downstream `requiredCapabilities`.

Aborting a modern HTTP request closes its response stream rather than sending
`notifications/cancelled`. Legacy connections and stdio in either era still
send the notification. A custom transport opts into modern cancellation with
`readonly hasPerRequestStream = true`.

## Implement multi-round handlers

Return `inputRequired()` from a modern handler instead of sending
server-to-client requests. On re-entry, inspect data with
`acceptedContent()` or discriminated `inputResponse()`:

```typescript
const answer = inputResponse(ctx.mcpReq.inputResponses, 'choice');
if (answer.kind === 'missing') {
  return inputRequired({
    inputRequests: { choice: inputRequired.elicit(request) },
    requestState
  });
}
```

Push-style elicitation, sampling, roots, and ping calls fail locally before
reaching the wire on a modern request.

Each retry carries only that round's responses. Configure
`ServerOptions.requestState.verify`, bind state to principal, operation, and
expiry, and remember that `createRequestStateCodec()` signs but does not
encrypt:

```typescript
const stateCodec = createRequestStateCodec<State>({
  key: SECRET,
  ttlSeconds: 300
});
const requestState = await stateCodec.mint({ step: 'awaiting-input' });
const state = ctx.mcpReq.requestState<State>();
```

A modern client automatically fulfills embedded requests using its elicitation,
sampling, and roots handlers for up to 10 rounds. Disable that with
`inputRequired: { autoFulfill: false }`; manually drive a call using
`allowInputRequired: true` and `withInputRequired()`.

The default legacy shim re-enters the same handler for up to 8 rounds with a
600,000 ms per-leg timeout. Full behavior needs a sessionful streaming
connection; stateless legacy HTTP returns a capability refusal.

## Publish and consume subscriptions

Serving entries implement `subscriptions/listen`. An HTTP handler publishes
through:

- `.notify.toolsChanged`;
- `.notify.promptsChanged`;
- `.notify.resourcesChanged`;
- `.notify.resourceUpdated(uri)`.

Supply a `ServerEventBus` for cross-process delivery.
`ClientOptions.listChanged` automatically opens a filtered modern stream;
`client.listen(filter)` opens one explicitly. `McpSubscription.closed`
distinguishes graceful `'graceful'` closure from unexpected `'remote'`
closure.

## Remove or isolate task APIs

The experimental 2025 task interception surface—stores, managers, stream
helpers, registration, task contexts, and `RequestOptions.task`—has no
mechanical v2 migration and must be removed.

Deprecated task wire types remain only for 2025-11-25 interoperability.
`RequestMethod`, method maps, and method-keyed overloads omit task operations,
so even 2025 calls require explicit-schema custom-method syntax:

```typescript
await client.request(
  { method: 'tasks/get', params },
  GetTaskResultSchema
);
```

A modern peer rejects inbound `tasks/*` with `-32601`; use the official
extension shape instead.
