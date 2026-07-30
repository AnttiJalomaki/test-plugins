---
name: wasm-wasi-knowledge-patch
description: WebAssembly / WASI
version: Wasm 3.0 / WASI 0.3.0
license: MIT
metadata:
  author: Nevaberry
---


# WebAssembly and WASI

Use this skill when authoring WIT, selecting core WebAssembly features, moving a
component from synchronous WASI interfaces to native async interfaces, or
choosing compatible tools.

Prefer the project's manifests, checked-in WIT, generated bindings, runtime
configuration, and tests when they establish behavior more specifically than
this guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [wit-language.md](references/wit-language.md) | Identifiers, comments, result shorthand, types, resources, async values, imports, worlds, and packages |
| [core-webassembly.md](references/core-webassembly.md) | Evergreen standard, standardized features, 64-bit and multiple memories, exceptions, determinism, annotations, and JavaScript strings |
| [wasi-async.md](references/wasi-async.md) | Interoperation, stream and future ownership, scheduling, reads, writes, and collapsed operations |
| [wasi-interfaces.md](references/wasi-interfaces.md) | `wasi:io`, HTTP, sockets, filesystem, clocks, CLI, service, and middleware changes |
| [toolchains.md](references/toolchains.md) | Initial runtime, binding generator, package tooling, JavaScript host, and Rust requirements |

## Breaking changes and migration decisions

### Do not look for a `wasi:io` 0.3 package

There is no 0.3 release of `wasi:io`. Translate its concepts to native async
values:

| Earlier shape | Native async shape |
| --- | --- |
| `pollable` | `future<T>` |
| `input-stream` | `stream<u8>` |
| `output-stream` | a `stream<u8>` passed into an operation |
| polling | awaiting a future |
| `subscribe()` | returning a future from the operation |

Do not mechanically preserve the earlier resource API. Reshape the operation
around data flow and completion.

### Separate read data from terminal completion

A read-like call returns both:

- a `stream<u8>` carrying data; and
- a future carrying the terminal result.

The completion future resolves even if the caller samples only part of the
stream or drops it. Callers can therefore observe success or failure without
draining all data.

```wit
read-via-stream: func(offset: filesize)
    -> tuple<stream<u8>, future<result<_, error-code>>>;
```

Use the same reasoning for stdin, TCP receive, and directory-listing APIs.

### Reverse the direction of writes

Do not request a host-owned output resource and repeatedly push bytes into it.
Pass the host a data stream and await the future that completes after the host
consumes it:

```wit
write-via-stream: func(data: stream<u8>)
    -> future<result<_, error-code>>;
```

This pattern applies to stdout, stderr, filesystem writes, and TCP sends.

### Collapse split nonblocking operations

Replace `start-foo`/`finish-foo` pairs and the intervening `pollable` with one
operation.

- Make host-suspending work such as TCP connect an `async func`.
- A split that existed only for nonblocking dispatch, such as bind or listen,
  can become an ordinary `func`.

```wit
connect: async func(remote-address: ip-socket-address)
    -> result<_, error-code>;
```

### Update HTTP roles and values

Reduce the request, response, body, out-parameter, and future resources to
`request` and `response`. Represent bodies with `stream<u8>` and trailers with
a future. Return the response directly:

```wit
handle: async func(request: request) -> result<response, error-code>;
```

Use the `service` world for an endpoint that imports the HTTP `client` and
exports the incoming `handler`. Use `middleware` when the component also
imports a downstream `handler`. The earlier `proxy` role maps to these more
explicit worlds.

### Update socket capabilities and acceptance

Do not thread a `network` resource through bind, connect, and lookup calls.
Grant network access at the world level.

Use the consolidated `types` and `ip-name-lookup` interfaces. Model TCP listen
as returning `stream<tcp-socket>` rather than requiring a separate accept loop.

### Treat stream and future handles as owned

Passing a `stream<T>` or `future<T>` across a component boundary transfers
ownership to the callee. Unlike a resource handle, neither value can be
borrowed. Design forwarding APIs and generated binding use around that
one-owner rule.

## Compatibility and adoption

WASI 0.3 is additive. A host can keep exposing 0.2, and a 0.3 runtime can
polyfill 0.2 by translating imports to native primitives at the host boundary.
Migrate when composable cross-component async or a reshaped interface is
needed; do not assume all existing components must move together.

Components compiled for the stable 0.3 line remain compatible as later 0.3.x
patch releases ship.

When adopting the initial async toolchain, check the exact runtime and binding
versions in [toolchains.md](references/toolchains.md). Rust builds have an
additional compiler-channel constraint caused by the bundled component linker.

## Native async quick reference

WIT provides three native async building blocks:

- `stream<T>` for incrementally produced, ordered values;
- `future<T>` for one value delivered later; and
- `async func` for a call that may suspend.

Streams and futures are Canonical ABI values, not resources. They can be
parameters, results, and values forwarded through composed components.
Bindings should expose the host language's ordinary async form.

The host runs one event loop for all composed components. Delivering a future
value schedules its awaiting task even when the future has crossed several
component boundaries. The producer may be the host, another component, or the
same component.

The ABI reports completion, not readiness. A readiness-oriented layer such as
one built around `epoll` or `kqueue` can be emulated for ported software.

Both stackful and stackless coroutine bindings fit the ABI. Go bindings may
offer synchronous-looking functions and blocking stream operations: only the
calling goroutine is parked at the boundary and resumed when data is ready.

## WIT authoring quick reference

### Spell identifiers and comments correctly

Identifiers use ASCII kebab-case. Every hyphen-delimited word is wholly
lowercase or wholly uppercase. Prefix a keyword used as a name with `%`.

Use `///` or `/** ... */` to document the following item. Ordinary
`/* ... */` comments can nest.

```wit
/// An interface whose escaped name would otherwise be a keyword.
interface %interface {
    HTTP-request: func();
}
```

### Use result shorthand deliberately

Either `result<T, E>` payload may be omitted:

| Form | Meaning |
| --- | --- |
| `result<T>` | no error payload |
| `result<_, E>` | no success payload |
| `result` | neither payload |

### Keep user-defined types concrete

Records and variants cannot declare type parameters. Parameterization belongs
to built-ins such as `list<T>`, `option<T>`, and `result<T, E>`.

### Make resource ownership visible

A resource has at most one constructor. An ordinary method receives an
implicit borrowed `self`; a `static func` receives no `self`.

`borrow<resource>` loans a handle only for the duration of a call. Passing an
owned resource handle transfers responsibility for eventually destroying it.

### Import types with braces

Use `use interface-name.{type-name, ...}`. The braces are required even for
one type, and the form works across peer WIT files in the same package.

## Core WebAssembly quick reference

The live core standard includes 64-bit memories and tables, multiple memories,
garbage collection, typed references, tail calls, native exception handling,
relaxed SIMD, deterministic execution, and text annotations. Its JavaScript
embedding also provides string builtins.

For a 64-bit memory or table, use `i64` as the address type. The theoretical
memory address space is 16 EiB; Web embeddings cap a 64-bit memory at 16 GiB.

Modules can define or import multiple memories and copy directly between them.
Use this for deliberately separate address spaces and for module composition
that cannot rely on a single shared memory.

Exception tags carry payload data. Handler blocks dispatch with tag/label pairs
or catch-all labels, keeping portable exception control flow inside Wasm.

The deterministic profile fixes choices left open by base floating-point NaN
and relaxed SIMD semantics, but reproducibility only holds among platforms
that implement the profile.

See [core-webassembly.md](references/core-webassembly.md) for the standardized
feature sets and the semantics of annotations and JavaScript string imports.
