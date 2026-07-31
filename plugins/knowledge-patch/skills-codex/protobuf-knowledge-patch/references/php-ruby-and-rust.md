# PHP, Ruby, and Rust

## PHP runtime baseline and type names

PHP requires 8.2 or newer from the 34.0 migration. (34.0-migration)

Replace removed runtime types at the v34 boundary:

| Removed | Replacement |
| --- | --- |
| `Google\Protobuf\Field_Kind` | `Google\Protobuf\Field\Kind` |
| `Google\Protobuf\Field_Cardinality` | `Google\Protobuf\Field\Cardinality` |
| `Google\Protobuf\Internal\RepeatedField` | `Google\Protobuf\RepeatedField` |

(34.0-announcement)

Generated PHP setters now have PHP type hints and redundant `GPBUtil` checks
are removed. Reflection, subclasses, or callers that assumed untyped signatures
may require changes. Pure-PHP type checks now match upb-PHP, including rejection
of `null` for string fields. (34.0, 34.0-announcement)

## PHP descriptors, defaults, and JSON

The runtime now honors proto2 and Editions scalar defaults instead of ignoring
them. Field descriptors implement `hasPresence()`; the broken
`hasOptionalKeyword()` is removed, and `getLabel()` was removed in the 34.0
migration. Use semantic presence queries. (34.0-announcement, 34.0,
34.0-migration)

Ruby and PHP reject nonnumeric strings such as `""`, `"12abc"`, and `"abc"`
for numeric JSON fields; these cases only warned in v29.x.
(30.0-migration)

The v34 PHP JSON parser additionally rejects out-of-range values, noninteger
numbers for integer fields, duplicate oneof fields, and non-string values for
string fields. Serialization rejects numeric `Infinity` and `NaN`. PHP also
adds a JSON option that emits fields whose values equal their defaults.
(34.0-announcement, 34.0)

## Ruby and JRuby

Ruby requires 3.1 or newer from 31.0. The runtime adds Ruby 4.0 support at
34.0. (31.0, 34.0)

The JRuby runtime uses its FFI implementation by default from 30.0. This can
break applications that depended on the earlier implementation. JRuby remains
best-effort rather than officially supported, so this change did not trigger a
Ruby major-version bump. (30.0)

Ruby surfaces invalid-UTF-8 errors earlier for protobuf `string` fields. It
also shares the stricter numeric-string JSON parsing described above.
(30.0-migration)

Ruby code generation can emit RBS files from 34.0, allowing generated message
types to participate in RBS-based type checking. (34.0)

## Rust version matching and sendability

Rust requires generated code and the runtime to match at the exact release,
not merely the same major. Regenerate and update the crate together.
(release-lifecycle)

At 34.0, `MessageMut` gains a `Send` bound. Implementations and generic code
using the trait must be cross-thread sendable. (34.0)

## Rust generated accessor changes

At 35.0, generated `_opt()` accessors return the standard `Option` rather than
`protobuf::Optional`. Update named types, conversions, bounds, and helper
implementations that refer to the old wrapper. (35.0)

When direct siblings `Xyz` and `XyzView` exist in one generated scope, the
generator now mangles the `XyzView` type. Regeneration can therefore change a
previously referenced identifier. (35.0)

## Rust generic field and view traits

The 35.0 Rust API adds `Singular` for types permitted as simple fields and
revises map traits. Replace `ProxiedInMapValue` with `MapValue`; do not treat
`f32` or `f64` as map-key types. (35.0)

`ProtoStr` is usable in const contexts, and `&T` implements `AsView` whenever
`T` does. Generic view-taking code can accept references without converting to
byte slices or adding adapters. (35.0)
