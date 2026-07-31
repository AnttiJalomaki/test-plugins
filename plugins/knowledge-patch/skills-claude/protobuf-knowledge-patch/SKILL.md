---
name: protobuf-knowledge-patch
description: Protocol Buffers
version: 35.0
license: MIT
metadata:
  author: Nevaberry
---



# Protocol Buffers

Use this skill when upgrading Protocol Buffers, regenerating code, adopting
Editions, or diagnosing compiler/runtime behavior that differs across recent
releases. Start with the compatibility rules, then open the language or tooling
reference that matches the project.

## How to use this patch

1. Read the compiler, plugin, and runtime versions from the actual build.
2. Identify every language runtime loaded by the process; shared release
   numbers do not imply equal language-package majors.
3. Check generated-code compatibility before changing schema or application
   code.
4. Apply breaking migrations before enabling new Edition features.
5. Regenerate code and run boundary, JSON, recursion, and reflection tests that
   exercise the affected APIs.

## Reference index

| Reference | Topics |
|---|---|
| [Build and tooling](references/build-and-tooling.md) | CMake, Bazel, `protoc`, toolchains, packages, and distribution |
| [Compatibility and lifecycle](references/compatibility-and-lifecycle.md) | Generated-code/runtime rules, release numbering, support, and language baselines |
| [C++](references/cpp.md) | Removed APIs, string views, arenas, repeated fields, JSON, and safety checks |
| [Editions and schema](references/editions-and-schema.md) | Edition 2024/2026 features, visibility, naming, imports, options, and validation |
| [Java, C#, and Objective-C](references/java-csharp-objective-c.md) | Runtime, code-generation, descriptor, JSON, recursion, and packaging changes |
| [PHP and Ruby](references/php-and-ruby.md) | Runtime baselines, reflection, JSON, generated setters, RBS, and JRuby |
| [Python](references/python.md) | Removed APIs, map behavior, descriptors, formatting, NumPy, upb, and recursion |
| [Rust and Go](references/rust-and-go.md) | Rust generated APIs and traits; Go Editions API and enum naming |

## Breaking changes first

### Pair generated code with a compatible runtime

- Never run generated code against a runtime older than the compiler and plugin
  that produced it, including patch-version mismatches.
- C++ and Rust require an exact generated-code/runtime release match. C++ does
  not promise ABI stability across minor or patch releases.
- Most other runtimes accept major `V` generated code on runtime `V` and
  `V+1`, but not `V+2`. Older minor generated code can run on later runtimes in
  the same major.
- Python generated code from 3.20.0 onward has an extended descriptor-based
  window through at least runtime 8.x.
- Do not load multiple major versions of one runtime into a process. Security
  fixes can still require a paired runtime upgrade and regeneration.

### Respect the new tool and language floors

- Protobuf 30 requires C++17. Its Python 6.30 runtime requires Python 3.9 or
  newer.
- The Ruby runtime requires Ruby 3.1 or newer beginning with release 31.
- Protobuf 34 requires Bazel 8, Python 3.10, and PHP 8.2; Bazel also defaults
  dependency setup to Bzlmod rather than WORKSPACE.
- Regenerate Objective-C code older than 3.22 before using the 4.30 runtime.

### Replace descriptor label access

Do not infer modern cardinality or presence from `FieldDescriptor.label` or
`getLabel`. Use `isRepeated`, `isRequired`, and `hasPresence`; use
`hasOptionalKeyword` or the real containing oneof only when that exact proto3
distinction is needed. The deprecated C++, Python, and PHP label accessors are
removed in release 34.

### Audit removed runtime APIs

- C++: replace `Arena::CreateMessage` with `Arena::Create`, query an arena from
  the value, replace `JsonOptions` with `JsonPrintOptions`, and stop calling
  removed arena-taking field-container constructors.
- Python: build dynamic classes with `message_factory.GetMessageClass()` or
  `GetMessageClassesForFiles()` and replace the legacy generic service API with
  an RPC-specific plugin.
- Objective-C: migrate to the ordering-preserving `GPBUnknownFields` model and
  use error-returning merge APIs.
- PHP: use the nested `Field\Kind` and `Field\Cardinality` types and the public
  `Google\Protobuf\RepeatedField`.

### Treat debug strings as logs, not serialization

C++ debug-string and stringify output can be redacted, carries a randomized
process prefix, and is not parseable TextFormat. Serialize with the binary
format, or explicitly use a TextFormat printer when parseable unredacted text
is required.

### Validate repeated-field ranges before calling C++ APIs

Repeated-field access and mutation increasingly abort on invalid bounds.
Validate indices and counts before `Get`, `ExtractSubrange`, `DeleteSubrange`,
`UnsafeArenaExtractSubrange`, `ReleaseLast`, or `SwapElements`. Reflection code
must also stop calling the removed `MutableRepeatedFieldRef<T>::Reserve()`.

### Use absolute compiler output locations

The release 35 `protoc` CLI rejects file writes whose output path is relative.
Resolve generator output directories to absolute paths before invoking the
compiler.

## Editions quick reference

### Visibility is a schema rule

Edition 2024 defaults to exporting top-level symbols while keeping nested
symbols local. Use `export` and `local` explicitly when imports need a different
boundary. Visibility checking also applies to service request and response
types, so every method type must be visible from the service file.

### Prefer feature options over removed syntax

- Replace weak imports used only for options with `import option`; in Bazel,
  place them in `option_deps`.
- Replace `ctype` with `features.(pb.cpp).string_type`.
- Use `features.(pb.java).nest_in_file_class` rather than the removed
  `java_multiple_files` option in Edition 2024.
- Edition 2026 C++ schemas must remove `cc_api_version`,
  `cc_utf8_verification`, and `cc_enable_arenas`.

### Anticipate generated API changes

- Edition 2024 C++ string fields and enum-name helpers default to borrowed
  views. Copy when ownership or null termination is required.
- Go Editions 2024 and 2026 default to the Opaque API. Select `API_HYBRID` for
  a staged migration or `API_OPEN` to preserve direct field access.
- Edition 2026 can opt C++ repeated fields into proxy accessors and can set a
  generated C++ namespace independently of the proto package.
- Edition 2026 enum values can declare custom JSON strings.

## High-value runtime features

### C++

- Use `Arena::Ptr` and `Arena::UniquePtr` to express ownership of arena-related
  values.
- Use protobuf messages and enums directly as Abseil flag values; enum vectors
  are supported too.
- `FieldMaskUtil::TrimMessage` now trims repeated message fields.

### Python and upb

- Repeated scalar fields support NumPy-oriented access.
- Set the optional text-format recursion-depth limit for untrusted or deeply
  nested text.
- The upb runtime supports free-threaded Python and fixes races in lazy message
  initialization and repeated-field presence.

### Java, C#, PHP, and Ruby

- Java lite supports the `large_enum` feature.
- `Google.Protobuf.Tools` includes well-known-type proto files for packaged C#
  compiler invocations.
- PHP can emit JSON fields equal to their defaults and exposes reflection
  presence through `hasPresence()`.
- Ruby code generation can emit RBS declarations.

### Rust and Go

- Rust optional accessors return standard `Option`, and `MessageMut` requires
  `Send`.
- Rust view APIs accept more generic references and support const `ProtoStr`.
- Go can strip enum prefixes at file, enum, or enum-value scope, with a
  generate-both mode for migrations.

## Upgrade verification checklist

- Regenerate all checked-in outputs with the selected compiler and plugins.
- Confirm package majors rather than assuming the shared release number is the
  package major.
- Compile reflection code without deprecated label or optional-keyword APIs.
- Exercise invalid indices, recursive inputs, malformed descriptors, and
  out-of-range JSON numeric values.
- Compare any golden JSON whose map-key sorting or default-value emission may
  change.
- Check CMake fetching policy, Bazel toolchain registration, Bzlmod setup, and
  prebuilt-compiler selection explicitly.
- Verify schema visibility, option-only dependencies, generated namespaces,
  and language-specific Edition features.
