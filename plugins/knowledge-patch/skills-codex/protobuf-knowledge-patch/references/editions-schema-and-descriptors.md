# Editions, Schema, and Descriptors

Use this reference when selecting an Edition, migrating schema syntax,
configuring language features, changing imports or visibility, or consuming
field descriptors.

## Descriptor cardinality and presence

`31.0` deprecates `FieldDescriptor.label`, `getLabel`, and the `LABEL_*`
constants. Reflection, code generators, and dynamic-message code should ask
the semantic question directly:

- Use `isRepeated` for cardinality.
- Use `isRequired` for required proto2 or Editions fields.
- Use `hasPresence` for singular-field presence.
- In proto3-specific code, use `hasOptionalKeyword` only when the optional
  keyword itself matters, and `getRealContainingOneof` when oneof membership
  matters.

For Editions, the transitional `getLabel` behavior reduces every singular
field to `LABEL_OPTIONAL` and every repeated field to `LABEL_REPEATED`;
`hasOptionalKeyword` always returns false. Proto2 labels temporarily continue
to mirror the declared `optional`, `required`, or `repeated` keyword.

`34.0-migration` removes C++ `FieldDescriptor::label()`, Python
`FieldDescriptor.label`, and PHP `FieldDescriptor::getLabel()`.
Language-specific replacements include:

- C++: replace `has_optional_keyword()` with `has_presence()`, and replace
  `is_optional()` with `!is_required() && !is_repeated()`.
- PHP: use `hasPresence()`; the broken `hasOptionalKeyword()` is removed in
  `34.0`.
- Objective-C: after `-[GPBFieldDescriptor optional]` is removed, test
  `!required && fieldType == GPBFieldTypeSingle`.

## Edition 2024 adoption

The `edition-2024-announcement` material was provisional and targeted Edition
2024 at protobuf 32.x. Confirm the final compiler behavior in project tests
when migrating older preview schemas.

### C++ string representations

The default `features.(pb.cpp).string_type` changes from `STRING` to `VIEW`.
Generated string-field APIs therefore use view behavior unless the schema
overrides the feature.

Generated enum-name helpers also default to borrowed views through
`enum_name_uses_string_view`:

```cpp
absl::string_view Foo_Name(int);
```

Callers must not assume ownership or null termination. The `ctype` field option
is prohibited; select the representation with
`features.(pb.cpp).string_type`.

### Naming enforcement

`feature.enforce_naming_style` enables strict naming-style checking by default
to keep schemas round-trippable. A feature value can opt a file back into the
legacy naming style.

### Symbol visibility

`default_symbol_visibility` controls whether imported messages and enums are
visible to a schema without changing code generation. Edition 2024 defaults to
`EXPORT_TOP_LEVEL`: top-level symbols export, while nested symbols are local.
Use `export` and `local` to state exceptions:

```proto
local message LocalMessage {
  export enum ExportedNestedEnum {
    UNKNOWN_EXPORTED_NESTED_ENUM_VALUE = 0;
  }
}
```

`35.0` extends visibility checks to service input and output message types. A
service method now fails when its request or response type is not visible from
the service file.

### Java file layout and large enums

`nest_in_file_class` replaces the removed `java_multiple_files` option.
Edition 2024 emits classes in separate files by default. The default outer
class name is always the camel-cased proto filename plus `Proto`; for example,
`foo/bar_baz.proto` becomes `BarBazProto`. Use `java_outer_classname` to
override it.

The opt-in `large_enum` feature permits Java output beyond the normal enum
constant limit. The generated type emulates an enum but cannot support every
enum operation, including `switch`. The Java lite runtime gains support in
`34.0`, including correct handling of aliased values.

### Option-only imports

Use `import option` when a file needs custom options without importing the
source file's messages or enums as normal symbols. Place these declarations
after normal imports:

```proto
edition = "2024";
import option "bar.proto";

option (file_opt1) = true;
option (file_opt2) = {bar: true};
```

Bazel targets put these files in `option_deps`, which requires Bazel 8 or
later:

```build
proto_library(
  name = "foo",
  srcs = ["foo.proto"],
  option_deps = [":custom_option"],
)
```

Edition 2024 removes `import weak` and the `weak` field option. Schemas that
used weak imports to consume custom options without C++ or Go generation
should migrate to `import option`.

### Compiler and upb validation

`34.0` validates feature support on custom options and validates options and
features during parsing. Unsupported feature definitions can now fail instead
of passing unchecked.

The same release makes the C++ implementation reject
`[unverified_lazy = true]` on extensions; remove it from extension fields.
upb adds stricter `syntax` and `edition` validation for dynamic descriptor
data, and upb generators enable Edition 2024.

## Edition 2026 adoption

### Stricter naming and visibility

`35.0` implements the Edition 2026 `enforce_naming_style` behavior. Fields
whose names collide after language-specific conversion can be rejected.

`36.0-rc1` changes `default_symbol_visibility` to `STRICT`. Do not assume the
Edition 2024 `EXPORT_TOP_LEVEL` behavior survives an Edition upgrade; declare
the intended visibility and validate every imported service and message type.

### Descriptor limits and reserved numbers

The `enforce_proto_limits` feature is implemented in `36.0-rc1`, and Edition
2026 enables descriptor proto-limit enforcement by default. Descriptors that
previously exceeded those limits can fail after upgrading.

The compiler also rejects `INT_MAX` in reserved field-number declarations.
Replace sentinel reservations with valid explicit ranges or numbers.

### Generic service options

`36.0-rc1` warns on `py_generic_service`, `cc_generic_service`, and
`java_generic_service`. Treat these options as deprecated migration work and
move RPC generation to the appropriate language-specific plugin.

### Go API level

The `edition-2026-guide` changes the default
`features.(pb.go).api_level` from `API_OPEN` in Edition 2023 to `API_OPAQUE` in
Editions 2024 and 2026. Opaque output hides generated struct fields behind
accessors. Select `API_OPEN` to retain direct fields or `API_HYBRID` to expose
both fields and accessors during migration.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).api_level = API_HYBRID;
```

### Go enum-prefix stripping

Edition 2024 and newer accept `features.(pb.go).strip_enum_prefix` at file,
enum, or enum-value scope:

- `STRIP_ENUM_PREFIX_KEEP` is the default and preserves generated names.
- `STRIP_ENUM_PREFIX_GENERATE_BOTH` emits both forms for migration.
- `STRIP_ENUM_PREFIX_STRIP` removes the repeated enum-name prefix.

```proto
edition = "2026";

import option "google/protobuf/go_features.proto";

option features.(pb.go).strip_enum_prefix = STRIP_ENUM_PREFIX_STRIP;
```

### C++ repeated-field proxies

The `edition-2026-guide` adds `features.(pb.cpp).repeated_type`.
Selecting `PROXY` makes repeated-field accessors return `RepeatedFieldProxy`
rather than pointers or references to `RepeatedField` or `RepeatedPtrField`.
The default is `LEGACY`, so APIs change only where the feature is selected at
file or field scope.

```proto
edition = "2026";

import option "google/protobuf/cpp_features.proto";

option features.(pb.cpp).repeated_type = PROXY;
```

Edition 2026 schemas must also remove `cc_api_version`,
`cc_utf8_verification`, and `cc_enable_arenas`.

### Custom enum JSON strings

`edition-2026` permits an enum value to override its JSON string with
`(pb.enumvalue.json).string`. This complements field-level `json_name` for
conflict avoidance, migrations, and external naming:

```proto
enum Foo {
  FOO_UNSPECIFIED = 0;
  FOO_BAR = 5 [(pb.enumvalue.json).string = "custom_string_here"];
}
```

### C++ generated namespaces

`edition-2026` can decouple the generated C++ namespace from the proto package
with `(pb.file.cpp).namespace`:

```proto
import option "google/protobuf/cpp_options.proto";

package clock.time;

option (pb.file.cpp).namespace = "clock_time";
```

## Edition and compiler relationship

Per `release-lifecycle`, Editions have their own numbering. Edition 2023
requires `protoc` 27.0 or newer, while Edition 2024 requires 32.0 or newer.
Using a newer compiler does not require migrating older proto2, proto3, or
Edition schemas.
