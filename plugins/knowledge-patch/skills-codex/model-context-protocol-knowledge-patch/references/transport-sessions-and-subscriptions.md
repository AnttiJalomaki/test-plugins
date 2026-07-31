# Transport, Sessions, and Subscriptions

## Use the single Streamable HTTP endpoint

Streamable HTTP replaces the old HTTP+SSE pair with one MCP endpoint that
supports POST and optional GET. Every client message gets a fresh POST. Set
both accepted response media types:

```http
POST /mcp HTTP/1.1
Accept: application/json, text/event-stream
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"ping"}
```

For a JSON-RPC request, the server returns either one JSON response or an SSE
stream; a client must support both. Accepted notification-only or response-only
input returns an empty HTTP 202.

The transport behavior above comes from compatibility batch
`2025-03-26-compat`.

## Keep one message per POST

Each Streamable HTTP POST body contains exactly one JSON-RPC request,
notification, or response. A top-level batch is invalid. After initialization,
the client sends the negotiated protocol revision on every later HTTP request:

```http
MCP-Protocol-Version: 2025-06-18
```

Without other version information, a missing header means `2025-03-26`. An
invalid or unsupported header value produces HTTP 400. These framing rules are
part of compatibility batch `2025-06-18-compat`.

## Open the optional server-initiated GET stream

A client may separately GET the endpoint with
`Accept: text/event-stream` for server-initiated traffic. A server that does
not offer this stream returns HTTP 405. The GET stream must not carry ordinary
JSON-RPC responses, except while replaying the response stream of a previous
request.

## Resume an SSE stream

Servers may attach an ID to each SSE event. IDs must be unique across the
session, or across the client when sessions are absent. When a connection
drops, reconnect with GET and `Last-Event-ID`. Replay is confined to the
disconnected stream.

A dropped SSE connection does not cancel its associated request. Cancellation
requires an explicit `CancelledNotification`.

## Poll Streamable HTTP SSE

Compatibility batch `2025-11-25-compat` refines reconnectable SSE behavior. A
server should first send an event ID with empty data. Once the stream has an
ID, the server may close the HTTP connection without ending the logical stream;
it should send an SSE `retry` delay before doing so.

The client must honor `retry` and reconnect with GET plus `Last-Event-ID`. This
applies whether the logical stream originated from POST or GET.

## Maintain stateful HTTP sessions

A server may include `Mcp-Session-Id` with its initialization response. The
client must repeat that value on every later HTTP request.

- A required but missing session ID should produce HTTP 400.
- A terminated or expired session ID produces HTTP 404.
- After 404, initialize a new session without sending an ID.
- Request session cleanup with DELETE.
- A server may reject DELETE with HTTP 405.

Authorization is independent of the session ID. Continue sending the Bearer
token on every authorized request.

## Protect Streamable HTTP

Validate `Origin` on every incoming connection to prevent DNS rebinding. Since
stable batch `2025-11-25`, rejection of an invalid `Origin` must return HTTP
403 Forbidden.

For local servers:

- bind to `127.0.0.1` instead of `0.0.0.0`;
- authenticate connections;
- serve authorization endpoints over HTTPS;
- allow only localhost or HTTPS redirect URIs.

## Interoperate with legacy HTTP+SSE

Servers that support older clients should retain the legacy SSE and POST
endpoints beside the Streamable HTTP endpoint.

For an unknown server URL, first POST an `InitializeRequest` using the
Streamable HTTP `Accept` header. A successful response selects Streamable
HTTP. Current fallback behavior is narrower than the original profile: fall
back to GET only if that first POST returns HTTP 400, 404, or 405. Other 4xx
responses do not trigger legacy detection.

On fallback, require an initial `endpoint` SSE event. It identifies the
2024-11-05 HTTP+SSE transport and supplies the POST endpoint for later
communication.
