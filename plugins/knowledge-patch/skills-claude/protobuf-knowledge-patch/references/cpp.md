# C++ runtime and generated APIs

## Borrowed descriptor strings (`30.0-migration`)

`MessageLite::GetTypeName`, `UnknownField::length_delimited`, and descriptor
name accessors such as `FieldDescriptor::full_name` now return
`absl::string_view`. Accept a borrowed view or explicitly copy into
`std::string` when ownership is needed. Never assume `view.data()` is
null-terminated.

Edition 2024 also defaults generated string fields to view behavior and changes
enum-name helpers to return `absl::string_view`; see the Editions reference.

## Debug output is redacted and non-parseable (`30.0-migration`)

`AbslStringify`, `proto2::ShortFormat`, `proto2::Utf8Format`, and all
`*DebugString` methods redact fields carrying `debug_redact`, prepend a
per-process randomized marker, and no longer produce parseable TextFormat. Use
them for logs only. Use binary encoding for serialization, or explicitly call
`TextFormat.printer().printToString(proto)` when parseable unredacted text is a
deliberate requirement.

## Removed foundational APIs (`30.0-migration`)

- Replace `Arena::CreateMessage` with `Arena::Create`.
- Replace `Arena::GetArena(value)` with `value->GetArena()`.
- Replace `JsonOptions` with `JsonPrintOptions`.
- `RepeatedPtrField::ClearedCount` has no direct replacement; migrate the
  allocation strategy to arenas.
- The descriptor no longer exposes the raw `ctype` option. Query
  `FieldDescriptor::cpp_string_type()`.
- C++17 is the minimum supported language level.

## Cleared arena oneofs (`30.0-migration`)

Debug builds clear the memory for an arena-allocated oneof message when the
oneof is cleared, and ASAN poisons that region. Holding or using the old message
pointer is a use-after-free; reacquire the active oneof member after mutation.

## Reflection capacity reservation (`30.0`)

`MutableRepeatedFieldRef<T>::Reserve()` is removed. Generic reflection code
must stop trying to reserve capacity through this view.

## Rollout macros and arena constructors (`34.0-announcement`)

These future-rollout macros are removed because their behavior is unconditional:

- `PROTOBUF_FUTURE_RENAME_ADD_UNUSED_IMPORT`
- `PROTOBUF_FUTURE_REMOVE_ADD_IGNORE_CRITERIA`
- `PROTOBUF_FUTURE_STRING_VIEW_DESCRIPTOR_DATABASE`
- `PROTOBUF_FUTURE_NO_RECURSIVE_MESSAGE_COPY`
- `PROTOBUF_FUTURE_REMOVE_REPEATED_PTR_FIELD_ARENA_CONSTRUCTOR`
- `PROTOBUF_FUTURE_REMOVE_MAP_FIELD_ARENA_CONSTRUCTOR`
- `PROTOBUF_FUTURE_REMOVE_REPEATED_FIELD_ARENA_CONSTRUCTOR`

The constructors `RepeatedField(Arena*)`, `RepeatedPtrField(Arena*)`, and
`Map(Arena*)` are deleted. Do not construct generated field containers directly
from an arena pointer.

## Generator and descriptor migrations (`34.0-announcement`)

- Replace `AddUnusedImportTrackFile()` and `ClearUnusedImportTrackFiles()` with
  `AddDirectInputFile()` and `ClearDirectInputFiles()`.
- Pass a `unique_ptr` to `AddIgnoreCriteria()` so ownership transfers
  explicitly.
- Replace `FieldDescriptor::has_optional_keyword()` with `has_presence()` when
  presence is the real question.
- Express `FieldDescriptor::is_optional()` as
  `!is_required() && !is_repeated()`.
- `UseDeprecatedLegacyJsonFieldConflicts()` has no replacement.

`RepeatedField::Get` and `RepeatedPtrField::Get` now perform comprehensive
out-of-bounds checks. Several logically constant APIs are also `[[nodiscard]]`,
so builds that discard their return values may newly fail under warning-as-error
policies.

## `RepeatedPtrField` storage migration (`34.0-migration`)

Elements now occupy contiguous chunks of preallocated memory, similar to
`std::deque`. Review code that depends on prior copy or move behavior. Some
`UnsafeArena` operations become equivalent to their arena-safe counterparts and
may later be deprecated.

The deprecated C++ `FieldDescriptor::label()` accessor is removed. Use
cardinality and presence predicates instead of reconstructing a label.

## Checked operations and invariants (`34.0`)

- `[unverified_lazy = true]` is rejected on extensions; remove the option from
  extension fields.
- `ExtractSubrange` and `DeleteSubrange` validate ranges and abort on invalid
  bounds.
- Debug builds reject `CopyFrom` when the destination is a descendant of the
  source. Never copy a parent message into one of its descendants.
- `BinaryToJson` propagates a parse failure when skipping an unknown field
  fails; callers can now receive an error for input previously treated as
  successful.
- `FieldMaskUtil::TrimMessage` now supports repeated message fields.

## Language and build cleanup (`34.0`)

`PROTOBUF_CONSTEXPR` is removed; use the `constexpr` language keyword. The old
`safe_boundary_check` switch is also removed; source builds select checking
with `--//third_party/protobuf:bounds_check_mode`.

## Arena smart pointers (`35.0`)

`Arena::Ptr` and `Arena::UniquePtr` express pointer ownership for values
associated with an arena. Prefer them where raw arena-backed pointers left
ownership or lifetime unclear.

## Abseil flags (`35.0`)

Messages and enums can be used directly as Abseil flag values. Enums are also
supported inside `std::vector` flag values.

## Expanded bounds checks (`35.0`)

`UnsafeArenaExtractSubrange`, `ReleaseLast`, and `SwapElements` now validate
their indices and ranges. Validate arguments in application code rather than
depending on previously unchecked behavior.
