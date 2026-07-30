# Rust and Go generated APIs

Source batches represented here: 34.0, 35.0, 36.0-rc1,
edition-2026-guide.

## Rust version pairing and crate releases

Rust generated code and runtime versions must match exactly. Do not apply the
V-to-V+1 compatibility window used by most other protobuf runtimes.

Beginning with protobuf minor 36, Rust Crates.io versions drop the `-release`
suffix. Update dependency constraints, publication checks, and release
automation to use the unsuffixed version.

## Rust ownership, mutation, and views

`MessageMut` includes a `Send` bound in v34. Implementations and generic code
must be safe to transfer across threads.

Generated `_opt()` accessors return the standard `Option` type in v35 instead
of `protobuf::Optional`. Update named types, conversions, trait bounds, and
implementations that depend on the old wrapper.

When a generated scope has sibling types named `Xyz` and `XyzView`, the v35
generator mangles the generated `XyzView` identifier. Recompile downstream
code against the regenerated name.

Rust adds the `Singular` trait for types allowed as simple fields and revises
map traits. Replace `ProxiedInMapValue` with `MapValue`; do not rely on `f32`
or `f64` satisfying the map-key trait.

`ProtoStr` can be used in const contexts in v35. `&T` implements `AsView`
whenever `T` does, so generic view consumers can accept references directly.

In 36.0-rc1, generated repeated message fields support iteration over mutable
handles, allowing in-place element mutation during traversal.

## Go Editions API shape

`features.(pb.go).api_level` defaults to `API_OPEN` in Edition 2023 and
`API_OPAQUE` in Editions 2024 and 2026. Opaque generation hides struct fields
behind accessors. Select:

- `API_OPEN` to preserve direct generated-field access;
- `API_HYBRID` to expose both fields and accessors while migrating;
- `API_OPAQUE` for the newer accessor-only shape.

Example:

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).api_level = API_HYBRID;
```

## Go enum-prefix stripping

Edition 2024 and newer support `features.(pb.go).strip_enum_prefix` at file,
enum, or enum-value scope:

- `STRIP_ENUM_PREFIX_KEEP` preserves existing generated names;
- `STRIP_ENUM_PREFIX_GENERATE_BOTH` emits migration-compatible names;
- `STRIP_ENUM_PREFIX_STRIP` removes the repeated enum-name prefix.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).strip_enum_prefix = STRIP_ENUM_PREFIX_STRIP;
```
