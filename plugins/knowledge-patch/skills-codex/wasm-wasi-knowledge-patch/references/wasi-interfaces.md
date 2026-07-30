# WASI interface migration

## Replace `wasi:io` concepts

The `wasi:io` package has no 0.3 release (wasi-0.3-guide). Map its concepts as
follows:

| `wasi:io` concept | Native replacement |
| --- | --- |
| `pollable` | `future<T>` |
| `input-stream` | `stream<u8>` |
| `output-stream` | a `stream<u8>` passed into a call |
| polling | awaiting a future |
| `subscribe()` | returning a future from the operation |

Pair this mapping with the data-flow and completion patterns in
[wasi-async.md](wasi-async.md).

## HTTP values and handler

The reshaped `wasi:http` reduces nine request, response, body, out-parameter,
and future resources to two principal values: `request` and `response`
(wasi-0.3-guide).

- Bodies are `stream<u8>`.
- Trailers are delivered through a future.
- A handler returns its response directly.

```wit
handle: async func(request: request) -> result<response, error-code>;
```

## HTTP worlds

The `service` world imports the HTTP `client` and exports the incoming
`handler` (wasi-0.3.0):

```wit
world service {
    import client;
    export handler;
}
```

The `middleware` world includes `service` and imports a downstream `handler`.
It is the successor to the 0.2 `proxy` world:

```wit
world middleware {
    include service;
    import handler;
}
```

Choose `service` for a terminal HTTP service. Choose `middleware` when the
component must call the next handler and also expose a handler upstream.

## Socket capabilities and interfaces

The 0.3 socket design removes the `network` resource previously threaded
through bind, connect, and name-lookup operations (wasi-0.3-guide). Network
access is granted at the world level instead.

The former seven socket interfaces consolidate into:

- `types`; and
- `ip-name-lookup`.

TCP `listen` returns `stream<tcp-socket>` directly. Do not add a separate
accept loop resource around that stream.

## Filesystem, clocks, and CLI

Additional interface changes in wasi-0.3-guide are:

- some filesystem methods become `async func`;
- `wasi:clocks/wall-clock` is renamed to `system-clock`;
- the clocks `datetime` type is renamed to `instant`; and
- CLI interfaces share the new `wasi:cli/types` interface.

Update imports, generated type names, and async call sites together so that
bindings do not retain a mixture of old and new interface shapes.
