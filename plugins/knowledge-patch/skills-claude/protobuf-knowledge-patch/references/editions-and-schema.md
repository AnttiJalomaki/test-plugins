# Editions and schema authoring

## Edition 2024 announcement status (`edition-2024-announcement`)

Edition 2024 was announced as a provisional target for protobuf 32.x in Q3
2025. The behaviors below were explicitly described as subject to change before
that release; validate generated APIs when adopting them.

## C++ string and enum views

Edition 2024 changes the default `features.(pb.cpp).string_type` from `STRING`
to `VIEW`. Generated string-field APIs therefore use view behavior unless the
feature is overridden.

The default `enum_name_uses_string_view` also changes generated enum-name
helpers from referenced `std::string` values to borrowed `absl::string_view`:

```cpp
absl::string_view Foo_Name(int);
```

Copy the result when it must outlive its backing storage or be null-terminated.

## Naming-style enforcement

`feature.enforce_naming_style` enables strict naming-style enforcement by
default in Edition 2024 so schemas remain round-trippable. A feature value can
opt a file back into legacy naming.

Release `35.0` implements the Edition 2026 collision behavior and adds its
corresponding `enforce_naming_style` value. Field names that collide after a
language-specific name conversion can now make the schema invalid.

## Symbol visibility

Edition 2024 `default_symbol_visibility` defaults to `EXPORT_TOP_LEVEL`:
top-level messages and enums are exported, while nested symbols are local. Use
`export` and `local` to set a different boundary without changing generated
code.

```proto
local message LocalMessage {
  export enum ExportedNestedEnum {
    UNKNOWN_EXPORTED_NESTED_ENUM_VALUE = 0;
  }
}
```

As of `35.0`, visibility checking includes service method input and output
messages. A request or response type that is not visible from the service file
is rejected.

## Java file layout and large enums

`nest_in_file_class` replaces the removed `java_multiple_files` option. Edition
2024 emits classes in their own files by default. The outer class name always
uses the camel-cased proto filename plus `Proto`; `foo/bar_baz.proto` becomes
`BarBazProto`. Set `java_outer_classname` to override it.

The opt-in `large_enum` feature allows generated Java enum-like types beyond
Java's ordinary enum-constant limit. They do not support every enum operation,
including `switch`. The Java lite runtime honors this feature as of `34.0` and
handles aliased values correctly.

## Descriptor cardinality and presence (`31.0`)

`FieldDescriptor.label`, `getLabel`, and the `LABEL_*` constants are deprecated.
Use `isRepeated` for cardinality, `isRequired` for required proto2 or Editions
fields, and `hasPresence` for singular-field presence. Proto3 code can use
`hasOptionalKeyword` and `getRealContainingOneof` only when it specifically
needs optional-keyword or oneof information.

For Editions, `getLabel` is simplified to `LABEL_OPTIONAL` for every singular
field and `LABEL_REPEATED` for every repeated field; `hasOptionalKeyword` always
returns false. Proto2 labels continue to report their declared `optional`,
`required`, or `repeated` keyword for now. The accessors themselves are removed
from C++, Python, and PHP in the release 34 migration.

## Option-only imports

`import option` imports custom options without exposing the source file's
messages and enums as normal symbols. It must follow ordinary imports. Bazel
targets place the dependency in `option_deps`, which requires Bazel 8 or newer.

```proto
edition = "2024";
import option "bar.proto";

option (file_opt1) = true;
option (file_opt2) = {bar: true};
```

Edition 2024 removes `import weak` and the `weak` field option. Code that used
weak imports solely to consume custom options without generating C++ or Go code
should migrate to `import option`.

## C++ string representation options

Edition 2024 removes the `ctype` field option. Select the generated C++ string
representation through `features.(pb.cpp).string_type`.

## Compiler limits and option validation

The `34.0-announcement` makes `protoc` reject field names longer than 2^16
characters.

In `34.0`, the compiler validates feature support on custom options and
validates options and features during parsing. Definitions using unsupported
features may now be rejected. The C++ implementation also rejects
`[unverified_lazy = true]` on extensions.

upb performs stricter validation of `syntax` and `edition` while parsing
descriptors, so malformed dynamic descriptors can fail where they were
previously accepted. Its generators also enable Edition 2024.

## Go Editions API (`edition-2026-guide`)

`features.(pb.go).api_level` defaults to `API_OPEN` in Edition 2023 and
`API_OPAQUE` in Editions 2024 and 2026. The Opaque API hides generated struct
fields behind accessors. Select `API_HYBRID` to expose fields and accessors
during migration, or `API_OPEN` to retain direct field access.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).api_level = API_HYBRID;
```

## C++ repeated-field proxy accessors (`edition-2026-guide`)

Edition 2026 adds `features.(pb.cpp).repeated_type`. Selecting `PROXY` makes
generated repeated-field accessors return `RepeatedFieldProxy` rather than
`RepeatedField` or `RepeatedPtrField` pointers and references. The default is
`LEGACY`, so APIs change only when the feature is selected at file or field
scope.

```proto
edition = "2026";

import option "google/protobuf/cpp_features.proto";

option features.(pb.cpp).repeated_type = PROXY;
```

## Go enum-prefix stripping (`edition-2026-guide`)

Edition 2024 and newer allow `features.(pb.go).strip_enum_prefix` at file, enum,
or enum-value scope. `STRIP_ENUM_PREFIX_KEEP` preserves names,
`STRIP_ENUM_PREFIX_GENERATE_BOTH` provides both forms during migration, and
`STRIP_ENUM_PREFIX_STRIP` removes the repeated enum-name prefix.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).strip_enum_prefix = STRIP_ENUM_PREFIX_STRIP;
```

## Removed C++ schema options (`edition-2026-guide`)

Edition 2026 schemas must remove `cc_api_version`, `cc_utf8_verification`, and
`cc_enable_arenas`.

## Custom enum JSON names (`edition-2026`)

An Edition 2026 enum value can override its JSON string through
`(pb.enumvalue.json).string`. This complements field-level `json_name` for
conflict avoidance, migrations, and external naming contracts.

```proto
enum Foo {
  FOO_UNSPECIFIED = 0;
  FOO_BAR = 5 [(pb.enumvalue.json).string = "custom_string_here"];
}
```

## Generated C++ namespace (`edition-2026`)

The file option `(pb.file.cpp).namespace` decouples the generated C++ namespace
from the proto package.

```proto
import option "google/protobuf/cpp_options.proto";

package clock.time;

option (pb.file.cpp).namespace = "clock_time";
```
