---
name: protobuf-knowledge-patch
description: Protocol Buffers
version: 35.0
license: MIT
metadata:
  author: Nevaberry
---


# Protocol Buffers Knowledge Patch

Use this skill when changing Protocol Buffers schemas, generators, build rules,
or language runtimes. Identify the compiler, generated-code, runtime, language,
and Edition versions before applying advice; those versions do not move in
lockstep.

## Reference index

| Reference | Topics |
| --- | --- |
| [Compatibility, builds, and releases](references/compatibility-builds-and-releases.md) | Gencode/runtime compatibility, release numbering and support, Bazel, CMake, `protoc` paths |
| [C++ runtime and generated APIs](references/cpp-runtime-and-generated-apis.md) | C++17, views, arenas, repeated fields, removed APIs, validation, debug output |
| [Editions, schema, and descriptors](references/editions-schema-and-descriptors.md) | Editions 2024 and 2026, visibility, naming, imports, language feature options, descriptor labels |
| [Java, C#, and Objective-C](references/java-csharp-and-objective-c.md) | Java enum generation, recursion, C# tooling, Objective-C unknown fields and generated APIs |
| [PHP, Ruby, and Rust](references/php-ruby-and-rust.md) | Runtime baselines, PHP validation, Ruby behavior and RBS, Rust generated API changes |
| [Python runtime and generated APIs](references/python-runtime-and-generated-apis.md) | Python baselines, removed reflection APIs, maps, conversions, upb, NumPy, free threading |

## First checks

1. Record the shared Protobuf release and each package's language-specific
   major. A shared release such as `34.1` does not imply package major 34.
2. Record the `protoc` and plugin version that produced checked-in generated
   code. Never pair gencode with an older runtime.
3. For C++ and Rust, require an exact generated-code/runtime release match.
   Do not rely on the wider compatibility window used by most languages.
4. Regenerate on every release update. Compatibility with older gencode is a
   rolling-upgrade allowance, not the preferred steady state.
5. Inspect the schema syntax or Edition separately from compiler and runtime
   versions. Edition numbers are independent.

## Breaking migrations at a glance

### Language and tool baselines

- C++ requires C++17 from the 30.0 migration onward.
- Python requires 3.9 for Python runtime 6.30 and 3.10 from the 34.0
  migration; Ruby requires 3.1 from 31.0; PHP requires 8.2 from 34.0.
- Protobuf 34 requires Bazel 8, whose default dependency mode is Bzlmod.
  Migrate WORKSPACE-era dependency setup while upgrading.
- CMake source builds no longer build tests by default. Explicitly enable test
  targets in CI jobs that need them.

### Removed descriptor-label APIs

Do not model field semantics through `FieldDescriptor.label`. The C++
`label()`, Python `label`, and PHP `getLabel()` accessors were removed in the
34.0 migration after their 31.0 deprecation.

Use semantic queries instead:

- `isRepeated` for cardinality;
- `isRequired` for required proto2 or Editions fields;
- `hasPresence` for singular-field presence;
- `hasOptionalKeyword` only for proto3 optional-keyword questions while the
  language binding still provides it;
- `getRealContainingOneof` for real-oneof membership.

For C++ code that previously called `is_optional()`, use
`!is_required() && !is_repeated()`. For Objective-C, replace the removed
`optional` property with `!required && fieldType == GPBFieldTypeSingle`.

### C++ views and removals

- Treat descriptor names, `MessageLite::GetTypeName`, and
  `UnknownField::length_delimited` as borrowed `absl::string_view` values.
  Copy when ownership is needed; never assume `data()` is null-terminated.
- Replace `Arena::CreateMessage` with `Arena::Create`, `Arena::GetArena` with
  `value->GetArena()`, and `JsonOptions` with `JsonPrintOptions`.
- Stop constructing `RepeatedField`, `RepeatedPtrField`, or `Map` directly
  from `Arena*`; their arena-taking constructors were removed.
- Replace `PROTOBUF_CONSTEXPR` with the `constexpr` language keyword.
- Stop calling removed repeated-field reflection `Reserve()` and validate all
  indices and ranges before access or mutation.

### Python removals and stricter assignment

- Replace removed `reflection.ParseMessage`, `reflection.MakeClass`, factory
  prototype/creation methods, and symbol-database creation methods with
  `message_factory.GetMessageClass()` or `GetMessageClassesForFiles()`.
- Replace legacy generic service interfaces with an RPC-specific generator
  plugin. The C++-extension-only `GetDebugString` has no replacement.
- Pass both key and value to scalar-map `setdefault`; never call `setdefault`
  on a message-valued map.
- Do not assign `bool` to enum or integer fields. Catch `TypeError`, not
  `AttributeError`, for invalid `Timestamp` or `Duration` conversions.
- Remove JSON `float_precision` and text-format `float_format` /
  `double_format` arguments.

### PHP, Ruby, and Objective-C removals

- PHP uses `Google\Protobuf\Field\Kind`,
  `Google\Protobuf\Field\Cardinality`, and
  `Google\Protobuf\RepeatedField`; the older underscored/internal types are
  removed.
- Ruby and PHP reject nonnumeric strings for numeric JSON fields. PHP also
  rejects out-of-range values, fractional numbers for integers, duplicate
  oneof members, non-string values for strings, and numeric `Infinity`/`NaN`.
- Objective-C uses the ordering-preserving `GPBUnknownFields` model. Regenerate
  gencode older than 3.22 and migrate removed merge, duration, text-format,
  syntax, and field-option APIs.

## Editions quick reference

### Edition 2024 defaults

- C++ string fields and enum-name helpers use borrowed string-view behavior by
  default.
- Strict naming-style enforcement is enabled by default.
- Symbol visibility defaults to `EXPORT_TOP_LEVEL`; top-level declarations
  are exported and nested declarations are local unless marked otherwise.
- Java generates separate files by default. The outer class defaults to the
  camel-cased proto filename plus `Proto`.
- Use `import option` for option-only dependencies, after normal imports. In
  Bazel 8+, list them in `option_deps` rather than `deps`.
- Remove `import weak`, the `weak` field option, and `ctype`; select C++ string
  representation with `features.(pb.cpp).string_type`.

At 35.0, visibility checks also apply to service request and response types.
Every method type must be visible from the service file.

### Edition 2026 choices

- Field-name collisions after language-specific name conversion are rejected
  under the Edition 2026 naming-style feature value.
- Go defaults to `API_OPAQUE` in Editions 2024 and 2026. Choose `API_HYBRID`
  while migrating direct field access, or `API_OPEN` to preserve it.
- C++ repeated accessors remain `LEGACY` unless
  `features.(pb.cpp).repeated_type = PROXY` is selected.
- Go enum-prefix stripping is opt-in: keep names, generate both forms for a
  migration, or strip the repeated enum prefix.
- Remove `cc_api_version`, `cc_utf8_verification`, and `cc_enable_arenas` from
  Edition 2026 schemas.
- Enum values can override their JSON spelling with
  `(pb.enumvalue.json).string`.
- A file can decouple its generated C++ namespace from its proto package with
  `(pb.file.cpp).namespace`.

## Build-system decisions

### CMake dependencies

The old `protobuf_*_PROVIDER` switches are gone. CMake uses installed
dependencies when available and fetches pinned copies when missing.

```sh
cmake . -Dprotobuf_LOCAL_DEPENDENCIES_ONLY=ON
cmake . -Dprotobuf_FORCE_FETCH_DEPENDENCIES=ON
```

Use the first setting to forbid fetching and the second to force it. Do not
include protoc's private generator headers from a CMake install; they are no
longer installed.

### Bazel toolchains

Prefer normal platform toolchain registration with
`--incompatible_enable_proto_toolchain_resolution`, the Bazel 9 default. The
short-term protobuf-repository flags are documented in the build reference.
Do not use native `--proto_toolchain_for*` or `--proto_compiler` flags.

At 34.0, `prefer_prebuilt_proto` defaults to true. Pin it when reproducibility
depends on selecting a source-built compiler. Replace `ProtoInfo`'s removed
`transitive_imports` with `transitive_sources`.

### Generator outputs

From 35.0, `protoc` rejects any relative file output path. Resolve output
locations to absolute paths before invoking generators, including through
build wrappers.

## Validation and security-sensitive behavior

- C++ debug strings are log-oriented, randomized, redacted when annotated,
  and not parseable TextFormat. Use binary serialization for interchange or
  an explicit `TextFormat` printer when parseable unredacted text is required.
- Debug builds poison cleared arena oneofs and reject copying a parent message
  into its descendant. Treat failures as lifetime or graph-ancestry bugs.
- Closed enums in Edition 2023 reject invalid Python/upb setter values.
- Recursion limits cover more Java JSON, C# JSON, Python, and upb nesting.
  Configure Python text-format depth limits for untrusted input.
- Protoc validates feature support on custom options and validates options and
  features while parsing. upb also rejects malformed `syntax` and `edition`
  descriptor data more strictly.

## Applying this patch

1. Open the topic reference that matches the code being changed.
2. Apply only guidance introduced at or below the project's shared release;
   account for language-package majors separately.
3. Prefer the repository's manifests, generated headers, code, and tests when
   they establish a more specific pinned behavior.
4. When upgrading, update compiler, plugins, runtimes, checked-in gencode,
   toolchain flags, and CI together, then exercise malformed and boundary
   inputs that are newly rejected.
