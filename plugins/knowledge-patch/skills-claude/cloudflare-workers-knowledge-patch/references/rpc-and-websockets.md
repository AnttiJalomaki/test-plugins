# RPC and WebSockets

## RPC enablement and entrypoints

Workers RPC requires compatibility date `2024-04-03` or later, or the `rpc`
compatibility flag.

Public methods on `WorkerEntrypoint` can be called through a Service Binding.
Durable Object methods can be called through a binding to that object. A
synchronous callee still appears asynchronous to its caller and must be
awaited.

```ts
// Service Worker
export default class extends WorkerEntrypoint {
  add(a: number, b: number) {
    return a + b;
  }
}

// Calling Worker, through the MATH_SERVICE binding
const sum = await env.MATH_SERVICE.add(1, 2);
```

From `2025-11-17`, `ctx.exports` automatically provides loopback bindings for a
Worker's top-level exports, allowing calls to its own `WorkerEntrypoint`
exports without an explicit service binding.

## Functions and `RpcTarget` capabilities

A function sent or returned over RPC becomes a callable stub. Invoking it runs
the original function in its originating Worker, so arguments can be callbacks
and returned closures can retain state.

An application-defined class must extend `RpcTarget` to cross RPC. Its methods
remain remote calls, and public-property reads require `await`.

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

A plain object with five functions creates five stubs; one `RpcTarget` with five
methods creates one stub. Plain-object non-function fields transmit
immediately. Instances of other application-defined classes are rejected
rather than flattened.

## Promise pipelining

RPC calls return custom thenables that also act as stubs. Omitting an
intermediate `await` allows a call on the eventual result to travel with the
first call in one round trip:

```ts
using pendingCounter = env.COUNTERS.create();
await pendingCounter.increment();
```

If the first call fails, its pipelined calls fail with the same exception.

## Ownership and lifetime

### Streams, requests, and responses

RPC supports byte-oriented `ReadableStream` and `WritableStream` values, plus
`Request` and `Response`. Flow control allows bodies larger than the serialized
RPC message limit.

Sending transfers ownership. Clone a request or response, or `tee()` a readable
stream, before sending when the caller must retain a usable copy.

### Stubs in parameters

From `2026-01-20` (batch `2026`), stubs embedded in RPC call parameters are
duplicated rather than transferred. Forwarding a parameter no longer
implicitly disposes the caller's stub. `rpc_params_transfer_stubs` restores the
old ownership behavior.

A parameter stub received by the callee is still disposed when the call
returns. A callee that retains it must store `stub.dup()`.

### Forwarding between Workers

A Worker can pass a stub received from one service to another. The recipient
can call the original target through the introducing Worker without a direct
binding.

```ts
using counter = env.COUNTERS.create();
await env.CONSUMER.useCounter(counter);
```

The proxy connection lasts only for the current execution contexts and cannot
be persisted.

## Placement and limits

Smart Placement does not apply to RPC. A Worker reached through another
Worker's Service Binding runs locally on the caller's machine, not at its
configured placement.

The JSRPC serialized-message limit is 32 MiB. Stream payloads when data can
exceed it.

## WebSocket limits and errors

The maximum WebSocket message size is 32 MiB. WebSocket client failures surface
as catchable JavaScript exceptions instead of internal errors (batch `2025`).

From `2026-03-03`, `WebSocket.close()` throws a `SyntaxError` `DOMException`
when the UTF-8 encoding of its reason exceeds 123 bytes. Count encoded bytes,
not JavaScript characters.

## Close handshake

From `2026-03-10`, receiving a Close frame automatically sends the reciprocal
frame and changes `readyState` to `CLOSED` before the `close` event. Calling
`close()` inside that event is unnecessary and ignored.

A proxy that needs the old half-open phase must use:

```ts
ws.accept({ allowHalfOpen: true });
```

Constructor-created WebSockets cannot use this option. Obtain the socket
through an upgrade `fetch()` when half-open handling is required.

## Binary messages

From `2026-03-17`, `WebSocket.binaryType` defaults to `"blob"` and binary
messages arrive as `Blob`. To retain `ArrayBuffer`, set the value before
accepting:

```ts
ws.binaryType = "arraybuffer";
ws.accept();
```

Durable Object hibernatable WebSocket handlers continue receiving
`ArrayBuffer` regardless of this gate.
