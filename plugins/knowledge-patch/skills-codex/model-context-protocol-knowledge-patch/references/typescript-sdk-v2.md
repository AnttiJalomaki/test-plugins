# TypeScript SDK v2

## Migration strategy

Run the codemod at each package root:

```bash
npx @modelcontextprotocol/codemod@latest v1-to-v2 .
grep -rn '@mcp-codemod-error' .
```

It updates fixed import, registration, mock, and manifest mappings. It marks
judgment calls with `@mcp-codemod-error` and does not format the result. Resolve
every marker, then format, type-check, and test.

The codemod changes only the nearest manifest and reports other workspace
members. Every member must declare the v2 packages used by its own imports.
V1 and v2 may coexist during a staged migration, but their objects, nominal
types, and class instances cannot cross the boundary. Add v2 first, migrate at
process or transport boundaries, and remove v1 last.

## Runtime and package split

V2 requires Node.js 20 or newer. Replace `@modelcontextprotocol/sdk` with:

- `@modelcontextprotocol/client`
- `@modelcontextprotocol/server`
- Public Zod schemas from `@modelcontextprotocol/core`

Never import unpublished `@modelcontextprotocol/core-internal`.

The SDK ships native ESM and CommonJS builds. Node, Express, Hono, and Fastify
integrations live in adapter packages. Express, Hono, and Fastify adapters
require their framework peer dependency; the Node adapter does not require
Hono.

Resource-server auth helpers live in the Express adapter or the Web-standard
server package. Authorization-server helpers are confined to deprecated,
frozen `@modelcontextprotocol/server-legacy/auth`; move that responsibility to
a dedicated OAuth library.

## Transports and serving entry points

Choose a transport by runtime:

```typescript
import { NodeStreamableHTTPServerTransport } from '@modelcontextprotocol/node';
import { WebStandardStreamableHTTPServerTransport } from '@modelcontextprotocol/server';
import { StdioClientTransport } from '@modelcontextprotocol/client/stdio';
```

- Use `NodeStreamableHTTPServerTransport` for Node request/response handlers.
- Use `WebStandardStreamableHTTPServerTransport` for Web
  `Request`/`Response` runtimes.
- Import stdio transports from package subpaths.
- Use temporary `@modelcontextprotocol/server-legacy/sse` only for the
  surviving `SSEServerTransport` migration bridge.
- Remove `WebSocketClientTransport`.
- Construct both ends of an `InMemoryTransport` pair from the same client or
  server package.

Direct `Server` or `McpServer` connection to stdio remains 2025-era. Use
revision-aware entry points:

```typescript
const handler = createMcpHandler(() => buildServer());
app.all('/mcp', toNodeHandler(handler));
serveStdio(() => buildServer());
```

`createMcpHandler()` serves per-request HTTP and supports modern plus stateless
legacy behavior by default. `serveStdio()` selects an era per connection.
Use `legacy: 'reject'` for modern-only service. On Node, wrap the fetch-shaped
handler with `toNodeHandler()`; the alpha-era `.node()` method is removed.

## Protocol negotiation

Installing v2 does not opt a client into the modern era. Without
`versionNegotiation`, `Client` still performs the 2025 initialize exchange.

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

Auto mode probes `server/discover` and falls back only if a legacy revision is
still allowed. `{ pin: '2026-07-28' }` rejects with
`SdkError(EraNegotiationFailed)`. Authentication responses, HTTP timeouts, and
5xx probe failures do not imply a legacy server; an official stdio probe
timeout or child exit may permit fallback.

For legacy initialization, `ProtocolOptions.supportedProtocolVersions` pins
and orders offered versions. For modern probing, use `versionNegotiation`.

A host may persist `client.getDiscoverResult()` and reconnect with:

```typescript
{ prior: { kind: 'modern', discover } }
```

It can use `{ kind: 'legacy' }` to skip probing, but owns discovery-cache
freshness.

The negotiated revision selects one codec and method registry for the whole
client or server. `MessageExtraInfo.classification` cannot switch eras per
message. A locally known spec method absent from the era fails with
`SdkError(MethodNotSupportedByProtocolVersion)`; an inbound deleted method
returns `-32601` even if an application handler exists.

## Testing negotiated eras

`InMemoryTransport.createLinkedPair()` tests only 2025-era instances. Test
modern HTTP without a socket by feeding the handler's fetch function to the
client transport:

```typescript
const handler = createMcpHandler(buildServer);
const transport = new StreamableHTTPClientTransport(
  new URL('http://test.local/mcp'),
  { fetch: (url, init) => handler.fetch(new Request(url, init)) }
);
```

Modern stdio tests must spawn a `serveStdio()` child.

## Handler context and registration

`RequestHandlerExtra` becomes `ServerContext` or `ClientContext`. Request state
and nested sends are in `ctx.mcpReq`; HTTP-only state is optional in
`ctx.http`.

```typescript
const { signal, id, _meta } = ctx.mcpReq;
await ctx.mcpReq.send(request);
await ctx.mcpReq.notify(notification);
const token = ctx.http?.req?.headers.get('authorization');
```

Register specification handlers by method string. Register a custom method
with `(method, { params, result? }, handler)` and Standard Schema objects.
Custom handlers receive parsed parameters, including `_meta` after reserved
envelope keys are removed. A strict custom parameter schema must allow an
optional `_meta`. Specification notification handlers still receive the full
`{ method, params }` envelope.

```typescript
server.setRequestHandler('tools/call', async (request, ctx) => result);
server.setRequestHandler(
  'acme/search',
  { params: SearchParams, result: SearchResult },
  async (params, ctx) => result
);
```

For known spec methods, `request()`, `callTool()`, and `ctx.mcpReq.send()` infer
and validate results from the method name. Remove the result-schema argument.
A custom or arbitrary forwarded method still supplies one; a gateway can use
`ResultSchema` as a passthrough.

```typescript
await client.request({ method: 'tools/list', params: {} });
await client.request({ method, params }, ResultSchema);
```

## High-level registration and schemas

Replace variadic `.tool()`, `.prompt()`, and `.resource()` with
`registerTool`, `registerPrompt`, and `registerResource`. Pass a configuration
object and Standard Schema instances. `registerResource` always requires a
metadata argument, even `{}`.

```typescript
server.registerTool(
  'greet',
  {
    description: 'Greet a user',
    inputSchema: z.object({ name: z.string() })
  },
  async ({ name }) => ({
    content: [{ type: 'text', text: `Hello, ${name}!` }]
  })
);
```

A tool or prompt without an input schema receives only `ctx`; it does not
receive an empty argument object.

Raw Zod shapes survive only through deprecated overloads. Zod 3 is unsupported;
use Zod 4.2 or newer, another Standard Schema implementation, or
`fromJsonSchema()`. An old Zod range can type-check and then fail on the first
`tools/list`.

For an optional completable field, make the completable result optional:

```typescript
completable(z.string(), callback).optional()
```

Import raw Zod `*Schema` constants only from `@modelcontextprotocol/core`.
Client and server roots expose Zod-free `isSpecType` guards and synchronous
`specTypeSchemas`. V2 inferred result objects have index signatures, so
property-`in` checks do not discriminate result unions; use a guard.

```typescript
import { CallToolResultSchema } from '@modelcontextprotocol/core';
import { isSpecType } from '@modelcontextprotocol/client';
```

## Renamed types and validation

V1 result-only `JSONRPCResponse*` symbols become
`JSONRPCResultResponse*`. V2 `JSONRPCResponse*` symbols now cover both result
and error responses.

The protocol wire type is `ResourceTemplateType`; the URI-template helper
class remains `ResourceTemplate`.

The default JSON Schema validator depends on runtime: AJV on Node and
`@cfworker/json-schema` on Workers. Validation dispatches on a schema's
declared dialect and defaults missing `$schema` to 2020-12. Importing an
explicit `/validators/ajv` or `/validators/cf-worker` adapter makes that
backend a direct peer dependency.

## HTTP request behavior

Inbound headers are Web `Headers`; read them with `.get()`.
`requestInit.headers` still accepts any `HeadersInit`. A custom `Accept` value
is appended to required MCP media types, not substituted for them.

Server POST entry points parse `Content-Type` and return HTTP 415 unless the
media type is `application/json`.

Localhost app factories validate a present `Origin` by default. Configure
`allowedOrigins` for browser clients. Opaque `Origin: null` always receives
HTTP 403.

On modern HTTP, `callTool()` mirrors arguments whose input-schema properties
have `x-mcp-header` into `Mcp-Param-*`. Supply
`CallToolRequestOptions.toolDefinition` if the client did not previously load
the definition through `tools/list`.

The handler validates argument headers and standard routing headers. Mismatch
returns HTTP 400 and `-32020`. A matching error body is delivered as
`ProtocolError`; browser clients skip dynamic header mirroring.

## Error hierarchy

Use:

- `ProtocolError` and `ProtocolErrorCode` for wire-visible failures.
- `SdkError` and string-valued `SdkErrorCode` for local failures such as
  timeout or closed connection.
- `SdkHttpError` for HTTP transport failures; its HTTP status is `.status`,
  not `.code`.

```typescript
if (error instanceof SdkHttpError && error.status === 401) reauthorize();
if (error instanceof SdkError &&
    error.code === SdkErrorCode.RequestTimeout) retry();
```

Custom transports and tests construct HTTP failures with
`(sdkCode, message, { status, statusText })`.

Unknown or disabled tool calls reject with
`ProtocolError(InvalidParams)` instead of resolving an `isError` tool result.
Relayed messages omit the old `MCP error <code>:` prefix. Closing a transport
aborts in-flight handlers through `ctx.mcpReq.signal`.

A gateway recreates an inbound wire error with
`ProtocolError.fromError(code, message, data)` before throwing it.

After dispatch, `createMcpHandler()` maps
`MissingRequiredClientCapabilityError` (`-32021`) to HTTP 400. Other
handler-generated protocol errors remain in-band on HTTP 200. Once an SSE
response is committed, errors remain within that 200 stream. A proxy must
translate required capabilities for its own hop.

## OAuth client and bearer behavior

V2 collapses separate OAuth exceptions into `OAuthError` and
`OAuthErrorCode`. Bearer middleware verifiers must throw the v2 error; legacy
or generic invalid-token errors become HTTP 500.

`AuthProvider` supports non-OAuth bearer credentials through `token()` and
optional `onUnauthorized()`, with one retry after a 401.

The host validates callback `state`, then passes `URLSearchParams` to
`finishAuth()` for `iss` validation. Non-loopback token endpoints use HTTPS.
Persist tokens and client information without reconstructing them, key
multi-issuer records by `ctx.issuer`, support discovery-state persistence, and
choose an insufficient-scope policy with `onInsufficientScope`.

## Client list methods and response caching

`listPrompts()`, `listResources()`, `listResourceTemplates()`, and
`listTools()` return empty results when the server did not advertise the
capability, unless `enforceStrictCapabilities` is enabled.

Without a cursor, a list call aggregates pages up to `listMaxPages`, whose
default is 64. An explicit cursor fetches exactly one page.

Output validators compile lazily on the first `callTool()`. An invalid tool
output schema fails there before a request is sent, not during `listTools()`.

On modern connections, cacheable typed calls reuse results according to
`ttlMs` and `cacheScope`. Per call, select `cacheMode: 'refresh'` or
`'bypass'`. Configure `cachePartition` or `defaultCacheTtlMs` on the client.

A custom `ResponseCacheStore` persists `CacheEntry.value` as the
JSON-serialized result string and implements `delete(key)`.

Modern servers always emit cache fields, conservatively defaulting to
`ttlMs: 0` and `cacheScope: 'private'`. Override by operation through
`ServerOptions.cacheHints` or with resource metadata `cacheHint`. Legacy
responses do not contain these fields.

`DiscoverResult` does not publicly expose `serverInfo`, `ttlMs`, or
`cacheScope`. Read identity through `client.getServerVersion()` and let the
cache layer consume hints. Identity is diagnostic, not authoritative.

## stdio behavior

Both stdio transports accept `maxBufferSize`, default 10 MB. They close through
`onerror` if a message exceeds the limit. `ReadBuffer.readMessage()` skips
non-JSON stdout lines but rejects JSON that fails JSON-RPC validation.

## Server defaults

`McpServer` eagerly installs handlers for declared primitive capabilities. An
empty capability returns an empty list and advertises `listChanged: true`
unless explicitly disabled.

## Modern multi-round handler APIs

Modern handlers return `inputRequired()` and read re-entry data with
schema-aware `acceptedContent()` or discriminated `inputResponse()`.

```typescript
const answer = inputResponse(ctx.mcpReq.inputResponses, 'choice');
if (answer.kind === 'missing') {
  return inputRequired({
    inputRequests: { choice: inputRequired.elicit(request) },
    requestState
  });
}
```

Push-style elicitation, sampling, roots, and ping APIs fail locally during a
modern request.

Public result types do not expose `resultType`; the SDK consumes it. A modern
wire response missing it is a typed protocol failure. Reserved retry and
envelope fields are lifted into `ctx.mcpReq`, even for a legacy custom request
whose ordinary parameters happen to use `inputResponses` or `requestState`.

A modern client auto-fulfills embedded roots, sampling, and elicitation
requests through registered handlers, retrying for at most 10 rounds. Disable
this with `inputRequired: { autoFulfill: false }`; then call with
`allowInputRequired: true` and drive the flow with `withInputRequired()`.

The default legacy shim re-enters a handler for at most 8 rounds with a
600,000 ms timeout per leg. Full emulation requires a sessionful streaming
connection; stateless legacy HTTP returns a capability refusal.

## Integrity-protected retry state

Only the current round's `inputResponses` travels on each retry. Put earlier
answers and the flow phase in `requestState`.

Configure `ServerOptions.requestState.verify` and bind the token to principal,
operation, and expiry. `createRequestStateCodec()` signs but does not encrypt.

```typescript
const stateCodec = createRequestStateCodec<State>({
  key: SECRET,
  ttlSeconds: 300
});
const serverOptions = { requestState: { verify: stateCodec.verify } };
const requestState = await stateCodec.mint({ step: 'awaiting-input' });
const state = ctx.mcpReq.requestState<State>();
```

## Cancellation and subscriptions

Aborting a modern HTTP request closes its SSE response stream instead of
sending `notifications/cancelled`. Legacy and stdio connections still send the
notification. A custom per-request transport advertises
`readonly hasPerRequestStream = true`.

Serving entry points implement `subscriptions/listen`. Publish with
`.notify.toolsChanged`, `.notify.promptsChanged`,
`.notify.resourcesChanged`, and `.notify.resourceUpdated(uri)`. Supply a
shared `ServerEventBus` for cross-process delivery.

`ClientOptions.listChanged` automatically opens a filtered modern stream;
`client.listen(filter)` opens one explicitly. `McpSubscription.closed`
distinguishes graceful closure from unexpected remote closure.

## Removed task interception

Remove the experimental 2025 task side-channel APIs: task stores and managers,
stream helpers, registration, task contexts, and `RequestOptions.task`.
There is no mechanical v2 migration.

Deprecated task wire types remain for 2025 interoperability, but
`RequestMethod`, method maps, and method-keyed overloads omit task methods.
Call a 2025 peer through the explicit-schema custom form:

```typescript
await client.request(
  { method: 'tasks/get', params },
  GetTaskResultSchema
);
```

A modern peer rejects inbound `tasks/*` core methods with `-32601`.
