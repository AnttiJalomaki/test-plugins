# Editions and schema authoring

Source batches represented here: 31.0, edition-2024-announcement, 34.0, 35.0,
36.0-rc1, edition-2026-guide, edition-2026.

## Descriptor labels and Editions

`FieldDescriptor.label`, `getLabel`, and the `LABEL_*` constants are
deprecated in v31 and the label accessors are removed in v34. Migrate
reflection, generators, and dynamic-message code to `isRepeated` for
cardinality, `isRequired` for required proto2 or Editions fields, and
`hasPresence` for singular-field presence. Proto3 code can use
`hasOptionalKeyword` or `getRealContainingOneof` only when it specifically
needs optional-keyword or oneof membership.

For Editions fields, the transitional `getLabel` behavior reports
`LABEL_OPTIONAL` for every singular field and `LABEL_REPEATED` for every
repeated field, while `hasOptionalKeyword` is always false. Proto2 labels
continue to reflect the declared `optional`, `required`, or `repeated` keyword
while the accessor remains available.

## Edition 2024 planning and defaults

The Edition 2024 announcement targeted protobuf 32.x in Q3 2025 and described
provisional behavior. Confirm the installed compiler's behavior before relying
on an announcement-only detail.

### C++ string and enum APIs

The default `features.(pb.cpp).string_type` changes from `STRING` to `VIEW`.
Generated C++ string fields therefore use view-oriented APIs unless explicitly
overridden.

The default for `enum_name_uses_string_view` changes generated enum-name
helpers from referenced `std::string` to `absl::string_view`:

```cpp
absl::string_view Foo_Name(int);
```

Callers must preserve borrowed-value lifetime and must not assume `data()` is
null-terminated.

### Naming and visibility

`feature.enforce_naming_style` enables strict naming-style enforcement by
default to keep schemas round-trippable. A feature value can opt a file back
into legacy naming.

`default_symbol_visibility` defaults to `EXPORT_TOP_LEVEL`: top-level symbols
are exported, nested symbols are local. `export` and `local` make the choice
explicit:

```proto
local message LocalMessage {
  export enum ExportedNestedEnum {
    UNKNOWN_EXPORTED_NESTED_ENUM_VALUE = 0;
  }
}
```

In v35, visibility checks also apply to service method request and response
types. A service cannot reference a type that is invisible from its file.

### Java generation

`nest_in_file_class` replaces the removed `java_multiple_files` option.
Edition 2024 generates classes in their own files by default. The default
outer class is the camel-cased proto filename plus `Proto`, so
`foo/bar_baz.proto` yields `BarBazProto`; `java_outer_classname` overrides it.

The opt-in `large_enum` feature supports Java enums beyond the normal constant
limit. The generated type emulates an enum and does not support every enum
operation, including `switch`. The Java lite runtime honors `large_enum` in
v34 and correctly handles aliased values.

### Option-only imports and removed weak options

Use `import option` to load custom options without making the imported file's
messages or enums ordinary symbols. Option imports must follow normal imports:

```proto
edition = "2024";
import option "bar.proto";

option (file_opt1) = true;
option (file_opt2) = {bar: true};
```

With Bazel, put these dependencies in `option_deps`, not `deps`.
`option_deps` requires Bazel 8 or later:

```build
proto_library(
  name = "foo",
  srcs = ["foo.proto"],
  option_deps = [":custom_option"],
)
```

Edition 2024 removes `import weak` and the weak field option. Migrate weak
custom-option imports that avoided C++ or Go generation to `import option`.

The `ctype` field option is also unavailable. Choose C++ string representation
with `features.(pb.cpp).string_type`.

### Compiler support

upb generators enable Edition 2024 in v34. The compiler also validates feature
support on custom options and validates options and features during parsing,
so unsupported combinations can now be rejected.

## Edition 2026 defaults

Edition 2026 changes `default_symbol_visibility` to `STRICT`. Explicitly
review cross-file references rather than inheriting Edition 2024's
top-level-export default.

`enforce_proto_limits` is implemented and descriptor proto-limit enforcement
is on by default. Descriptors exceeding those limits can fail after the
edition upgrade.

Edition 2026 naming-style enforcement includes collision detection after
language-specific field-name conversion. Rename fields that collapse to the
same generated name.

C# supports Edition 2026, and nullable-reference-type generation moves into
that edition.

## Edition 2026 language feature options

### Go generated API level

`features.(pb.go).api_level` defaults to `API_OPAQUE` in Editions 2024 and
2026, rather than Edition 2023's `API_OPEN`. Opaque generation hides struct
fields behind accessors. Select `API_OPEN` for direct field access or
`API_HYBRID` to expose fields and accessors during migration:

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).api_level = API_HYBRID;
```

### C++ repeated-field access

Edition 2026 adds `features.(pb.cpp).repeated_type`. `PROXY` returns
`RepeatedFieldProxy` from repeated-field accessors instead of pointers or
references to `RepeatedField`/`RepeatedPtrField`. The default remains
`LEGACY`, so the API changes only when selected at file or field scope:

```proto
edition = "2026";

import option "google/protobuf/cpp_features.proto";

option features.(pb.cpp).repeated_type = PROXY;
```

### Go enum-prefix stripping

Edition 2024 and later support `features.(pb.go).strip_enum_prefix` at file,
enum, or enum-value scope. `STRIP_ENUM_PREFIX_KEEP` preserves generated names,
`STRIP_ENUM_PREFIX_GENERATE_BOTH` assists migration, and
`STRIP_ENUM_PREFIX_STRIP` removes the repeated enum-name prefix:

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).strip_enum_prefix = STRIP_ENUM_PREFIX_STRIP;
```

### Custom enum JSON strings

An Edition 2026 enum value can override its JSON spelling with
`(pb.enumvalue.json).string`, complementing field-level `json_name`:

```proto
enum Foo {
  FOO_UNSPECIFIED = 0;
  FOO_BAR = 5 [(pb.enumvalue.json).string = "custom_string_here"];
}
```

Use this for conflict avoidance, migrations, or external naming contracts.

### Generated C++ namespace

The file option `(pb.file.cpp).namespace` decouples the generated C++ namespace
from the proto package:

```proto
import option "google/protobuf/cpp_options.proto";

package clock.time;

option (pb.file.cpp).namespace = "clock_time";
```

Edition 2026 schemas must remove the old C++ language-specific options
`cc_api_version`, `cc_utf8_verification`, and `cc_enable_arenas`.
