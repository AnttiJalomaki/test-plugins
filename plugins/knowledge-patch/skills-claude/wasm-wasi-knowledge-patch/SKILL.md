---
name: wasm-wasi-knowledge-patch
description: WebAssembly / WASI
version: Wasm 3.0 / WASI 0.3.0
license: MIT
metadata:
  author: Nevaberry
---


# WebAssembly and WASI Compatibility Guide

Use this skill when authoring WIT, composing components, targeting current core
WebAssembly, or moving a WASI component between the synchronous 0.2 interfaces
and the native-async 0.3 interfaces.

## Reference map

| Reference | Topics |
| --- | --- |
| [WIT language and composition](references/wit-language.md) | Identifiers, documentation, result shorthand, types, resources, interface reuse, worlds, and packages |
| [Core WebAssembly](references/core-webassembly.md) | Evergreen standard, standardized proposals, 64-bit and multiple memories, exceptions, determinism, annotations, and JavaScript strings |
| [Async component model](references/async-component-model.md) | Streams, futures, ownership, scheduling, coroutine styles, and completion-oriented I/O |
| [WASI interface migration](references/wasi-interface-migration.md) | 0.2 interoperation, interface replacements, HTTP, sockets, clocks, CLI, and initial tooling |

## Start with the compatibility boundary

Before changing code, identify all three boundaries:

1. Is the artifact a core Wasm module or a component?
2. Is its interface written against WASI 0.2 or WASI 0.3?
3. Does it need synchronous host calls, or composable async across components?

Do not treat WASI 0.3 as a mandatory replacement for 0.2. Hosts can keep
exposing 0.2, and a 0.3 runtime can translate 0.2 imports to native 0.3
primitives. Migrate when the reshaped interfaces or cross-component async are
valuable.

For a native-async component, audit ownership as well as types:

- `stream<T>` and `future<T>` are Canonical ABI values, not resources.
- Passing either across a component boundary transfers ownership.
- Neither value can be borrowed.
- The host schedules suspended work across the complete component graph.

## High-impact WASI 0.3 changes

### Replace readiness-shaped I/O

There is no WASI 0.3 release of `wasi:io`. Translate its resource-oriented
patterns as follows:

| WASI 0.2 shape | WASI 0.3 shape |
| --- | --- |
| `pollable` | `future<T>` |
| `input-stream` | `stream<u8>` |
| host-owned `output-stream` | caller-supplied `stream<u8>` |
| poll an operation | await its future |
| call `subscribe()` | return a future from the operation |

Keep streamed data and terminal completion separate for read-like work:

```wit
read-via-stream: func(offset: filesize)
    -> tuple<stream<u8>, future<result<_, error-code>>>;
```

The completion future resolves even if the caller stops consuming or drops the
data stream. Use this shape for files, stdin, TCP receives, and directory
listings.

Reverse the direction for writes. The guest supplies bytes and waits for the
host to finish consuming them:

```wit
write-via-stream: func(data: stream<u8>)
    -> future<result<_, error-code>>;
```

This pattern applies to stdout, stderr, filesystem writes, and TCP sends.

### Collapse split operations

Replace `start-foo`/`finish-foo` plus an intervening `pollable` with one call.
Make genuinely host-suspending work an `async func`:

```wit
connect: async func(remote-address: ip-socket-address)
    -> result<_, error-code>;
```

An operation split only to permit nonblocking dispatch, such as bind or listen,
may become a plain `func`.

### Reshape HTTP roles

The HTTP surface uses `request` and `response` rather than the former family of
nine request, response, body, out-parameter, and future resources. Bodies are
`stream<u8>` values, trailers arrive through a future, and a handler returns
its response directly:

```wit
handle: async func(request: request) -> result<response, error-code>;
```

Use the `service` world for an endpoint that imports the HTTP `client` and
exports the incoming `handler`. Use `middleware` when the component also
imports a downstream `handler`; it succeeds the 0.2 `proxy` role.

### Update capabilities and names

- Grant socket network access at the world level; do not pass a `network`
  resource through bind, connect, or name lookup.
- Use the consolidated socket `types` and `ip-name-lookup` interfaces.
- Consume TCP listeners as `stream<tcp-socket>` rather than through a separate
  accept loop.
- Expect some filesystem methods to be `async func`.
- Rename clock `wall-clock` to `system-clock` and `datetime` to `instant`.
- Share CLI types through `wasi:cli/types`.

## Native async rules

Declare suspension and incremental values directly in WIT:

```wit
interface handler {
    handle: async func(request: string) -> result<string, u32>;
    body: func() -> tuple<stream<u8>, future<result>>;
}
```

The host owns one event loop for all composed components. Completing a future
schedules its awaiting task even when the value has crossed several component
boundaries. The producer can be the host, another component, or the same
component.

The ABI reports completion rather than readiness. A port that fundamentally
needs `epoll`- or `kqueue`-style readiness can emulate that layer, but the
component interface should retain completion semantics.

Bindings may implement stackful and stackless coroutines side by side. For
example, Go can expose synchronous-looking calls and blocking stream operations
while the runtime parks only the calling goroutine at an ABI boundary.

## WIT authoring quick reference

### Names, comments, and results

- Write ASCII kebab-case identifiers. Each hyphen-delimited word must be all
  lowercase or all uppercase.
- Prefix a keyword used as a name with `%`.
- Use `///` or `/** ... */` to document the following item.
- Ordinary `/* ... */` comments may nest.
- `result<T>` omits the error payload.
- `result<_, E>` omits the success payload.
- Bare `result` omits both payloads.
- Do not rely on a NaN payload surviving an interface boundary; WIT logically
  exposes one `nan` value.

### Types and resources

User-defined records and variants cannot declare type parameters. Only built-in
types such as `list<T>`, `option<T>`, and `result<T, E>` are generic.

A resource may declare at most one `constructor`. Its ordinary methods receive
an implicit borrowed `self`; `static func` members receive no `self`.
`borrow<resource>` loans a handle for one call. Passing an owned resource handle
transfers the duty to destroy it eventually.

### Reuse and composition

Import interface types with mandatory braces, even for one type:

```wit
use types.{point};
```

The referenced interface can live in another peer `.wit` file in the same
package. Worlds can import or export whole interfaces or individual functions,
declare inline interfaces, and `include` another world. Refer to an external
interface as `package/interface`; package resolution belongs to tooling.

A package ID is `namespace:name` with optional `@semver`. Peer `.wit` files in
one directory can form one package. Only one file needs the package declaration;
if several repeat it, every declaration must match.

## Core WebAssembly quick reference

Treat the Candidate Recommendation as an evergreen standard whose hosted
specification receives fixes and format updates in place.

WebAssembly 2.0 standardized SIMD, bulk memory and table operations, multi-value
results and block inputs, typed tables and references, non-trapping float-to-int
conversions, and sign extension while remaining backward-compatible.

When targeting WebAssembly 3.0, account for:

- `i64` address types for memories and tables;
- multiple directly addressable memories and cross-memory copies;
- native exception tags, throws, and handler dispatch;
- an opt-in deterministic profile for NaN generation and relaxed SIMD edges;
- generic, optionally ignored text annotations;
- importable JavaScript string builtins;
- garbage collection, typed references, tail calls, and relaxed SIMD.

An `i64` memory has a theoretical 16 EiB address space, but the Web embedding
limits a 64-bit memory to 16 GiB. Do not infer practical allocation limits from
the core address width.

## Initial WASI 0.3 toolchain check

For the initial WASI 0.3 release line, verify:

- Wasmtime 46 or newer; WASI 0.3 and `component-model-async` are enabled by
  default.
- `wit-bindgen` 0.46 or newer with its `async` feature.
- `wkg` 0.15 or newer for 0.3 packages.
- jco's `preview3-shim` for JavaScript host bindings.

Rust builds currently need nightly because stable Rust includes a
`wasm-component-ld` too old for the 0.3 output emitted by `wit-bindgen` 0.58.
Consult the migration reference before assuming the general minimum alone is
sufficient for that Rust path.
