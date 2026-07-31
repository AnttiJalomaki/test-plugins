# C++ Runtime and Generated APIs

## Language baseline and borrowed strings

C++17 is the minimum supported language level from the 30.0 migration.
(30.0-migration)

`MessageLite::GetTypeName`, `UnknownField::length_delimited`, and descriptor
name methods such as `FieldDescriptor::full_name` return
`absl::string_view`. Treat results as borrowed; copy to `std::string` when
ownership is required, and do not assume `data()` is null-terminated.
(30.0-migration)

Edition 2024 also defaults C++ string-field generation to view behavior and
enum-name helpers to `absl::string_view`; see the Editions reference for the
feature switches. (edition-2024-announcement)

## Removed and replaced APIs

Apply these 30.0 migration replacements:

| Removed API | Replacement |
| --- | --- |
| `Arena::CreateMessage` | `Arena::Create` |
| `Arena::GetArena` | `value->GetArena()` |
| `JsonOptions` | `JsonPrintOptions` |
| `FieldDescriptor` exposure of `ctype` | `FieldDescriptor::cpp_string_type()` |

`RepeatedPtrField::ClearedCount` has no direct replacement; migrate ownership
to arenas. `MutableRepeatedFieldRef<T>::Reserve()` was also removed, so generic
reflection code must stop pre-reserving through that API. (30.0-migration,
30.0)

At v34, replace `AddUnusedImportTrackFile()` and
`ClearUnusedImportTrackFiles()` with `AddDirectInputFile()` and
`ClearDirectInputFiles()`. `AddIgnoreCriteria()` now takes ownership through a
`unique_ptr`. Replace `FieldDescriptor::has_optional_keyword()` with
`has_presence()`, and express `FieldDescriptor::is_optional()` as
`!is_required() && !is_repeated()`. `UseDeprecatedLegacyJsonFieldConflicts()`
has no replacement. (34.0-announcement)

The v34 rollout macros were deleted because their behavior is unconditional:

- `PROTOBUF_FUTURE_RENAME_ADD_UNUSED_IMPORT`;
- `PROTOBUF_FUTURE_REMOVE_ADD_IGNORE_CRITERIA`;
- `PROTOBUF_FUTURE_STRING_VIEW_DESCRIPTOR_DATABASE`;
- `PROTOBUF_FUTURE_NO_RECURSIVE_MESSAGE_COPY`;
- `PROTOBUF_FUTURE_REMOVE_REPEATED_PTR_FIELD_ARENA_CONSTRUCTOR`;
- `PROTOBUF_FUTURE_REMOVE_MAP_FIELD_ARENA_CONSTRUCTOR`;
- `PROTOBUF_FUTURE_REMOVE_REPEATED_FIELD_ARENA_CONSTRUCTOR`.

In particular, `RepeatedField(Arena*)`, `RepeatedPtrField(Arena*)`, and
`Map(Arena*)` no longer exist. (34.0-announcement)

`PROTOBUF_CONSTEXPR` was removed at 34.0; use `constexpr` directly. Edition
2026 schemas also remove the `cc_api_version`, `cc_utf8_verification`, and
`cc_enable_arenas` options. (34.0, edition-2026-guide)

## Arenas, ownership, and message lifetime

In debug builds, clearing an arena-allocated oneof message clears its memory;
ASAN poisons the region. Any later access is a use-after-free bug rather than
a supported stale-object pattern. (30.0-migration)

The 34.0 `RepeatedPtrField` layout stores elements in contiguous chunks of
preallocated memory, similar to `std::deque`. Review code that depends on old
copy/move behavior. Some `UnsafeArena` operations consequently become
equivalent to arena-safe operations and may be deprecated. (34.0-migration)

At 35.0, `Arena::Ptr` and `Arena::UniquePtr` provide explicit smart-pointer
forms for values associated with an arena. Prefer them where raw arena-backed
pointers previously obscured ownership. (35.0)

## Bounds and graph-safety checks

The v34 changes make `RepeatedField::Get` and `RepeatedPtrField::Get`
comprehensively bounds-checked, and mark several logically constant APIs
`[[nodiscard]]`. Code that ignored results may gain diagnostics.
(34.0-announcement)

At 34.0, `ExtractSubrange` and `DeleteSubrange` validate ranges and abort on
out-of-bounds access. Debug builds also reject `CopyFrom` when the destination
is a descendant of the source. Validate indices/counts and never copy a parent
message into one of its descendants. (34.0)

At 35.0, `UnsafeArenaExtractSubrange`, `ReleaseLast`, and `SwapElements` also
validate their bounds. The `safe_boundary_check` source-build mechanism is
gone; builds choose checking through
`--//third_party/protobuf:bounds_check_mode`. (35.0, 34.0)

## Debug and text output

`AbslStringify`, `proto2::ShortFormat`, `proto2::Utf8Format`, and
`*DebugString` redact fields annotated with `debug_redact`, add a randomized
per-process prefix, and no longer emit parseable TextFormat. Use them for logs,
not serialization. Use binary encoding for interchange, or an explicit
`TextFormat.printer().printToString(proto)` call when parseable unredacted text
is required.
(30.0-migration)

## Parsing, conversion, and validation

`BinaryToJson` now propagates a parse failure when skipping an unknown field
fails. Inputs that previously appeared successful can reach error paths.
`FieldMaskUtil::TrimMessage` now supports repeated message fields.
(34.0)

The C++ implementation rejects `[unverified_lazy = true]` on extensions;
remove the option from extension fields. Protoc also rejects field names longer
than 2^16 characters. (34.0, 34.0-announcement)

## Generated APIs and integration

At 35.0, protobuf messages and enums can be used directly as native Abseil flag
values; enum support includes `std::vector` values. `ProtoStr` can be used in
const contexts on the Rust side, but that language's view changes are covered
in its own reference. (35.0)

Edition 2026 offers `features.(pb.cpp).repeated_type = PROXY`, causing repeated
accessors to return `RepeatedFieldProxy` instead of `RepeatedField` or
`RepeatedPtrField` pointers/references. `LEGACY` remains the default, so this
API change occurs only when selected at file or field scope.
(edition-2026-guide)

Edition 2026 also allows `(pb.file.cpp).namespace` to decouple the generated
C++ namespace from the proto package. (edition-2026)
