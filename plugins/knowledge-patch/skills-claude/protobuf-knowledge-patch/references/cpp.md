# C++ runtime and generated APIs

Source batches represented here: 30.0-migration, 30.0, 31.0,
edition-2024-announcement, 34.0-announcement, 34.0-migration, 34.0, 35.0,
36.0-rc1, edition-2026-guide, edition-2026.

## Language and distribution baseline

The v30 migration raises the minimum C++ language level from C++14 to C++17.
The C++ CocoaPods release is removed; consume the C++ runtime from the GitHub
release.

Generated code and the C++ runtime must match exactly. Do not assume ABI
stability across minor or patch releases.

## Borrowed strings and debug output

`MessageLite::GetTypeName`, `UnknownField::length_delimited`, and descriptor
name methods such as `FieldDescriptor::full_name` return
`absl::string_view` after the v30 migration. Update callers to retain borrowed
views safely or make an explicit `std::string` copy. Never assume
`string_view::data()` is null-terminated.

Edition 2024's C++ string-field and enum-name defaults also use views; audit
generated-call sites when adopting the edition.

`AbslStringify`, `proto2::ShortFormat`, `proto2::Utf8Format`, and
`*DebugString` methods redact fields marked `debug_redact`, prepend a
per-process randomized marker, and no longer emit parseable TextFormat. Use
debug forms for logging, binary encoding for serialization, or
`TextFormat.printer().printToString(proto)` when explicitly requiring
parseable, unredacted text.

## Arena and container migration

Replace removed APIs:

| Removed API | Migration |
| --- | --- |
| `Arena::CreateMessage` | `Arena::Create` |
| `Arena::GetArena(value)` | `value->GetArena()` |
| `JsonOptions` | `JsonPrintOptions` |
| `RepeatedPtrField::ClearedCount` | No direct replacement; redesign around arenas |

Do not construct `RepeatedField(Arena*)`, `RepeatedPtrField(Arena*)`, or
`Map(Arena*)`; those constructors are removed at the v34 boundary.

`RepeatedPtrField` uses contiguous chunks of preallocated storage in v34,
similar to `std::deque`. Review code depending on older copy/move behavior.
Some `UnsafeArena` operations can become equivalent to their arena-safe forms
and may be deprecated.

`Arena::Ptr` and `Arena::UniquePtr` are available in v35 for explicit
smart-pointer management of arena-associated values.

In debug builds, clearing an arena-allocated oneof message clears its memory
and ASAN poisons it. Any access through a stale pointer is a diagnosed
use-after-free.

## Descriptor and reflection APIs

`FieldDescriptor::label()` is deprecated in v31 and removed in v34. Use:

- `is_repeated()` for repeated cardinality;
- `is_required()` for required fields;
- `has_presence()` for presence;
- `!is_required() && !is_repeated()` when testing optional cardinality.

Do not replace label checks mechanically with
`FieldDescriptor::has_optional_keyword()`: that API is also removed at the v34
boundary. Edition singular fields do not carry proto2-style label meaning.

The v30 migration no longer exposes `ctype` through `FieldDescriptor` options.
Use `FieldDescriptor::cpp_string_type()`. Edition 2024 removes the schema
`ctype` option in favor of `features.(pb.cpp).string_type`.

`MutableRepeatedFieldRef<T>::Reserve()` is removed in v30. Generic repeated
field reflection must stop reserving capacity through that view.

## Removed and changed runtime APIs

At the v34 boundary:

- replace `AddUnusedImportTrackFile()` with `AddDirectInputFile()`;
- replace `ClearUnusedImportTrackFiles()` with `ClearDirectInputFiles()`;
- pass a `unique_ptr` to `AddIgnoreCriteria()` to transfer ownership;
- replace `FieldDescriptor::has_optional_keyword()` with `has_presence()`;
- express `FieldDescriptor::is_optional()` as
  `!is_required() && !is_repeated()`;
- remove calls to `UseDeprecatedLegacyJsonFieldConflicts()`, which has no
  replacement.

The v34 rollout macros below are removed and their behaviors become
unconditional:

- `PROTOBUF_FUTURE_RENAME_ADD_UNUSED_IMPORT`
- `PROTOBUF_FUTURE_REMOVE_ADD_IGNORE_CRITERIA`
- `PROTOBUF_FUTURE_STRING_VIEW_DESCRIPTOR_DATABASE`
- `PROTOBUF_FUTURE_NO_RECURSIVE_MESSAGE_COPY`
- `PROTOBUF_FUTURE_REMOVE_REPEATED_PTR_FIELD_ARENA_CONSTRUCTOR`
- `PROTOBUF_FUTURE_REMOVE_MAP_FIELD_ARENA_CONSTRUCTOR`
- `PROTOBUF_FUTURE_REMOVE_REPEATED_FIELD_ARENA_CONSTRUCTOR`

`PROTOBUF_CONSTEXPR` is removed in v34; use the C++ `constexpr` keyword.

Edition 2026 schemas must remove `cc_api_version`, `cc_utf8_verification`, and
`cc_enable_arenas`.

## Bounds and ownership hardening

`RepeatedField::Get` and `RepeatedPtrField::Get` gain comprehensive bounds
checks at the v34 boundary. Several logically constant APIs become
`[[nodiscard]]`, so ignoring results can newly trigger compiler diagnostics.

In v34, `ExtractSubrange` and `DeleteSubrange` validate their ranges and abort
on out-of-bounds access. In v35, `UnsafeArenaExtractSubrange`, `ReleaseLast`,
and `SwapElements` are also checked. Validate indices, counts, and ranges
before calling any of them.

Debug builds check that a `CopyFrom` destination is not a descendant of its
source. Never copy a recursive parent into one of its descendants.

The `[unverified_lazy = true]` option is rejected on extensions in v34. Remove
it from extension declarations.

## Parsing, JSON, and field masks

`BinaryToJson` returns a parse failure when skipping an unknown field fails;
do not assume unknown-field skipping always succeeds.

`FieldMaskUtil::TrimMessage` handles repeated message fields in v34, so masks
can trim messages containing repeated submessages.

`TimeUtil::FromString` validates `Duration` and `Timestamp` values against
their specification limits in 36.0-rc1. Syntactically valid but out-of-range
values fail.

## New generated/runtime capabilities

Protobuf message and enum types can be used directly as Abseil flag values in
v35. Enum support includes `std::vector` values.

Edition 2026 offers `features.(pb.cpp).repeated_type = PROXY`, which makes
repeated-field accessors return `RepeatedFieldProxy` rather than pointers or
references to concrete repeated containers. It remains opt-in because the
default is `LEGACY`.

Edition 2026 also offers `(pb.file.cpp).namespace` to choose a generated C++
namespace independently of the proto package.
