---
name: protobuf-knowledge-patch
description: Protocol Buffers
version: 36.0-rc1
license: MIT
metadata:
  author: Nevaberry
---


# Protocol Buffers Knowledge Patch

Use this skill when upgrading Protocol Buffers, adopting Editions, regenerating
code, changing compiler or runtime versions, or diagnosing behavior that differs
across language runtimes. Prefer the project's manifests, generated-code
headers, build configuration, and tests when they disagree with general
guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [compatibility-and-lifecycle.md](references/compatibility-and-lifecycle.md) | Gencode/runtime windows, language package versions, release cadence, supported lines |
| [build-and-tooling.md](references/build-and-tooling.md) | CMake, Bazel, Bzlmod, `protoc`, plugins, distribution |
| [editions-and-schema.md](references/editions-and-schema.md) | Editions 2024/2026, imports, visibility, naming, language feature options |
| [cpp.md](references/cpp.md) | C++ API removals, arenas, descriptors, repeated fields, JSON, validation |
| [python.md](references/python.md) | Python/upb APIs, maps, descriptors, recursion, buffers, free-threading |
| [java-csharp-objective-c.md](references/java-csharp-objective-c.md) | Java, C#, and Objective-C generation and runtime changes |
| [php-and-ruby.md](references/php-and-ruby.md) | PHP validation, reflection, JSON, Ruby and JRuby support |
| [rust-and-go.md](references/rust-and-go.md) | Rust generated APIs and Go Editions features |

## Start with compatibility

Before changing any protobuf dependency:

1. Identify the `protoc` version, each language plugin version, generated-code
   version, and runtime version independently.
2. Never run generated code on a runtime older than its compiler/plugin,
   including a patch-level mismatch.
3. Require exact generated-code/runtime matches for C++ and Rust. Treat C++ ABI
   as unstable across minor and patch releases.
4. For most other languages, expect major V generated code to work through
   runtime V+1, but not V+2. Python gencode from 3.20.0 has a longer,
   descriptor-based window.
5. Regenerate on every release update even where an older generated artifact is
   temporarily supported.
6. Do not load multiple runtime majors into one process.

Consult
[compatibility-and-lifecycle.md](references/compatibility-and-lifecycle.md)
before interpreting shared release numbers: a protobuf release number does not
equal every language package's major version.

## Breaking migration checkpoints

### Toolchain and language baselines

- C++ requires C++17 beginning with the v30 migration.
- Python requires 3.9 for 6.30, then 3.10 for v34.
- Ruby requires 3.1 at v31, but 36.0-rc1 removes Ruby 3.1 support.
- PHP requires 8.2 at v34.
- Bazel 8 is the minimum at v34, and Bzlmod becomes the default dependency
  mode. Bazel 7 and implicit WORKSPACE assumptions must be migrated.
- Relative `protoc` output paths are rejected in v35; pass absolute output
  locations.

### Removed reflection and construction APIs

- Replace C++ `FieldDescriptor::label()` with `is_repeated()`,
  `is_required()`, and presence queries.
- Replace Python `FieldDescriptor.label` and PHP
  `FieldDescriptor::getLabel()` with their cardinality/presence APIs.
- Replace Python `MessageFactory` and `SymbolDatabase` creation/prototype
  methods with module-level `message_factory.GetMessageClass()` or
  `GetMessageClassesForFiles()`.
- Replace C++ `Arena::CreateMessage` with `Arena::Create`; use
  `value->GetArena()` instead of `Arena::GetArena`.
- Stop constructing `RepeatedField`, `RepeatedPtrField`, or `Map` directly from
  an `Arena*`.
- Replace `JsonOptions` with `JsonPrintOptions`.

Read the language reference before applying a mechanical rename: several
removed APIs have no direct replacement or require a semantic rewrite.

### Descriptor label migration

Do not infer all field semantics from a label:

- use repetition predicates for cardinality;
- use required predicates for proto2 or Editions required fields;
- use presence predicates for singular-field presence;
- use optional-keyword or real-oneof queries only when that exact distinction
  matters.

Editions singular fields do not preserve proto2-style label meaning. The
deprecated accessors were removed in v34.

### Generated-code behavior changes

- Treat debug-string output as redacted, randomized, and non-parseable. Use
  binary serialization, or an explicit TextFormat printer when parseable,
  unredacted text is required.
- Audit borrowed C++ string returns: descriptor names, type names, and Edition
  string/enum-name APIs may return `absl::string_view`; `data()` need not be
  null-terminated.
- Validate repeated-field indices and ranges before every checked operation.
  New checks can abort rather than preserve formerly unchecked behavior.
- Expect stricter JSON, descriptor, UTF-8, recursion, and scalar-conversion
  validation across runtimes.

## Build-system quick reference

### CMake

The old `protobuf_*_PROVIDER` switches are gone. Installed dependencies are
preferred and pinned dependencies are fetched when absent:

```sh
cmake . -Dprotobuf_LOCAL_DEPENDENCIES_ONLY=ON
cmake . -Dprotobuf_FORCE_FETCH_DEPENDENCIES=ON
```

Use the first setting to prohibit fetching and the second to force it. Enable
protobuf's tests explicitly when source builds or CI need them; v34 no longer
builds those tests by default.

### Bazel

For v34 migrations:

- move from Bazel 7/WORKSPACE assumptions to Bazel 8/Bzlmod;
- enable normal proto toolchain resolution and register platform toolchains;
- replace `ProtoInfo.transitive_imports` with `transitive_sources`;
- migrate internal Python proto rules to public rules;
- account for `prefer_prebuilt_proto` defaulting to true.

The temporary v30 Windows Bazel escape hatch
`--define=protobuf_allow_msvc=true` is removed in v34. See
[build-and-tooling.md](references/build-and-tooling.md) for the version-scoped
MSVC behavior and exact flag replacements.

## Editions quick reference

### Edition 2024

Review these defaults and removals:

- C++ string fields and enum-name helpers move toward borrowed views.
- strict naming-style enforcement is enabled;
- top-level symbols are exported while nested symbols are local by default;
- Java classes default to separate files, and the outer class derives from the
  proto filename plus `Proto`;
- `import option` replaces weak custom-option import patterns;
- `ctype`, `import weak`, and the weak field option are unavailable.

Use `option_deps` for Bazel option-only imports; it requires Bazel 8 or later.

### Edition 2026

Migration-sensitive defaults include:

- strict symbol visibility;
- protobuf descriptor-limit enforcement;
- Go's opaque generated API;
- C# nullable-reference generation;
- removal of C++ `cc_api_version`, `cc_utf8_verification`, and
  `cc_enable_arenas`.

Opt-in features include C++ repeated-field proxy accessors, Go enum-prefix
stripping, custom JSON strings for enum values, and a generated C++ namespace
separate from the proto package. See
[editions-and-schema.md](references/editions-and-schema.md) for syntax and
scope.

## Validation-sensitive upgrades

When tests fail only after an upgrade, check whether the input was always
invalid but previously tolerated:

- closed-enum setters reject unknown values under Edition 2023;
- bool-to-integer or bool-to-enum Python assignments are rejected;
- PHP and Ruby reject malformed JSON numeric strings;
- PHP rejects range errors, fractional integer values, duplicate oneof fields,
  wrong string types, and non-finite numeric serialization;
- C++ repeated-field and time parsing operations enforce bounds;
- upb validates descriptor syntax and edition;
- recursion limits cover more JSON, binary, and text-format paths;
- reserved field number `INT_MAX` is rejected;
- Edition 2026 validates field-name collisions and descriptor limits.

Update fixtures and error handling instead of disabling validation unless the
reference documents an explicit configurable limit.

## Language-focused workflow

For a runtime upgrade:

1. Read the compatibility policy and the target language reference.
2. Regenerate into a clean output directory using absolute paths.
3. Compile with deprecations visible.
4. Run malformed-input, recursion-depth, unknown-field, oneof, reflection, and
   JSON round-trip tests relevant to that runtime.
5. Inspect generated API name changes, nullability, ownership, and borrowed
   returns.
6. Pin build flags where a changed default affects reproducibility.

## Evidence rules

- Treat provisional announcements as planning guidance until the corresponding
  release behavior appears in the project's compiler/runtime.
- Scope advice to the actual language package and release; shared protobuf
  release numbers can map to different package majors.
- Prefer explicit feature options when a generated API shape matters across an
  Editions migration.
- When a stricter check exposes invalid data, preserve the new invariant and
  repair the producer.
- Verify experimental APIs before depending on them; they may disappear
  without a major-version transition.
