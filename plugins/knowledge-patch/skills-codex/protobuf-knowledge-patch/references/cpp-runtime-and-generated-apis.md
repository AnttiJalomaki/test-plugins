# C++ Runtime and Generated APIs

Use this reference when upgrading generated C++ code, working with arenas or
repeated fields, consuming descriptors, or converting protobuf data to debug,
JSON, text, or time representations.

## Language, packaging, and compatibility

`30.0-migration` raises the language minimum from C++14 to C++17. The same
migration removes the C++ CocoaPods distribution; use the C++ runtime from the
release artifacts.

C++ generated code and runtime must match exactly. The `release-lifecycle`
rules also make no ABI-stability promise across minor or patch releases.
Rebuild all generated code and native dependants together.

## Borrowed strings and descriptor results

Since `30.0-migration`, the following return `absl::string_view` rather than
`std::string`:

- `MessageLite::GetTypeName`
- `UnknownField::length_delimited`
- descriptor name APIs such as `FieldDescriptor::full_name`

Update callers to preserve the borrowed lifetime, or copy explicitly to
`std::string`. A view's `data()` is not guaranteed to be null-terminated.

Edition 2024 also defaults generated C++ string fields to view behavior and
enum-name helpers to `absl::string_view`; see the Editions reference before
assuming the generated accessor type.

## Debug output is redacted and non-parseable

In `30.0-migration`, `AbslStringify`, `proto2::ShortFormat`,
`proto2::Utf8Format`, and the `*DebugString` methods:

- redact fields annotated with `debug_redact`;
- add a per-process randomized prefix; and
- no longer produce parseable TextFormat.

Use debug output for logs only. Use binary encoding for serialization. When an
explicitly parseable, unredacted textual representation is required, call
`TextFormat.printer().printToString(proto)` and review the data-exposure
implications.

## Removed and changed runtime APIs

The `30.0-migration` replacements are:

| Removed API | Replacement |
| --- | --- |
| `Arena::CreateMessage` | `Arena::Create` |
| `Arena::GetArena` | `value->GetArena()` |
| `JsonOptions` | `JsonPrintOptions` |
| `FieldDescriptor` option access to `ctype` | `FieldDescriptor::cpp_string_type()` |

`RepeatedPtrField::ClearedCount` has no direct replacement; migrate the
allocation strategy to arenas.

`30.0` removes `MutableRepeatedFieldRef<T>::Reserve()`. Generic repeated-field
reflection must stop trying to reserve capacity.

The `34.0-announcement` replacements are:

- `AddUnusedImportTrackFile()` becomes `AddDirectInputFile()`.
- `ClearUnusedImportTrackFiles()` becomes `ClearDirectInputFiles()`.
- `AddIgnoreCriteria()` now takes ownership through `unique_ptr`.
- `FieldDescriptor::has_optional_keyword()` becomes `has_presence()`.
- Express `FieldDescriptor::is_optional()` as
  `!is_required() && !is_repeated()`.
- `UseDeprecatedLegacyJsonFieldConflicts()` has no replacement.

`34.0` removes `PROTOBUF_CONSTEXPR`; use the C++ `constexpr` keyword directly.

## Rollout macros and constructors

The `34.0-announcement` makes the behavior of these rollout macros
unconditional and removes the macros:

- `PROTOBUF_FUTURE_RENAME_ADD_UNUSED_IMPORT`
- `PROTOBUF_FUTURE_REMOVE_ADD_IGNORE_CRITERIA`
- `PROTOBUF_FUTURE_STRING_VIEW_DESCRIPTOR_DATABASE`
- `PROTOBUF_FUTURE_NO_RECURSIVE_MESSAGE_COPY`
- `PROTOBUF_FUTURE_REMOVE_REPEATED_PTR_FIELD_ARENA_CONSTRUCTOR`
- `PROTOBUF_FUTURE_REMOVE_MAP_FIELD_ARENA_CONSTRUCTOR`
- `PROTOBUF_FUTURE_REMOVE_REPEATED_FIELD_ARENA_CONSTRUCTOR`

It also deletes `RepeatedField(Arena*)`, `RepeatedPtrField(Arena*)`, and
`Map(Arena*)`. Do not construct field containers directly from an arena
pointer.

## Arena lifetimes and ownership

Debug builds in `30.0-migration` clear the memory of an arena-allocated oneof
message when the oneof is cleared; ASAN poisons the region. Any later access is
a use-after-free. Do not retain a pointer, view, or reference across a clear or
case change.

`34.0-migration` changes `RepeatedPtrField` storage to contiguous chunks of
preallocated memory, similar to `std::deque`. Review code that relies on the
old copy or move behavior. Some `UnsafeArena` operations can become equivalent
to their arena-safe counterparts and may be deprecated.

`35.0` adds `Arena::Ptr` and `Arena::UniquePtr` as explicit smart-pointer forms
for arena-associated values. Prefer them where raw arena-backed pointers made
ownership unclear.

## Bounds, recursion, and ancestry checks

The checks expand over several releases:

- In `34.0-announcement`, `RepeatedField::Get` and
  `RepeatedPtrField::Get` perform comprehensive bounds checks.
- In `34.0`, `ExtractSubrange` and `DeleteSubrange` validate ranges and abort
  on out-of-bounds access.
- In `35.0`, `UnsafeArenaExtractSubrange`, `ReleaseLast`, and `SwapElements`
  validate their indices and ranges.

Always validate the start, count, and index before calling these methods.

`34.0` debug builds also reject a `CopyFrom` whose destination is a descendant
of its source. Recursive-message code must not copy a parent into one of its
own descendants.

The old `safe_boundary_check` build mechanism is removed in `34.0`. Source
builds configure the behavior with:

```text
--//third_party/protobuf:bounds_check_mode
```

## Conversion and utility behavior

### Binary-to-JSON failures

In `34.0`, `BinaryToJson` reports a parse failure when skipping an unknown
field fails. Callers that formerly received apparent success can now enter
their error path; retain and test that error propagation.

### Field-mask trimming

`FieldMaskUtil::TrimMessage` handles repeated message fields in `34.0`.
Callers can trim messages that contain repeated submessages instead of
special-casing them.

### Time parsing

`36.0-rc1` makes `TimeUtil::FromString` validate parsed `Duration` and
`Timestamp` values against specification limits. Syntactically valid but
out-of-range values are rejected.

## Access and compiler hardening

The `34.0-announcement` marks more logically constant APIs `[[nodiscard]]`.
Ignoring their values can produce new diagnostics; inspect and either use the
result or explicitly justify discarding it.

The compiler also rejects field names longer than 2^16 characters instead of
accepting an unbounded name.

`34.0` rejects `[unverified_lazy = true]` on extensions. Remove the option from
extension fields before regenerating C++.

## New value and generated-API integrations

`35.0` lets protobuf messages and enum values serve directly as native Abseil
flag values. Enum support also covers `std::vector` values.

The `edition-2026-guide` adds an opt-in repeated-field proxy API:

```proto
edition = "2026";

import option "google/protobuf/cpp_features.proto";

option features.(pb.cpp).repeated_type = PROXY;
```

With `PROXY`, generated repeated accessors return `RepeatedFieldProxy` rather
than `RepeatedField` or `RepeatedPtrField` pointers and references. `LEGACY`
remains the default.

Edition 2026 removes `cc_api_version`, `cc_utf8_verification`, and
`cc_enable_arenas`. The `edition-2026` file option
`(pb.file.cpp).namespace` can decouple the generated namespace from the proto
package:

```proto
import option "google/protobuf/cpp_options.proto";

package clock.time;

option (pb.file.cpp).namespace = "clock_time";
```

## Upgrade checks

- Build all native consumers in C++17 mode.
- Regenerate and relink against the exact same runtime release.
- Search for removed arena, descriptor-label, JSON, constexpr, and rollout
  APIs.
- Audit every stored string view and arena-backed pointer for lifetime.
- Exercise repeated-field bounds under the build's configured check mode.
- Test recursive `CopyFrom`, malformed unknown fields in `BinaryToJson`, and
  out-of-range time strings.
- Keep debug strings out of persistence and parser round trips.
