# Transport, sessions, and subscriptions

## Streamable HTTP request flow

Streamable HTTP replaces the legacy HTTP+SSE pair with one MCP endpoint. Each
client message gets a fresh POST.

For a 2025-era request:

```http
POST /mcp HTTP/1.1
Accept: application/json, text/event-stream
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}
```

The `Accept` header lists both required media types. A request-bearing POST
returns either one JSON response or an SSE response stream; clients must
support both. A POST containing only accepted notifications or responses
returns an empty HTTP 202.

Each POST body contains one JSON-RPC request, notification, or response. A
top-level batch array is invalid on this transport. Servers must parse
`Content-Type`; reject a POST with HTTP 415 unless its media type is
`application/json`.

After legacy initialization, every request sends the negotiated
`MCP-Protocol-Version`.

## Optional 2025 GET stream

A 2025 client may separately GET the endpoint with:

```http
Accept: text/event-stream
```

The GET stream is for server-initiated traffic. A server that does not offer it
returns HTTP 405. It must not carry ordinary JSON-RPC responses, except while
replaying a previous request's response stream.

The modern protocol removes this GET event stream and replaces it with a
request-bearing `subscriptions/listen` POST response.

## Legacy HTTP sessions

A legacy server may return `Mcp-Session-Id` with the initialization response.
The client repeats it on every subsequent HTTP request.

Use these status rules:

- A required but missing session ID produces HTTP 400.
- A terminated or expired ID produces HTTP 404.
- After 404, initialize again without a session ID.
- Request cleanup with DELETE; a server may reject cleanup with HTTP 405.

Modern MCP removes the protocol session and the `Mcp-Session-Id` header.
Applications needing cross-call state use explicit handles in ordinary tool
arguments.

## SSE resumption in older revisions

An older server may assign SSE event IDs that are unique across a session, or
across a client when sessions do not exist.

On disconnection:

1. The client reconnects with GET and `Last-Event-ID`.
2. Replay is limited to the disconnected stream.
3. The dropped stream does not cancel its request.
4. Cancellation requires an explicit `CancelledNotification`.

For pollable Streamable HTTP in `2025-11-25-compat`, a server should first send
an event ID with empty data. After assigning that ID, it can close the HTTP
connection without ending the logical stream and should first send an SSE
`retry` delay. The client honors the delay and reconnects with GET plus
`Last-Event-ID`, whether the original response stream began with POST or GET.

## Modern response streams and cancellation

The `2026-07-28-rc` removes SSE event IDs, `Last-Event-ID`, and protocol
redelivery. If a response stream breaks, its in-flight request is lost. The
client resubmits the operation with a new JSON-RPC request ID.

Aborting a modern Streamable HTTP request closes only that request's SSE
stream; it does not POST `notifications/cancelled`. Legacy connections and
stdio connections in either era still send the cancellation notification.
A custom transport that maps one request to one stream can advertise that
behavior to an SDK.

## Legacy HTTP+SSE compatibility

A compatibility server can keep the old SSE and POST endpoints alongside the
Streamable HTTP endpoint.

When given an unknown server URL, an older client first POSTs an
`InitializeRequest` with the Streamable HTTP `Accept` header. The original
compatibility rule fell back to GET on any 4xx. Since
`2025-11-25-compat`, fallback is restricted to HTTP 400, 404, or 405. Other 4xx
responses do not trigger it.

If the fallback GET produces an initial `endpoint` event, select the
2024-11-05 HTTP+SSE transport for all later communication.

## Origin and network security

Validate `Origin` on every incoming HTTP connection to prevent DNS rebinding.
When an invalid origin is rejected, return HTTP 403.

For local servers:

- Bind to `127.0.0.1`, not `0.0.0.0`, unless remote exposure is intentional.
- Authenticate connections.
- Serve authorization endpoints over HTTPS.
- Accept only loopback or HTTPS redirect URIs.
- Configure explicit allowed origins for browser clients.
- Reject opaque `Origin: null`.

SDK loopback app factories may enable DNS-rebinding protection by default, but
the host still owns the browser-origin allowlist.

## Modern routing headers

Every modern Streamable HTTP POST includes `Mcp-Method`; the three tool-like
operations also include `Mcp-Name`.

```http
Mcp-Method: tools/call
Mcp-Name: weather
```

Tool input-schema properties marked `x-mcp-header` can be mirrored into
`Mcp-Param-*` request headers. The server cross-checks routing and argument
headers against the JSON body.

A mismatch returns HTTP 400 with protocol error `-32020`. When a well-formed
error body addresses the request, the client should surface it as a protocol
error rather than as an opaque HTTP failure. Browser clients skip dynamic
tool-argument header mirroring.

## Subscription stream

Modern MCP replaces the GET event stream and
`resources/subscribe`/`resources/unsubscribe` with
`subscriptions/listen`. This is a long-lived POST response stream.

The requested filter can opt into:

- `toolsListChanged`
- `promptsListChanged`
- `resourcesListChanged`
- `resourceSubscriptions`

The server acknowledges the subscription and marks its notifications with
`io.modelcontextprotocol/subscriptionId`.

Request-scoped progress and logging notifications do not use the subscription
stream; they stay on the response stream for the request that caused them.

In SDK servers, the serving entry point owns `subscriptions/listen`. Publish
tool, prompt, resource-list, and resource-update events through the entry
point's notification helpers. Multi-process or multi-replica servers need a
shared event bus so subscribers see events produced by another process.

Clients can open a filtered subscription automatically for list changes or
explicitly with a listen API. Track whether the server honored all requested
filters and distinguish graceful closure from an unexpected remote close.

## Transport implementation checklist

- Accept both JSON and SSE responses where the negotiated era permits them.
- Keep each response and its request-scoped notifications on the same stream.
- Apply era-specific cancellation and retry behavior.
- Do not attach modern sessions or legacy resumption semantics to modern
  requests.
- Validate content type, origin, protocol version, method/name routing, and
  mirrored argument headers before dispatch.
- Preserve transport-level failures separately from in-band protocol errors.
