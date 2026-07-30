# WIT Language and Composition

These WIT details are attributed to `wasi-0.2-guide`.

## Identifier and documentation syntax

WIT identifiers use ASCII kebab-case. Every hyphen-delimited word must be
entirely lowercase or entirely uppercase. If a keyword is needed as a name,
escape it with a `%` prefix.

`///` and `/** ... */` document the item that follows. An ordinary
`/* ... */` comment is not documentation and may be nested.

```wit
/// An interface whose escaped name would otherwise be a keyword.
interface %interface {
    HTTP-request: func();
}
```

## Result shorthands

Either payload of `result<T, E>` may be omitted:

| Syntax | Success payload | Error payload |
| --- | --- | --- |
| `result<T, E>` | `T` | `E` |
| `result<T>` | `T` | none |
| `result<_, E>` | none | `E` |
| `result` | none | none |

```wit
interface results {
    type print-result = result<_, u32>;
    type signal-result = result;
}
```

## Floats and NaNs

Although `f32` and `f64` otherwise represent IEEE 754 values, WIT logically has
one `nan` value. Code must not depend on a NaN's bit-level payload surviving an
interface crossing.

## Generic and user-defined types

Records and variants cannot declare type parameters. Parameterization is
limited to WIT's built-in generic types, including `list<T>`, `option<T>`, and
`result<T, E>`.

## Resource members and ownership

A resource can have at most one `constructor`. An ordinary method has an
implicit borrowed `self`; a `static func` has no `self`.

`borrow<resource>` loans a handle only for the duration of the call. Passing an
owned resource handle transfers responsibility for eventually destroying it.

```wit
interface storage {
    resource blob {
        constructor(init: list<u8>);
        read: func(n: u32) -> list<u8>;
        merge: static func(lhs: blob, rhs: blob) -> blob;
    }
}
```

Streams and futures differ from resources: they are native Canonical ABI values
with transfer-only ownership. See
[Async component model](async-component-model.md).

## Native async declarations

WASI 0.3 adds:

- `stream<T>` for ordered values produced incrementally;
- `future<T>` for one value delivered later;
- `async func` for a call that can suspend.

Streams and futures are Canonical ABI values rather than resources. They may be
parameters or results and can be forwarded across component boundaries. The
runtime schedules async calls, while bindings expose the host language's normal
async form.

```wit
interface handler {
    handle: async func(request: string) -> result<string, u32>;
    body: func() -> tuple<stream<u8>, future<result>>;
}
```

## Reusing interface types

Import types from another interface with
`use interface-name.{type-name, ...}`. Braces are mandatory even when importing
one type. This also works when the source interface is in another `.wit` file
of the same package.

```wit
interface types { type point = tuple<u32, u32>; }
interface canvas {
    use types.{point};
    draw: func(at: point);
}
```

## World composition

A world can:

- import or export an entire interface;
- import or export an individual function;
- declare an interface inline;
- `include` another world to acquire all its imports and exports.

Name an external interface with `package/interface` syntax. WIT itself does not
define package resolution; tooling is responsible for resolving it.

```wit
world diagnostics { export report: func(message: string); }
world proxy {
    export wasi:http/incoming-handler;
    import wasi:http/outgoing-handler;
    include diagnostics;
}
```

## Multi-file packages

A package ID has the form `namespace:name`, optionally followed by `@semver`.
A package may span peer `.wit` files in one directory. Only one file needs the
package declaration. If multiple files declare it, the declarations must
match.

```wit
package documentation:http@1.0.0;
```
