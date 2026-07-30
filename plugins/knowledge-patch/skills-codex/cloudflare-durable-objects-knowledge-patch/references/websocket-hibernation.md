# WebSocket hibernation

## Select the hibernation API

Accept the server side of an inbound WebSocket with
`ctx.acceptWebSocket()` and handle wake-up events through Durable Object class
methods such as `webSocketMessage()` and `webSocketClose()`.

```ts
export class Room extends DurableObject {
  async fetch(): Promise<Response> {
    const [client, server] = Object.values(new WebSocketPair());
    this.ctx.acceptWebSocket(server);
    return new Response(null, { status: 101, webSocket: client });
  }

  webSocketMessage(
    ws: WebSocket,
    message: string | ArrayBuffer,
  ) {
    ws.send(message);
  }
}
```

The standard `ws.accept()` path does not enable hibernation. Only inbound
WebSockets served by the Durable Object can hibernate; outbound WebSockets
cannot.

## Persist bounded per-socket state

`serializeAttachment(value)` saves a structured-clone snapshot with the socket
across hibernation. The limit is 16,384 bytes. Mutating the original value does
not update the snapshot; call the method again.

`deserializeAttachment()` returns the latest snapshot or `null`. The
attachment is lost when either side closes, so larger or longer-lived state
belongs in Durable Object storage.

```ts
server.serializeAttachment({ userId });

webSocketMessage(
  ws: WebSocket,
  message: string | ArrayBuffer,
) {
  const state = ws.deserializeAttachment() as {
    userId: string;
  };
  ws.send(`${state.userId}: ${message}`);
}
```

## Tag accepted sockets

`acceptWebSocket()` may associate at most 10 tags of at most 256 characters
each with a socket. `getWebSockets(tag)` filters attached sockets, and
`getTags(ws)` retrieves a socket's tags. `getTags()` throws if that socket was
not accepted by the Durable Object.

```ts
this.ctx.acceptWebSocket(server, ["room:42"]);
const roomSockets = this.ctx.getWebSockets("room:42");
const tags = this.ctx.getTags(server);
```

## Handle keepalives without waking the object

`setWebSocketAutoResponse()` installs one request/response pair for the
runtime to handle without waking the object. Each string is limited to 2,048
characters. Omitting the pair clears the configuration.

```ts
this.ctx.setWebSocketAutoResponse(
  new WebSocketRequestResponsePair("ping", "pong"),
);
```

`getWebSocketAutoResponseTimestamp(ws)` still reports the most recent
auto-response time for that socket after the pair is cleared.

## Bound event execution time

`setHibernatableWebSocketEventTimeout(milliseconds)` limits a hibernatable
event to at most 604,800,000 ms, or seven days. Passing `0` or omitting the
value clears the limit. The getter returns the current millisecond value or
`null`.

```ts
this.ctx.setHibernatableWebSocketEventTimeout(30_000);
```

## Account for the closing handshake

A server-closed socket may remain in `CLOSING` and continue to appear in
`getWebSockets()` until its peer completes the close handshake. With
compatibility date `2026-04-07` or later, the default
`web_socket_auto_reply_to_close` flag automatically completes the handshake,
so sockets reach `CLOSED` sooner.
