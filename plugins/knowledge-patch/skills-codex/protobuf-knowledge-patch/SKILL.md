---
name: protobuf-knowledge-patch
description: Protocol Buffers
version: 36.0-rc1
license: MIT
metadata:
  author: Nevaberry
---


# Protocol Buffers Knowledge Patch

Load this skill when upgrading Protocol Buffers, changing generated-code and
runtime combinations, adopting Editions, or maintaining protobuf build rules
and generated APIs. Start with compatibility and breaking changes, then open
the reference for the language or schema surface being changed.

## Working method

1. Identify the shared protobuf release, each language runtime package version,
   the `protoc` and plugin versions, and the Edition or syntax in every affected
   schema.
2. Inspect generated-code provenance before changing a runtime. Never assume
   that a shared release number is also a language package major.
3. Regenerate after release updates. Treat older-gencode support as a rolling
   upgrade allowance, not as the normal steady state.
4. Read the build/toolchain reference before changing Bazel, CMake, compiler,
   plugin, or output-path configuration.
5. Read the Editions reference before changing imports, visibility, naming,
   language features, or custom options.
6. Exercise newly strict failure paths: bounds, recursion, UTF-8, JSON,
   descriptor parsing, time ranges, presence, and type validation.
7. Prefer the repository's manifests, lockfiles, generated headers, tests, and
   observed runtime behavior when they conflict with assumptions outside this
   patch.

## Reference index

| Reference | Topics |
| --- | --- |
| [Compatibility, builds, and releases](references/compatibility-builds-and-releases.md) | Gencode/runtime windows, language package versions, release policy, Bazel, CMake, `protoc`, plugins |
| [Editions, schema, and descriptors](references/editions-schema-and-descriptors.md) | Edition 2024 and 2026 defaults, visibility, naming, imports, features, descriptor presence |
| [C++ runtime and generated APIs](references/cpp-runtime-and-generated-apis.md) | Removed APIs, string views, arenas, repeated fields, redaction, range checks, JSON and time conversion |
| [Python runtime and generated APIs](references/python-runtime-and-generated-apis.md) | Runtime baselines, removed reflection APIs, setters and maps, upb, NumPy and buffer support, recursion |
| [Java, C#, and Objective-C](references/java-csharp-and-objective-c.md) | Java generation and JSON, C# recursion and nullability, Objective-C unknown fields and descriptors |
| [PHP, Ruby, and Rust](references/php-ruby-and-rust.md) | Runtime baselines, PHP validation and reflection, Ruby/JRuby behavior, Rust generated API changes |

## Breaking compatibility rules

### Keep generators and runtimes in a supported relationship

- A runtime must never be older than the `protoc` and plugin release that
  produced its generated code, including patch releases.
- Most languages support major-V generated code through runtime major V+1.
  Runtime V+2 or later is outside the normal window.
- C++ and Rust require exact generated-code/runtime release matches.
- C++ does not promise ABI stability across minor or patch releases.
- Python generated code from 3.20.0 onward has an extended descriptor-based
  window through at least runtime 8.x.
- Do not load multiple major runtime versions into one process.
- Security changes can require a paired runtime update and regeneration even
  when the normal window would otherwise allow the combination.

### Do not infer package majors from the shared release

The shared `minor.point` release maps to language-specific packages. For
example, shared release `34.1` maps to Java `4.34.1` and C# `3.34.1`.
Confirm the package coordinate and runtime major for each language separately.

Important baseline changes include:

- C++ requires C++17.
- Python 6.30 requires Python 3.9; the v34 line requires Python 3.10.
- PHP v34 requires PHP 8.2.
- Ruby v31 requires Ruby 3.1, while the v36 line drops Ruby 3.1.
- Objective-C's first breaking runtime line is 4.30.
- Bazel 8 is the minimum for protobuf v34 and defaults dependency management
  to Bzlmod.

## Descriptor presence migration

Do not use descriptor labels as a proxy for cardinality or presence:

- Replace C++ `FieldDescriptor::label()`, Python `FieldDescriptor.label`, and
  PHP `FieldDescriptor::getLabel()`; these accessors are removed.
- Use `isRepeated` for cardinality, `isRequired` for required fields, and
  `hasPresence` for singular-field presence.
- In C++, replace `has_optional_keyword()` with `has_presence()`, and express
  optional as `!is_required() && !is_repeated()`.
- In PHP, use `hasPresence()`; `hasOptionalKeyword()` is removed.
- When old Objective-C code tested `optional`, use
  `!required && fieldType == GPBFieldTypeSingle`.
- Treat proto3 optional keywords and real containing oneofs as distinct
  questions from general field presence.

## Build and toolchain migrations

### CMake

- Dependency providers are automatic: installed dependencies are preferred,
  and pinned dependencies are fetched when missing.
- Set `protobuf_LOCAL_DEPENDENCIES_ONLY=ON` to prohibit fetching, or
  `protobuf_FORCE_FETCH_DEPENDENCIES=ON` to force it.
- Installed packages no longer expose private protoc generator headers.
- Protobuf's own CMake tests are disabled by default; enable them explicitly
  in source builds and CI that need the targets.
- The C++ CocoaPods distribution is gone; consume the C++ runtime from the
  release artifacts.

### Bazel

- Migrate protobuf v34 builds to Bazel 8 and Bzlmod.
- Prefer registered platform toolchains with
  `--incompatible_enable_proto_toolchain_resolution`.
- Proto rule flags moved under `--@protobuf//bazel/flags`; do not depend on the
  removed native `--proto_toolchain_for*` and `--proto_compiler` flags.
- Replace `ProtoInfo.transitive_imports` with `transitive_sources`.
- Remove `--define=protobuf_allow_msvc`; the temporary flag is gone.
- Account for `prefer_prebuilt_proto` defaulting to true.
- Migrate away from `internal_py_proto_library` before its announced removal.

### Compiler and plugin invocations

- Use absolute output locations: relative `protoc` file output paths fail.
- `protoc` supports language-specific `--<lang>_prefix` options.
- Standalone plugins are available as Bazel targets.
- Keep field names below the compiler's 2^16-character limit.

## Edition adoption risks

### Edition 2024

- C++ string fields default to `VIEW`, and enum-name helpers default to
  `absl::string_view`.
- Naming-style enforcement is enabled by default.
- Symbol visibility defaults to `EXPORT_TOP_LEVEL`: top-level symbols export,
  while nested symbols remain local unless modifiers say otherwise.
- Visibility validation also applies to service input and output types.
- Java generates types in their own files by default; `nest_in_file_class`
  replaces `java_multiple_files`.
- Use `import option` and Bazel `option_deps` for custom-option-only imports.
- Remove `import weak`, the `weak` field option, and `ctype`.

### Edition 2026

- Default symbol visibility is `STRICT`.
- Descriptor proto-limit enforcement is enabled by default.
- Naming enforcement rejects field names that collide after
  language-specific conversion.
- Go defaults to the Opaque API in Editions 2024 and 2026.
- C++ repeated accessors can opt into `RepeatedFieldProxy` with
  `features.(pb.cpp).repeated_type = PROXY`; the default remains `LEGACY`.
- Remove `cc_api_version`, `cc_utf8_verification`, and `cc_enable_arenas`.
- C++ can set a generated namespace with `(pb.file.cpp).namespace`.
- Enum values can set a custom JSON spelling with
  `(pb.enumvalue.json).string`.

## High-impact C++ changes

- Descriptor names, type names, and length-delimited unknown fields can return
  borrowed `absl::string_view`; copy when ownership or null termination is
  required.
- Debug strings redact `debug_redact` fields, include a randomized prefix, and
  are not parseable TextFormat.
- Replace removed arena and JSON APIs, and do not construct repeated or map
  containers directly from an `Arena*`.
- Validate every index and range before repeated-field access. `Get`,
  `ExtractSubrange`, `DeleteSubrange`, `UnsafeArenaExtractSubrange`,
  `ReleaseLast`, and `SwapElements` enforce bounds.
- Do not retain or access a cleared arena oneof object; debug and ASAN builds
  deliberately expose the invalid lifetime.
- Do not copy a parent message into one of its descendants.
- Use `constexpr` directly instead of `PROTOBUF_CONSTEXPR`.

## High-impact dynamic-language changes

### Python

- Replace removed `MessageFactory`, `SymbolDatabase`, and reflection creation
  APIs with `message_factory.GetMessageClass()` or
  `GetMessageClassesForFiles()`.
- Closed-enum setters reject invalid values, and integer or enum fields reject
  `bool`.
- Scalar-map `setdefault` requires key and value; message-valued maps reject
  `setdefault`.
- Add recursion bounds for untrusted text-format input.
- Expect scalar `GetOptions()` results from upb to be immutable.

### PHP and Ruby

- PHP JSON parsing is strict about types, ranges, duplicate oneof members, and
  integer values; serialization rejects numeric `Infinity` and `NaN`.
- PHP now honors proto2 and Editions scalar defaults and aligns pure-PHP type
  checks with upb.
- Ruby and PHP reject nonnumeric strings for numeric JSON fields.
- JRuby uses FFI by default and remains best-effort rather than officially
  supported.

### Rust

- Keep generated code and runtime on the exact same release.
- `MessageMut` requires `Send`.
- Generated `_opt()` accessors return standard `Option`.
- Use `MapValue` instead of the removed `ProxiedInMapValue` alias.
- For minor 36 and later, crate versions no longer use a `-release` suffix.

## Validation checklist

- Regenerate with the selected compiler and every selected language plugin.
- Verify generator/runtime compatibility for each language package.
- Compile C++ with C++17 and treat new `[[nodiscard]]` warnings as actionable.
- Test Bazel toolchain resolution, Bzlmod dependencies, and prebuilt compiler
  selection in a clean environment.
- Validate schema imports, custom-option feature support, naming, symbol
  visibility, reserved numbers, and Edition feature choices.
- Exercise malformed JSON, invalid UTF-8, over-deep messages, out-of-range
  time values, repeated-field bounds, and malformed descriptor data.
- Compare deterministic or sorted Java JSON output where map key order matters.
- Test Python free-threaded execution when that interpreter mode is supported.
- Recheck Swift call sites affected by corrected Objective-C nullability.
- Confirm generated Rust identifiers after schemas introduce `Xyz` and
  `XyzView` siblings.
