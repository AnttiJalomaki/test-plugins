# WIT language and composition

## Identifiers and documentation

WIT identifiers are ASCII kebab-case. Each word separated by a hyphen must be
entirely lowercase or entirely uppercase. When a keyword is used as a name,
escape it with a `%` prefix (wasi-0.2-guide).

`///` and `/** ... */` attach documentation to the following item. Ordinary
`/* ... */` comments may be nested.

```wit
/// An interface whose escaped name would otherwise be a keyword.
interface %interface {
    HTTP-request: func();
}
```

## Result shorthands

Both payload positions of `result<T, E>` are optional (wasi-0.2-guide):

- `result<T>` omits the error payload.
- `result<_, E>` omits the success payload.
- bare `result` omits both payloads.

```wit
interface results {
    type print-result = result<_, u32>;
    type signal-result = result;
}
```

## Floating-point NaNs

Although `f32` and `f64` otherwise represent IEEE 754 values, WIT logically
has one `nan` value (wasi-0.2-guide). Do not depend on a NaN's payload bits
surviving an interface crossing.

## Generic and concrete types

User-defined records and variants cannot declare type parameters
(wasi-0.2-guide). Only built-in generic types such as `list<T>`, `option<T>`,
and `result<T, E>` can be parameterized.

## Resources

A resource can declare:

- at most one `constructor`;
- ordinary methods, each with an implicit borrowed `self`; and
- `static func` members, which have no `self`.

```wit
interface storage {
    resource blob {
        constructor(init: list<u8>);
        read: func(n: u32) -> list<u8>;
        merge: static func(lhs: blob, rhs: blob) -> blob;
    }
}
```

`borrow<resource>` loans a handle only for a call. Passing an owned resource
handle transfers the responsibility to destroy it eventually
(wasi-0.2-guide).

Do not apply resource borrowing rules to native streams and futures. Each is
an owned Canonical ABI value and cannot be borrowed; crossing a component
boundary transfers its ownership (wasi-0.3.0).

## Native async declarations

WASI 0.3 adds these WIT native async values and calls:

- `stream<T>` for ordered values produced incrementally;
- `future<T>` for one value produced later; and
- `async func` for a call that may suspend.

Streams and futures are Canonical ABI values rather than resources. They can
be parameters, results, or values forwarded across component boundaries. The
runtime schedules async calls, while bindings expose the host language's
normal async form (wasi-0.2-guide).

```wit
interface handler {
    handle: async func(request: string) -> result<string, u32>;
    body: func() -> tuple<stream<u8>, future<result>>;
}
```

## Reusing interface types

Import types from another interface with
`use interface-name.{type-name, ...}`. Braces are mandatory even when importing
one type. The same form works between `.wit` files in one package
(wasi-0.2-guide).

```wit
interface types { type point = tuple<u32, u32>; }
interface canvas {
    use types.{point};
    draw: func(at: point);
}
```

## Composing worlds

A world can:

- import or export an entire interface;
- import or export an individual function;
- declare an interface inline; and
- `include` another world to acquire all of its imports and exports.

Name an external interface with `package/interface` syntax. WIT deliberately
leaves package resolution to tooling (wasi-0.2-guide).

```wit
world diagnostics { export report: func(message: string); }
world proxy {
    export wasi:http/incoming-handler;
    import wasi:http/outgoing-handler;
    include diagnostics;
}
```

## Multi-file packages

A package identifier is `namespace:name`, optionally followed by `@semver`.
A package can span peer `.wit` files in the same directory. Only one file must
contain the package declaration; if several contain it, every declaration must
match (wasi-0.2-guide).

```wit
package documentation:http@1.0.0;
```
