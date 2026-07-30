# WebSocket hibernation

## Accept sockets through the Durable Object context

Accept the server side of an inbound pair with `ctx.acceptWebSocket()`. Handle
wake-up events in Durable Object methods such as `webSocketMessage()` and
`webSocketClose()`.

The standard `ws.accept()` path does not enable hibernation. Only inbound
WebSockets served by the Durable Object can hibernate; outbound WebSocket
connections cannot.

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

## Persist small per-connection attachments

`serializeAttachment(value)` saves a structured-clone snapshot with the socket
across hibernation, up to 16,384 bytes. Later mutation of the original value is
not persisted unless `serializeAttachment()` is called again.

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

## Tag hibernating sockets

`acceptWebSocket()` accepts at most 10 tags, each no longer than 256
characters. `getWebSockets(tag)` filters the attached sockets.
`getTags(ws)` returns the tags and throws if that socket was not accepted by
the Durable Object.

```ts
this.ctx.acceptWebSocket(server, ["room:42"]);
const roomSockets = this.ctx.getWebSockets("room:42");
const tags = this.ctx.getTags(server);
```

## Handle keepalive without waking the object

`setWebSocketAutoResponse()` installs one request/response pair that the
runtime handles without waking a hibernating object. The request and response
strings are each limited to 2,048 characters.

```ts
this.ctx.setWebSocketAutoResponse(
  new WebSocketRequestResponsePair("ping", "pong"),
);
```

Omitting the pair clears the auto-response. Even after it is cleared,
`getWebSocketAutoResponseTimestamp(ws)` reports the last auto-response time for
that socket.

## Limit hibernatable event execution

`setHibernatableWebSocketEventTimeout(milliseconds)` caps one event's runtime.
The maximum is 604,800,000 milliseconds, or seven days. Passing `0` or
omitting the value clears the limit. The getter returns the current
millisecond value or `null`.

```ts
this.ctx.setHibernatableWebSocketEventTimeout(30_000);
```

## Allow the close handshake to finish

A server-closed socket may remain in `CLOSING` and continue to appear in
`getWebSockets()` until the peer completes the close handshake.

At compatibility date `2026-04-07` or later, the default
`web_socket_auto_reply_to_close` flag automatically completes that handshake,
so sockets reach `CLOSED` sooner.
