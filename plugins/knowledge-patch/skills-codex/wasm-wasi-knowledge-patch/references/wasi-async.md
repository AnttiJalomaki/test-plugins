# WASI native async and data flow

## Interoperation and migration scope

WASI 0.3 is additive rather than a forced migration (wasi-0.3-guide). Hosts can
continue exposing 0.2. A 0.3 runtime can polyfill 0.2 by translating the older
imports to native 0.3 primitives at the host boundary.

Migration is primarily needed for composable async across components or for
interfaces reshaped in 0.3. Components compiled for stable 0.3.0 are
guaranteed to continue working with later 0.3.x patch releases
(wasi-0.3.0).

## Ownership of streams and futures

Each `stream<T>` and `future<T>` is an owned handle (wasi-0.3.0). Passing it
across a component boundary transfers ownership to the callee. Unlike a
resource handle, it cannot be borrowed.

Account for the transfer when forwarding a stream or future through several
components. Do not retain a second logical owner in the caller.

## Host-wide scheduling

One host event loop serves all composed components (wasi-0.3.0). Delivering a
future value schedules its awaiting task even if the value has crossed
multiple component boundaries. Its producer can be:

- the host;
- another component; or
- the same component.

The ABI is completion-based rather than readiness-based. When porting software
whose architecture requires `epoll`- or `kqueue`-style readiness, emulate that
layer over completion events.

## Stackful and stackless bindings

The async ABI supports stackful and stackless coroutines together
(wasi-0.3.0). Go bindings can expose synchronous-looking functions and
blocking stream operations. At an ABI boundary, the runtime parks only the
calling goroutine and resumes it when the stream becomes ready.

## Read data and completion independently

Read-like operations return a data stream and a separate terminal-result
future (wasi-0.3-guide):

```wit
read-via-stream: func(offset: filesize)
    -> tuple<stream<u8>, future<result<_, error-code>>>;
```

The future resolves even when the caller reads only some of the stream or
drops it. A caller does not have to drain the data to learn whether the
operation succeeded.

Use this shape for:

- filesystem reads;
- stdin;
- TCP receives; and
- directory listings.

## Pass write data into the operation

Writing reverses the earlier output-resource direction (wasi-0.3-guide). The
guest supplies a `stream<u8>` and receives a future that resolves after the
host consumes it:

```wit
write-via-stream: func(data: stream<u8>)
    -> future<result<_, error-code>>;
```

This shape covers stdout, stderr, filesystem writes, and TCP sends. Do not
design the native API around a host-owned `output-stream` into which the guest
pushes chunks.

## Collapse start, poll, and finish

Turn a 0.2 `start-foo`/`finish-foo` pair and its intervening `pollable` into a
single operation (wasi-0.3-guide).

Use `async func` when the host operation can suspend, as with TCP connect:

```wit
connect: async func(remote-address: ip-socket-address)
    -> result<_, error-code>;
```

When the split existed only to permit nonblocking dispatch, the replacement
can be an ordinary `func`. Bind and listen are examples of this case.
