# Protocol lifecycle, messages, schemas, and results

Use this reference to select a protocol era, validate messages, expose
capabilities and metadata, implement cacheable results, and assign errors.

Relevant protocol attributions: `2025-03-26`, `2025-06-18`,
`2025-11-25`, and `2026-07-28-rc`.

## Select one protocol era

The negotiated revision chooses a coherent method registry, lifecycle, result
shape, transport behavior, and capability vocabulary. Do not mix individual
features from different eras on one connection.

### Legacy lifecycle

The 2025-06-18 revision strengthens the lifecycle operation requirement from
SHOULD to MUST. Legacy implementations must perform the initialization
lifecycle and use its negotiated version and capabilities.

### Modern stateless lifecycle

The modern RC removes protocol-level sessions, `Mcp-Session-Id`, `initialize`,
and `notifications/initialized`. List endpoints no longer vary by connection.
Servers that need cross-call state expose an explicit handle, usually as an
ordinary tool argument.

Every request carries these reserved `_meta` values:

- `io.modelcontextprotocol/protocolVersion`;
- `io.modelcontextprotocol/clientCapabilities`;
- preferably `io.modelcontextprotocol/clientInfo`.

Every modern server response should carry
`io.modelcontextprotocol/serverInfo`. A version mismatch produces
`UnsupportedProtocolVersionError`.

## Discover server support

Modern servers implement `server/discover` and report supported protocol
versions, capabilities, and identity:

```json
{"jsonrpc":"2.0","id":1,"method":"server/discover"}
```

A client may call discovery before any other request to select a revision. It
can also use the call as a backward-compatibility probe over stdio. Treat
self-reported server identity as display and diagnostic data, never as input to
security or behavioral decisions.

## Frame one JSON-RPC message at a time

The 2025-03-26 revision introduced JSON-RPC batches as a top-level array. The
2025-06-18 revision removed that feature: a top-level array is no longer valid
MCP. Streamable HTTP likewise accepts one request, notification, or response in
each POST body.

Only produce an array for a peer explicitly pinned to the brief
2025-03-26 behavior:

```json
[
  {"jsonrpc":"2.0","id":1,"method":"ping"},
  {"jsonrpc":"2.0","id":2,"method":"tools/list"}
]
```

## Negotiate and respect capabilities

- Tool definitions may carry behavioral annotations, including whether a tool
  is read-only or destructive. Treat them as presentation and planning hints,
  not as an authorization boundary.
- A server advertises the `completions` capability before a client relies on
  argument completion. `CompletionRequest.context` can include variables
  already resolved so later suggestions are context-aware.
- In the modern era, `ClientCapabilities` and `ServerCapabilities` also expose
  an `extensions` field. Negotiate extension-specific behavior there rather
  than assuming it is core.

## Content and progress values

MCP content can include audio as well as text and images.

`ProgressNotification` supports a descriptive `message` in addition to
numeric progress values:

```json
{
  "jsonrpc":"2.0",
  "method":"notifications/progress",
  "params":{"progressToken":"job-7","progress":42,"message":"Indexing files"}
}
```

Keep request-scoped progress on the transport stream belonging to that request.

## Names, titles, icons, and descriptions

Use `name` as the programmatic identifier in protocol calls and optional
`title` as the human-friendly display label. Servers may provide icon metadata
for tools, resources, resource templates, and prompts.

`Implementation.description` supplies optional human-readable client or server
context during initialization. None of these display fields changes identity
or method dispatch.

## Use `_meta` for extensibility

The 2025-06-18 schemas permit `_meta` on additional interface shapes.
Validators and generated bindings must allow it where the selected revision
defines it.

Modern OpenTelemetry propagation uses conventional `traceparent`,
`tracestate`, and `baggage` keys inside `_meta`.

Reserved envelope, protocol, tracing, retry, and subscription metadata must be
handled separately from application parameters. Do not depend on unknown
top-level fields surviving an SDK parse.

## Apply JSON Schema 2020-12

JSON Schema 2020-12 is the default dialect for MCP schema definitions unless a
schema explicitly selects another dialect.

Modern tool `inputSchema` and `outputSchema` values may use any keyword in that
dialect. Implementations must resolve `$ref` and put resource bounds around
composition keywords. `structuredContent` may be any JSON value, not only an
object. Generated schema definitions for `minimum`, `maximum`, and `default`
accept numbers, not only integers.

Request parameter schemas are standalone from RPC method definitions. Schema
consumers and generated bindings must import or generate those separate
parameter types rather than assuming every method owns an inline payload
schema.

## Distinguish tool and protocol failures

A tool argument validation failure should be a Tool Execution Error rather
than a Protocol Error, so the caller can inspect and correct its input.
Malformed MCP envelopes, unsupported methods, capability failures, and
transport-contract violations remain protocol errors.

For modern errors:

| Condition or range | Code |
| --- | --- |
| Resource not found | `-32602` |
| Implementation-defined server errors | `-32000` through `-32019` |
| MCP-reserved errors | `-32020` through `-32099` |
| Header mismatch | `-32020` |
| Missing required client capability | `-32021` |
| Unsupported protocol version | `-32022` |

The modern schema includes `HeaderMismatchError`. Do not retain the older
resource-not-found allocation `-32002`.

## Produce typed modern results

Every modern result carries `resultType`:

- `"complete"` for an ordinary result;
- `"input_required"` for a result that embeds client work.

A modern codec must reject a missing discriminator. A compatibility client
reading a result from an older revision may interpret a missing value as
`"complete"`. SDKs may consume the wire discriminator and omit it from public
result objects.

## Cache only cacheable operations

Results for these modern operations implement `CacheableResult`:

- `tools/list`;
- `prompts/list`;
- `resources/list`;
- `resources/read`;
- `resources/templates/list`.

Each contains `ttlMs` and `cacheScope`, whose value is `"public"` or
`"private"`:

```json
{"ttlMs":30000,"cacheScope":"private"}
```

Return `tools/list` in deterministic order. A zero TTL and private scope are
safe conservative defaults. Cache keys must partition any response whose
contents depend on user, tenant, authorization, or other private context.

## Removed and deprecated core behavior

The modern RC removes `ping`, `logging/setLevel`, and
`notifications/roots/list_changed`. A request opts into logging with
`_meta["io.modelcontextprotocol/logLevel"]`; a server must not emit
`notifications/message` for a request that omitted it.

Roots, Sampling, Logging, HTTP+SSE, and Sampling's `includeContext` values
`"thisServer"` and `"allServers"` are deprecated. Existing compatibility paths
may continue during the deprecation window, but new implementations should:

- pass explicit tool parameters, resource URIs, or server configuration instead
  of Roots;
- call provider APIs directly instead of Sampling;
- use stderr or OpenTelemetry instead of protocol Logging;
- use Streamable HTTP instead of HTTP+SSE;
- omit `includeContext` or select `"none"`.

See [interaction-patterns.md](interaction-patterns.md) for multi-round
replacement patterns and [http-and-subscriptions.md](http-and-subscriptions.md)
for the transport-level replacements.
