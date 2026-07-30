# Async Component Model

Native async declarations first appear in the `wasi-0.2-guide` material; the
ratified ownership and scheduling behavior is attributed to `wasi-0.3.0`.

## Value forms

WIT uses three native async forms:

- `stream<T>` represents ordered values produced incrementally.
- `future<T>` represents one value delivered later.
- `async func` marks a call that can suspend.

Streams and futures are Canonical ABI values. They can appear as parameters and
results and can be forwarded through component boundaries.

```wit
interface handler {
    handle: async func(request: string) -> result<string, u32>;
    body: func() -> tuple<stream<u8>, future<result>>;
}
```

Bindings should expose these operations through the host language's customary
async form.

## Ownership across boundaries

Each `stream<T>` and `future<T>` acts as an owned handle. Passing one across a
component boundary transfers ownership to the callee.

Unlike a resource handle, a stream or future cannot be borrowed. Do not model a
shared reference to either value in a binding.

## Host-wide scheduling

The host operates one event loop shared by all composed components. Delivering
a future value schedules its waiting task even when the future has traveled
through multiple component boundaries.

The producer may be:

- the host;
- another component;
- the same component as the consumer.

This host-wide rule lets independently composed components suspend and resume
without assigning an event loop to each component.

## Completion, not readiness

The async ABI is completion-based. Operations deliver their result when work
completes rather than merely reporting that a file descriptor is ready.

Software whose internal design requires an `epoll`- or `kqueue`-style readiness
layer can emulate one during a port, but that emulation sits above the
completion-oriented ABI.

## Stackful and stackless bindings

Stackful and stackless coroutines can coexist on the async ABI. A language
binding is not forced into one implementation strategy.

Go bindings can expose synchronous-looking functions and blocking stream
operations. At an ABI boundary, the runtime parks only the calling goroutine
and resumes it when the stream becomes ready.

## Separate read data from completion

A read-like operation returns a data stream and a separate future for its
terminal result:

```wit
read-via-stream: func(offset: filesize)
    -> tuple<stream<u8>, future<result<_, error-code>>>;
```

The future resolves even when the caller consumes only part of the stream or
drops it immediately. The caller can therefore learn whether the operation
succeeded without draining all data.

This shape is used for:

- filesystem reads;
- stdin;
- TCP receives;
- directory listings.

## Reverse write data flow

For a write, the guest passes a byte stream to the host and receives a future
that completes after the host consumes it:

```wit
write-via-stream: func(data: stream<u8>)
    -> future<result<_, error-code>>;
```

This replaces pushing bytes through a host-owned output-stream resource. The
pattern applies to stdout, stderr, filesystem writes, and TCP sends.

## Stable compatibility line

WASI 0.3.0 is a stable, ratified release. Components compiled for 0.3.0 are
guaranteed to remain usable as later 0.3.x patch releases ship.
