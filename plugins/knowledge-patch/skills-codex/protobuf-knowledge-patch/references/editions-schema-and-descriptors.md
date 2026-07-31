# Editions, Schema, and Descriptors

## Edition and compiler relationship

Edition numbers are independent of shared releases and runtime majors.
Edition 2023 requires `protoc` 27.0 or later; Edition 2024 requires 32.0 or
later. The Edition 2024 announcement targeted protobuf 32.x in Q3 2025 and
marked its then-announced behavior as provisional. (release-lifecycle,
edition-2024-announcement)

upb generators enable Edition 2024 from 34.0. At the same release, upb parses
descriptor `syntax` and `edition` more strictly, so malformed dynamic
descriptor data can be rejected. (34.0)

## Edition 2024 C++ defaults

`features.(pb.cpp).string_type` defaults to `VIEW` instead of `STRING`, so
generated string-field APIs have view semantics unless overridden. The
`enum_name_uses_string_view` default similarly changes enum-name helpers to
return borrowed `absl::string_view`. (edition-2024-announcement)

```cpp
absl::string_view Foo_Name(int);
```

Edition 2024 removes the `ctype` field option. Select the representation with
`features.(pb.cpp).string_type`. (edition-2024-announcement)

## Naming-style enforcement

Edition 2024 enables strict naming-style enforcement by default through
`feature.enforce_naming_style`, preserving schema round-trippability. A feature
value can opt a file back into legacy naming. (edition-2024-announcement)

At 35.0, the compiler implements the Edition 2026 naming-style behavior and
its corresponding feature value. Edition 2026 schemas can be rejected when
field names collide after language-specific name conversion. (35.0)

## Symbol visibility

Edition 2024 introduces `default_symbol_visibility` and defaults it to
`EXPORT_TOP_LEVEL`: top-level messages/enums are exported and nested symbols
are local. `export` and `local` modifiers override visibility without changing
code generation. (edition-2024-announcement)

```proto
local message LocalMessage {
  export enum ExportedNestedEnum {
    UNKNOWN_EXPORTED_NESTED_ENUM_VALUE = 0;
  }
}
```

At 35.0, visibility checking extends to service method request and response
messages. A type not visible from the service file produces an error.
(35.0)

## Imports and option dependencies

`import option` imports custom options without exposing the source file's
messages or enums as ordinary symbols. It must follow normal imports. Bazel
targets put these imports in `option_deps`, which requires Bazel 8 or later.
(edition-2024-announcement)

```proto
edition = "2024";
import option "bar.proto";

option (file_opt1) = true;
option (file_opt2) = {bar: true};
```

```build
proto_library(
  name = "foo",
  srcs = ["foo.proto"],
  option_deps = [":custom_option"],
)
```

Edition 2024 removes `import weak` and the `weak` field option. Migrate weak
imports used only for custom options to `import option`.
(edition-2024-announcement)

At 34.0, the compiler validates feature support on custom options and validates
both options and features during parsing. Unsupported feature definitions that
previously passed unchecked can now fail. The C++ implementation also rejects
`[unverified_lazy = true]` on extension fields. (34.0)

## Descriptor label migration

The 31.0 release deprecated `FieldDescriptor.label`, `getLabel`, and the
`LABEL_*` constants. Use semantic queries:

- `isRepeated` for cardinality;
- `isRequired` for required proto2 or Editions fields;
- `hasPresence` for singular-field presence;
- `hasOptionalKeyword` and `getRealContainingOneof` only for proto3
  optional-keyword and real-oneof questions.

For Editions fields, the transitional `getLabel` behavior simplifies every
singular field to `LABEL_OPTIONAL` and every repeated field to
`LABEL_REPEATED`; `hasOptionalKeyword` is always false. Proto2 labels continue
to reflect declared keywords during that transition. (31.0)

The 34.0 migration removes C++ `FieldDescriptor::label()`, Python
`FieldDescriptor.label`, and PHP `FieldDescriptor::getLabel()`.
(34.0-migration)

For current C++, replace `has_optional_keyword()` with `has_presence()` and
write the old `is_optional()` meaning as
`!is_required() && !is_repeated()`. PHP provides `hasPresence()` and removes
its broken `hasOptionalKeyword()` helper. Objective-C replaces the removed
`-[GPBFieldDescriptor optional]` with
`!required && fieldType == GPBFieldTypeSingle`.
(34.0-announcement, 34.0)

## Java Edition 2024 features

`nest_in_file_class` replaces the removed `java_multiple_files` option.
Edition 2024 generates classes in separate files by default. The default outer
class is the camel-cased proto filename plus `Proto`; for example,
`foo/bar_baz.proto` becomes `BarBazProto`. Override with
`java_outer_classname`. (edition-2024-announcement)

The opt-in `large_enum` feature permits Java generated enums beyond the normal
enum-constant limit. Such types emulate enums and do not support every enum
operation, including `switch`. From 34.0, the Java lite runtime honors
`large_enum` and correctly handles aliased values. (edition-2024-announcement,
34.0)

## Edition 2026 language features

### Go API level

`features.(pb.go).api_level` defaults to `API_OPAQUE` in Editions 2024 and
2026, hiding generated struct fields behind accessors. Select `API_OPEN` to
preserve direct field access or `API_HYBRID` to expose fields and accessors
during migration. (edition-2026-guide)

```proto
edition = "2026";
import option "google/protobuf/go_features.proto";
option features.(pb.go).api_level = API_HYBRID;
```

### Go enum-prefix stripping

Edition 2024 and newer support `features.(pb.go).strip_enum_prefix` at file,
enum, or enum-value scope. `STRIP_ENUM_PREFIX_KEEP` preserves names;
`STRIP_ENUM_PREFIX_GENERATE_BOTH` supports staged migration; and
`STRIP_ENUM_PREFIX_STRIP` removes the repeated enum prefix.
(edition-2026-guide)

```proto
edition = "2026";
import option "google/protobuf/go_features.proto";
option features.(pb.go).strip_enum_prefix = STRIP_ENUM_PREFIX_STRIP;
```

### C++ repeated accessors and options

`features.(pb.cpp).repeated_type = PROXY` makes generated repeated accessors
return `RepeatedFieldProxy`. `LEGACY` remains the default, so the API changes
only when selected at file or field scope. Edition 2026 schemas must remove
`cc_api_version`, `cc_utf8_verification`, and `cc_enable_arenas`.
(edition-2026-guide)

```proto
edition = "2026";
import option "google/protobuf/cpp_features.proto";
option features.(pb.cpp).repeated_type = PROXY;
```

### Custom JSON enum strings and C++ namespaces

Enum values can override their JSON string with `(pb.enumvalue.json).string`,
complementing the field-level `json_name` option for conflicts, migrations,
and external naming requirements. (edition-2026)

```proto
enum Foo {
  FOO_UNSPECIFIED = 0;
  FOO_BAR = 5 [(pb.enumvalue.json).string = "custom_string_here"];
}
```

The file option `(pb.file.cpp).namespace` decouples generated C++ bindings from
the proto package. (edition-2026)

```proto
import option "google/protobuf/cpp_options.proto";
package clock.time;
option (pb.file.cpp).namespace = "clock_time";
```
