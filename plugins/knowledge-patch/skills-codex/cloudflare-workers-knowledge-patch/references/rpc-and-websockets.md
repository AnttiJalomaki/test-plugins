# RPC and WebSockets

Use this reference for RPC entrypoints, remote capabilities, stub and stream
ownership, pipelining, forwarding, placement, and WebSocket compatibility.

## RPC enablement and entrypoints

Workers RPC requires compatibility date `2024-04-03` or later, or the `rpc`
compatibility flag. Public methods on a `WorkerEntrypoint` are callable
through a Service Binding; Durable Object methods are callable through that
object's binding. Calls are asynchronous to the caller and must be awaited
even if the callee method is synchronous.

```ts
export default class extends WorkerEntrypoint {
  add(a: number, b: number) {
    return a + b;
  }
}

const sum = await env.MATH_SERVICE.add(1, 2);
```

From `2025-11-17`, `ctx.exports` automatically supplies loopback bindings for
a Worker's top-level exports. This permits same-Worker `WorkerEntrypoint`
calls without an explicit service binding.

## Functions and `RpcTarget` capabilities

A function sent or returned over RPC becomes a callable stub. It runs the
original function in the originating Worker, allowing callback parameters and
stateful returned closures.

An application-defined class must extend `RpcTarget` to cross RPC. Its methods
remain remote calls, and reading a public property requires `await`.

```ts
class Counter extends RpcTarget {
  #value = 0;
  increment() {
    return ++this.#value;
  }
  get value() {
    return this.#value;
  }
}

export default class extends WorkerEntrypoint {
  create() {
    return new Counter();
  }
}

using counter = await env.COUNTERS.create();
await counter.increment();
const value = await counter.value;
```

A plain object containing five functions creates five stubs, while one
`RpcTarget` containing five methods creates one stub. Plain-object
non-function fields are sent immediately rather than read on demand.
Instances of other application-defined classes are rejected instead of being
flattened into plain objects.

## Pipelining and ownership

RPC calls return custom thenables that also act as stubs. Calling the eventual
result before awaiting it pipelines the calls into one round trip. If the
first call fails, all pipelined calls fail with the same exception.

```ts
using pendingCounter = env.COUNTERS.create();
await pendingCounter.increment();
```

From `2026-01-20`, stubs embedded in call parameters are duplicated rather
than transferred, so forwarding a parameter does not implicitly dispose the
caller's stub. A received parameter stub is still disposed when the call
returns. A callee retaining it must store `stub.dup()`.
`rpc_params_transfer_stubs` restores transfer ownership.

RPC carries byte-oriented `ReadableStream` and `WritableStream` values,
`Request`, and `Response`, with body flow control for payloads larger than the
serialized message limit. Sending transfers ownership. Clone a request or
response, or `tee()` a readable stream, if the caller still needs a usable
copy.

## Forwarding and placement

A Worker can forward a stub received from one service to a different service.
The recipient can then call the original target through the introducing
Worker without a direct binding:

```ts
using counter = env.COUNTERS.create();
await env.CONSUMER.useCounter(counter);
```

The proxy connection survives only for the current execution contexts and
cannot be persisted.

RPC ignores Smart Placement. A Worker invoked through another Worker's
Service Binding runs locally on the caller's machine, not at its configured
placement.

## Size limits and failures

The JSRPC serialized message limit and the maximum WebSocket message size are
both 32 MiB. Use RPC streaming for payloads beyond the serialized limit.

WebSocket client failures surface as catchable JavaScript exceptions rather
than internal errors.

## WebSocket close handling

From `2026-03-03`, `WebSocket.close()` throws a `SyntaxError`
`DOMException` when the UTF-8 encoding of the reason exceeds 123 bytes.
Measure bytes, not JavaScript characters.

From `2026-03-10`, receipt of a Close frame automatically sends the reply and
sets `readyState` to `CLOSED` before the `close` event. Calling `close()` in
the handler is unnecessary and ignored.

A proxy that needs the older half-open phase must use:

```ts
ws.accept({ allowHalfOpen: true });
```

Constructor-created WebSockets cannot use this option. Obtain the WebSocket
through an upgrade `fetch()` when half-open behavior is required.

## WebSocket binary messages

From `2026-03-17`, `WebSocket.binaryType` defaults to `"blob"` and binary
messages arrive as `Blob`, replacing the historical `"arraybuffer"` default.
Set `ws.binaryType = "arraybuffer"` before `accept()` to retain `ArrayBuffer`.
Durable Object hibernatable WebSocket handlers continue to receive
`ArrayBuffer` regardless of this gate.
