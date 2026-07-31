# Streamable HTTP, Sessions, and Resumption

## Endpoint and POST framing (`2025-03-26-compat`)

Streamable HTTP replaces the split HTTP+SSE transport with one MCP endpoint
that supports POST and optional GET. Send every client message in a fresh POST:

```http
POST /mcp HTTP/1.1
Accept: application/json, text/event-stream
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"ping"}
```

The `Accept` header must list both response media types. For accepted
notification or response-only input, return an empty HTTP 202. For request
input, return either one `application/json` response or an SSE stream. Clients
must support both forms.

As clarified in `2025-06-18-compat`, each POST body is exactly one JSON-RPC
request, notification, or response. A top-level JSON-RPC array is invalid.

## Optional GET stream (`2025-03-26-compat`)

A client may make a separate GET with `Accept: text/event-stream` for
server-initiated traffic. A server without that stream returns 405. The GET
stream does not carry ordinary JSON-RPC responses, except while replaying the
stream of a previous request.

## Protocol-version header (`2025-06-18-compat`)

After initialization, send the negotiated version on every subsequent HTTP
request:

```http
MCP-Protocol-Version: 2025-06-18
```

When no other version information is available, an absent header means
`2025-03-26`. An invalid or unsupported value produces HTTP 400.

## Stateful sessions (`2025-03-26-compat`)

A server may return `Mcp-Session-Id` with the initialization response. Repeat
the value on every later HTTP request. If the server requires a session and the
header is missing, return 400. If the ID has expired or was terminated, return
404; the client then starts a new initialization without an ID.

Clients should request explicit cleanup with DELETE. Servers may decline that
operation with 405.

## SSE event IDs, replay, and cancellation (`2025-03-26-compat`)

Servers may attach event IDs that are unique across a session, or across a
client when sessions are absent. A reconnecting client sends `Last-Event-ID` on
a GET. Replay only the disconnected stream; do not mix events from unrelated
streams.

A dropped stream does not cancel the JSON-RPC request that created it. To stop
the operation, send an explicit `CancelledNotification`.

## Pollable SSE responses (`2025-11-25-compat`)

For an SSE response that can be resumed, first send an event ID with empty data.
Once the stream has an ID, the server may close the HTTP connection while
keeping the logical stream alive. Before closing, it should send an SSE `retry`
delay. The client honors that delay and reconnects with GET plus
`Last-Event-ID`, regardless of whether the original stream came from POST or
GET.

## Legacy HTTP+SSE compatibility

To serve older clients, keep the legacy SSE and POST endpoints alongside the
Streamable HTTP endpoint. A client given an unknown server URL first POSTs an
`InitializeRequest` with the Streamable HTTP `Accept` header. Success selects
Streamable HTTP. An allowed probe failure triggers a GET; an initial `endpoint`
event identifies the `2024-11-05` HTTP+SSE transport and supplies the endpoint
used for later communication.

The original `2025-03-26-compat` rule described any 4xx as a probe failure.
`2025-11-25-compat` narrows this: only 400, 404, or 405 permit the legacy GET
fallback. Treat every other 4xx as the operation's real error.

## Connection security and Origin failures

Validate `Origin` on every incoming connection. Local servers bind to
`127.0.0.1` rather than `0.0.0.0` and authenticate clients. Authorization
endpoints use HTTPS, and redirect URIs are localhost or HTTPS. Under
`2025-11-25`, an invalid `Origin` response is HTTP 403 Forbidden.
