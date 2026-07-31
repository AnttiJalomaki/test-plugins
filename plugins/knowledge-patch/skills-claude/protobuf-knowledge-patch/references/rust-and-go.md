# Rust and Go generated APIs

## Rust `MessageMut` sendability (`34.0`)

`MessageMut` includes a `Send` bound. Implementations and generic code must be
safe to move across threads and satisfy that bound; thread-confined wrapper
types may need redesign.

## Standard optional accessors (`35.0`)

Generated Rust `_opt()` accessors return the standard `Option` type instead of
`protobuf::Optional`. Replace named uses, conversions, and trait implementations
that depend on the old wrapper.

## Generated `XyzView` collisions (`35.0`)

If one generated scope contains direct siblings named `Xyz` and `XyzView`, the
Rust generator mangles the `XyzView` type. Regeneration can therefore change a
previously referenced identifier; consume the new generated name rather than
assuming the unmangled form.

## Generic field traits (`35.0`)

Rust adds `Singular` for types permitted as simple fields and revises its map
traits. `ProxiedInMapValue` is removed in favor of `MapValue`. `f32` and `f64`
also no longer incorrectly satisfy the map-key trait. Update generic bounds and
aliases to match the actual field category.

## View ergonomics (`35.0`)

`ProtoStr` is usable in const contexts. In addition, `&T` implements `AsView`
whenever `T` does, so generic view-taking functions can accept references
without byte-slice conversions or local adapter traits.

## Go Opaque API default (`edition-2026-guide`)

`features.(pb.go).api_level` defaults to `API_OPEN` for Edition 2023 and
`API_OPAQUE` for Editions 2024 and 2026. Opaque generated structs hide fields
behind accessors. Select `API_HYBRID` to expose fields and accessors during a
staged migration, or `API_OPEN` to preserve direct access.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).api_level = API_HYBRID;
```

## Go enum-prefix stripping (`edition-2026-guide`)

Edition 2024 and later expose `features.(pb.go).strip_enum_prefix` at file,
enum, and enum-value scope:

- `STRIP_ENUM_PREFIX_KEEP` preserves existing generated names and is the
  default.
- `STRIP_ENUM_PREFIX_GENERATE_BOTH` emits both forms for migration.
- `STRIP_ENUM_PREFIX_STRIP` removes the repeated enum-name prefix.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).strip_enum_prefix = STRIP_ENUM_PREFIX_STRIP;
```
