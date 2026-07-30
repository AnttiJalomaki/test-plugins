# Streamable HTTP, sessions, and subscriptions

Use this reference to implement HTTP message flow, legacy sessions and SSE
recovery, modern cancellation and routing, origin checks, and change streams.

Relevant protocol attributions: `2025-03-26-compat`,
`2025-06-18-compat`, `2025-11-25-compat`, `2025-11-25`, and
`2026-07-28-rc`.

## Implement legacy Streamable HTTP

Streamable HTTP replaces the split HTTP+SSE transport with one MCP endpoint
supporting POST and GET.

Send every client message in a fresh POST:

```http
POST /mcp HTTP/1.1
Accept: application/json, text/event-stream
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"ping"}
```

The `Accept` value lists both required media types. For an accepted notification
or response-only message, return an empty HTTP 202. For a request, return either
one JSON response or an SSE stream; clients must support both.

A client may separately GET the endpoint with
`Accept: text/event-stream` for server-initiated traffic. A server without that
stream returns 405. The GET stream must not carry ordinary JSON-RPC responses
except while replaying a previous request's stream.

## Frame and version legacy POSTs

Each POST body contains one JSON-RPC request, notification, or response. A
top-level batch is invalid on this transport.

After initialization, send the negotiated revision on every later HTTP
request:

```http
MCP-Protocol-Version: 2025-06-18
```

Without other version information, a missing header means `2025-03-26`. An
invalid or unsupported header value receives HTTP 400.

## Manage legacy HTTP sessions

A legacy server may return `Mcp-Session-Id` with its initialization response.
The client repeats it on every later HTTP request.

- A required but missing ID produces HTTP 400.
- A terminated or expired ID produces HTTP 404.
- After 404, initialize a new session without an ID.
- Request cleanup with DELETE; the server may reject deletion with 405.

Do not apply these rules to the modern stateless protocol, which removes
protocol-level sessions and this header.

## Resume legacy SSE streams

A legacy server may assign SSE event IDs unique across the session, or across
the client when sessions are absent. On reconnect, the client sends
`Last-Event-ID` in a GET. Replay is confined to the disconnected logical
stream.

A dropped legacy stream does not cancel its request. Send an explicit
`CancelledNotification` when cancellation is intended.

For the 2025-11-25 polling form:

1. The server first emits an event ID with empty data.
2. After assigning the ID, it may close the HTTP connection without terminating
   the logical stream.
3. Before closing, it should emit an SSE `retry` delay.
4. The client honors the delay and reconnects with GET plus `Last-Event-ID`,
   whether the original stream began with POST or GET.

## Detect legacy HTTP+SSE carefully

To support old clients, a server can retain legacy SSE and POST endpoints next
to the Streamable HTTP endpoint.

An unknown server URL is probed with a Streamable HTTP initialization POST. On
the 2025-11-25 behavior, fall back to GET only when that initial POST fails with
400, 404, or 405. Other 4xx responses do not trigger transport fallback. An
initial `endpoint` SSE event identifies the 2024-11-05 HTTP+SSE transport and
selects it for later communication.

This narrows the earlier 2025-03-26 compatibility rule, which treated any 4xx
as a fallback signal.

## Handle modern response streams

Modern Streamable HTTP removes SSE event IDs, `Last-Event-ID`, and message
redelivery. If a response stream breaks, the in-flight request is lost. Retry
the operation with a new request ID rather than attempting to resume it.

Aborting a modern HTTP request closes that request's response stream; it does
not POST `notifications/cancelled`. Legacy connections and stdio in either era
still use the cancellation notification. A custom one-request-per-message
transport can declare `hasPerRequestStream = true` to adopt the modern
behavior.

## Send and validate modern routing headers

Every modern Streamable HTTP POST identifies its protocol method with
`Mcp-Method`. Named tool-like operations also send `Mcp-Name`:

```http
Mcp-Method: tools/call
Mcp-Name: weather
```

An input-schema property marked `x-mcp-header` can mirror its argument into an
`Mcp-Param-*` header. The server cross-checks the routing and parameter headers
against the body. A mismatch returns HTTP 400 with protocol error `-32020`.

Browser clients may be unable to mirror dynamic parameter headers; design tool
schemas and CORS policy with that limitation in mind.

## Open modern change subscriptions

The modern protocol replaces the general GET event stream and
`resources/subscribe`/`resources/unsubscribe` with
`subscriptions/listen`, a long-lived POST-response stream.

A subscription can request:

- `toolsListChanged`;
- `promptsListChanged`;
- `resourcesListChanged`;
- `resourceSubscriptions`.

The server acknowledges the subscription and tags its notifications with
`io.modelcontextprotocol/subscriptionId`. Request-scoped progress and logging
notifications remain on the response stream for their originating request;
they are not moved onto the subscription.

For a multi-process or multi-replica server, publish changes through a shared
event bus so a subscription opened on one process can observe changes made by
another.

## Validate HTTP boundaries

Parse `Content-Type` and return HTTP 415 unless the media type is
`application/json`.

Validate every present `Origin` to prevent DNS rebinding. Return HTTP 403 when
an origin is invalid. Local deployments should bind to `127.0.0.1`, enable
transport-level rebinding protection, authenticate requests, and explicitly
allow trusted browser origins. Opaque `Origin: null` must remain rejected.

See [authorization.md](authorization.md) for bearer tokens, redirect safety,
and protected-resource discovery.
