---
name: c-cpp-knowledge-patch
description: C and C++
version: null
license: MIT
metadata:
  author: Nevaberry
---


# C and C++ Knowledge Patch

Use this skill when changing, reviewing, or diagnosing C or C++ projects that
build with recent Clang, GCC, libc++, or libstdc++ toolchains. Start with the
project's declared language dialect, compiler family, compiler version, target,
standard library, sanitizer set, and binary-compatibility constraints. Do not
infer conformance from a language-mode flag alone.

Prefer project evidence over generic advice:

1. Read the build manifest, presets, toolchain files, and CI matrix.
2. Confirm the actual compiler and standard-library versions from verbose build
   output rather than the executable name alone.
3. Identify objects, modules, plugins, libraries, and serialized layouts that
   cross compiler-version or ABI boundaries.
4. Reproduce with the project's exact dialect and warning policy.
5. Use the references below for changed defaults, removed spellings, ABI
   transitions, newly available facilities, and known conformance gaps.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration and ABI](references/migration-and-abi.md) | Dialect/default changes, removals, source compatibility, ABI transitions, porting controls |
| [Language and standards](references/language-and-standards.md) | C23/C2y, C++20/23/26/29, modules, constexpr, templates, conformance status |
| [Diagnostics and safety](references/diagnostics-and-safety.md) | Warnings, machine-readable diagnostics, analyzers, sanitizers, thread safety |
| [Code generation and builtins](references/codegen-and-builtins.md) | Optimization semantics, attributes, builtins, LTO, debug/profile controls |
| [Targets and offload](references/targets-and-offload.md) | Architecture flags, target defaults, CUDA/HIP, OpenMP, OpenACC, platform drivers |
| [Libraries and tooling](references/libraries-and-tooling.md) | libstdc++, clang-format, libclang, Python bindings, AST tooling, GCC plugins |

## Upgrade triage

### Pin language modes explicitly

GCC 15 changed the C default to GNU C23, and GCC 16 changed the C++ default to
GNU C++20. An unqualified `gcc` or `g++` invocation can therefore change source
meaning during a compiler upgrade. Set `-std=` explicitly in production builds,
feature probes, generated configure tests, and standalone tooling.

Older Autoconf can counterintuitively force GCC 16 back to GNU C++11. If modern
facilities disappear only in configured builds, inspect the generated flags and
regenerate with a corrected Autoconf setup.

### Treat mixed compiler objects as an ABI risk

Rebuild a complete binary boundary when it contains any of these transitions:

- Clang record returns around Clang 21;
- Windows scalar and vector deleting destructors around Clang 22;
- Microsoft or Itanium mangling changed by Clang 20;
- Arm empty-structure calling conventions;
- Solaris 8-bit integer typedef identity under GCC 16;
- libstdc++ component or `std::variant` layout changes under GCC 16.

Compatibility switches and macros are migration bridges, not a substitute for
a coherent toolchain. See the migration reference for the precise boundary and
temporary control.

### Audit assumptions invalidated by optimization

Do not detect pointer wrap with `ptr + offset < ptr`; pointer-addition overflow
can be optimized as undefined. Validate the offset before pointer arithmetic or
use integer-domain checks. Likewise, type-punning through incompatible pointer
types can be newly exposed by stronger type-based alias analysis.

Do not inspect an automatic union's complete representation after `{0}` and
assume padding is zero. Hashing, serialization, comparison, and information
exposure must operate on initialized members or explicitly initialized storage.

### Replace removed interfaces, do not suppress around them

Important removals include Concepts TS mode, legacy AVX10 width spellings,
Clang trivial-relocation opt-ins, GCC JSON diagnostics, several analyzer checker
names, old AST matchers, and direct MMX implementation builtins. Follow the
replacement table in the detailed references before adding compatibility flags.

## High-value compiler controls

### Diagnostics pipelines

Use SARIF for machine-readable GCC diagnostics. GCC 15 deprecates the old JSON
format and GCC 16 removes it. If a parser expects flat C++ diagnostics, account
for GCC 16's nested explanations or request the old flat presentation.

Clang 22 makes incompatible C pointer types an error by default. Demote only as
a bounded migration measure:

```sh
clang -Wno-error=incompatible-pointer-types ...
```

Clang warning-suppression mappings now use the last matching entry, so review
ordered rules after an upgrade.

### Undefined behavior and sanitizers

Request `-fsanitize=vptr` explicitly with Clang 21 or later when virtual-pointer
checks are required; it is no longer implied by `-fsanitize=undefined`.

Use `-fsanitize=pointer-overflow` to find invalid pointer-wrap idioms. Clang also
offers realtime checking for unsafe operations in nonblocking functions,
experimental type-aliasing checks, allocation-token instrumentation, granular
overflow-pattern exclusions, and sanitizer-aware trap diagnostics. Select only
controls supported by the compiler pinned in the build.

### Modules

Clang 22 enables reduced BMIs by default. Two-phase module builds must preserve
the reduced-BMI workflow and cannot depend on discarded implementation details.
Clang 20's non-experimental control is `-fmodules-reduced-bmi`.

GCC's standard modules remain experimental. Build them deliberately and keep
feature probing separate from assumptions about full C++20 module semantics.

## Language-mode guidance

### C

In C23, `f()` means no parameters, and `bool`, `true`, `false`, `nullptr`, and
`thread_local` are keywords. Rename conflicting identifiers and declare actual
function parameter types. Include owning headers rather than relying on C++
library transitive inclusions.

Clang and GCC expose useful C23 and C2y features, but their sets differ. Gate
individual facilities with compiler/version or feature tests, and keep fallback
paths when using `_Countof`, named loops, `#embed`, `defer`, integer-limit
operators, or draft generic-selection behavior.

### C++

Do not equate `-std=c++23`, `-std=c++2c`, or `-std=c++2d` with complete standard
support. Check the exact facility, library implementation, target, and ABI.
Windows coroutines, modules, reflection, contracts, constexpr library work, and
several template edge cases have compiler-specific limitations.

GCC reflection requires both C++26 mode and `-freflection`:

```sh
g++ -std=c++26 -freflection source.cc
```

Clang's trivial relocation remains a builtin operation, but explicit
replaceability/relocatability markers were removed. Never replace relocation of
a non-trivially-copyable object with raw `memcpy` merely because it is reported
relocatable.

## Library and tooling checks

### libstdc++

Add direct includes for every used declaration. In particular, choose
`<stdint.h>` for global fixed-width typedefs, `<cstdint>` for `std::` typedefs,
and `<ostream>` for stream declarations and manipulators. Remove compatibility
headers such as `<cstdbool>` and `<cstdalign>`.

Unoptimized GCC 15 builds enable libstdc++ assertions by default. Treat an
assertion as evidence of an invalid operation before considering the temporary
`_GLIBCXX_NO_ASSERTIONS` opt-out.

### Formatting and compiler APIs

Version-control the formatter version with its configuration. Recent options
include renamed enum/boolean values and keys whose old forms are deprecated.

Consumers embedding Clang must link dependencies explicitly after library
splits. For libclang Python bindings, handle `None` and raised errors where old
versions returned null cursor objects or silent status values.

## Verification checklist

- Confirm explicit C and C++ dialect flags in every build path.
- Compare warning groups and error promotions under the exact compiler version.
- Run ABI-sensitive tests with all objects rebuilt, then test any intentional
  mixed-version boundary separately.
- Exercise optimized and sanitizer builds; debug-only success is insufficient.
- Validate machine-readable diagnostic consumers against emitted SARIF.
- Rebuild module artifacts instead of reusing stale BMIs or header units.
- Check formatter, analyzer, plugin, and binding clients against renamed or
  removed APIs.
- For offload builds, verify host and device target feature detection, runtime
  version requirements, and actual lowering support.
- Consult the standards reference before promising a facility solely because a
  language-mode flag is accepted.
