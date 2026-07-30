# PHP, Ruby, and Rust

Use this reference for PHP runtime and JSON migrations, Ruby and JRuby runtime
support, RBS generation, and Rust generated API or crate-version changes.

## PHP runtime baseline and renamed types

`34.0-migration` raises the PHP runtime minimum to PHP 8.2.

The `34.0-announcement` removes legacy type names:

| Removed | Replacement |
| --- | --- |
| `Google\Protobuf\Field_Kind` | `Google\Protobuf\Field\Kind` |
| `Google\Protobuf\Field_Cardinality` | `Google\Protobuf\Field\Cardinality` |
| `Google\Protobuf\Internal\RepeatedField` | `Google\Protobuf\RepeatedField` |

Update imports, type tests, reflection, and serialized class-name references
before upgrading the package.

## PHP JSON parsing and serialization

`30.0-migration` makes PHP and Ruby reject nonnumeric strings such as `""`,
`"12abc"`, and `"abc"` for numeric fields during JSON parsing. The same inputs
only warned in v29.x.

PHP becomes stricter again in `34.0-announcement`. JSON parsing rejects:

- out-of-range numbers;
- non-integer numeric values for integer fields;
- duplicate members of a oneof; and
- non-string values for string fields.

JSON serialization rejects `Infinity` and `NaN` as numeric values. Add failure
tests for each class of invalid input instead of relying on coercion.

`34.0` adds an option to emit fields whose values equal their defaults during
PHP JSON serialization. Select it only when the wire consumer requires
explicit default-valued members.

## PHP defaults, setters, and presence

Since `34.0-announcement`, PHP honors proto2 and Editions scalar-field defaults
instead of silently ignoring them. Pure-PHP type checking also aligns with
upb-PHP, including rejection of `null` for string fields.

Generated setters in `34.0` carry PHP type hints, and redundant `GPBUtil`
checks are removed. Reflection, subclasses, decorators, or callers that
depended on untyped signatures must adapt to the declared parameter types.

PHP descriptors implement `hasPresence()` in `34.0`; the broken
`hasOptionalKeyword()` helper is removed. Use presence semantics instead of
trying to reconstruct them from an optional keyword.

The earlier `FieldDescriptor::getLabel()` deprecation from `31.0` culminates in
removal in `34.0-migration`. Use `isRepeated`, `isRequired`, and `hasPresence`
for the corresponding semantic questions.

## PHP recursion limits

`36.0-rc1` adds a configurable `recursion_limit` to both `mergeFromString` and
`serializeToString`. Set a finite limit for untrusted or potentially deep
messages and handle the limit error during both parse and serialization paths.

## Ruby and JRuby runtime changes

`30.0` switches JRuby to the FFI implementation by default. Applications that
depended on the previous implementation must test native boundaries and
runtime-specific behavior. This does not cause a Ruby package-major bump
because JRuby is not officially supported.

The support baseline evolves as follows:

- `31.0` removes Ruby 3.0 and requires Ruby 3.1 or newer.
- `34.0` adds Ruby 4.0 support.
- `36.0-rc1` removes Ruby 3.1 support, requiring deployments on that
  interpreter to upgrade.

JRuby remains best-effort under `release-lifecycle`; the target is the latest
JRuby compatible with the minimum supported Ruby version.

## Ruby validation and typing

`30.0-migration` surfaces UTF-8 enforcement errors earlier when a protobuf
`string` contains invalid UTF-8. It also changes Ruby JSON numeric parsing from
warning to rejection for nonnumeric strings. Ensure callers handle both
failures at their new parse or assignment point.

`34.0` adds Ruby RBS generation. Enable it when generated protobuf types should
participate in RBS-based type checking, and include the generated declarations
in the type checker's inputs.

## Rust release matching

Rust follows the exact-version rule in `release-lifecycle`: generated code and
runtime releases must match rather than using the normal major-V through V+1
window. Update the generator, regenerate, and update the runtime crate as one
operation.

For minor 36 and later, `36.0-rc1` removes the `-release` suffix from Crates.io
versions. Update dependency pins, package matching, and release automation to
use the unsuffixed version.

## Rust traits and optional accessors

`34.0` adds a `Send` bound to the `MessageMut` trait. Implementations and
generic users must now be cross-thread sendable.

`35.0` changes generated `_opt()` accessors to return standard `Option` instead
of `protobuf::Optional`. Update named types, conversions, trait bounds, and
pattern matches that refer to the old wrapper.

The same release revises generic field and map traits:

- `Singular` identifies values allowed as simple fields.
- `ProxiedInMapValue` is removed; use `MapValue`.
- `f32` and `f64` no longer incorrectly implement the map-key trait.

Do not preserve generic bounds that accidentally admitted floating-point map
keys.

## Rust generated naming and views

When a generated scope has direct siblings named `Xyz` and `XyzView`, the
`35.0` generator mangles the `XyzView` type. Recompile downstream source after
regeneration and use the actual emitted identifier rather than predicting it.

`ProtoStr` becomes usable in const contexts. Also, `&T` implements `AsView`
whenever `T` does, so generic view-taking code can accept references without
converting to byte slices or adding custom adapters.

## Rust repeated messages

`36.0-rc1` adds mutable iteration over handles in repeated message-typed
fields. Callers can mutate each existing element in place while traversing the
field rather than indexing and reacquiring a mutable handle.

Respect Rust borrowing rules around the iterator; do not structurally modify
the repeated field while retaining an element handle.

## Upgrade checks

- Run PHP parsing tests for invalid types, ranges, numeric spellings, duplicate
  oneof members, non-finite values, and recursion limits.
- Verify PHP defaults, typed setter reflection, and field presence.
- Test Ruby invalid UTF-8 and JSON errors on every supported interpreter.
- Include generated RBS files in type-checking and packaging.
- Regenerate Rust with exactly the runtime release in the manifest.
- Compile generic Rust code against `Send`, `Option`, `Singular`, `MapValue`,
  and the revised map-key bounds.
- Inspect mangled Rust type names and exercise mutable repeated-message
  iteration.
