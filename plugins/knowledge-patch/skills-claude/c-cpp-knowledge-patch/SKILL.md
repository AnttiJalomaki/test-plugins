---
name: c-cpp-knowledge-patch
description: C and C++
version: null
license: MIT
metadata:
  author: Nevaberry
---


# C and C++ Compatibility Guide

Use this skill when upgrading Clang, GCC, libstdc++, or compiler-adjacent
tooling; selecting a modern C or C++ language mode; diagnosing a new warning,
error, ABI mismatch, optimizer change, or sanitizer result; or adopting recent
language and library facilities.

Treat an accepted `-std=` flag as mode selection, not proof that every proposal
is implemented. Check compiler identity and exact version, target triple,
standard-library implementation, language mode, build flags, and feature-test
results before choosing a workaround.

## Reference index

| Reference | Topics |
|---|---|
| [migration-and-abi.md](references/migration-and-abi.md) | Default dialect changes, source migration, ABI breaks, removals, and compatibility flags |
| [c-language-and-standards.md](references/c-language-and-standards.md) | C23, C2y, GNU C extensions, conformance limits, and C-specific compatibility |
| [cpp-language-and-modules.md](references/cpp-language-and-modules.md) | C++20 through C++29 language behavior, templates, constexpr, modules, and reflection |
| [libraries-builtins-and-attributes.md](references/libraries-builtins-and-attributes.md) | libstdc++, headers, builtins, intrinsics, attributes, and annotations |
| [diagnostics-and-tooling.md](references/diagnostics-and-tooling.md) | Warning policy, diagnostic formats, clang-format, libclang, bindings, analyzers, and plugins |
| [optimization-debugging-and-sanitizers.md](references/optimization-debugging-and-sanitizers.md) | Optimizer contracts, code generation, LTO, debug information, profiles, and sanitizers |
| [targets-offloading-and-openmp.md](references/targets-offloading-and-openmp.md) | Target ABIs and CPUs, CUDA/HIP, OpenACC, OpenMP, WebAssembly, and platform defaults |

## Upgrade workflow

1. Record the old and new compiler, target, standard library, linker, and
   language mode.
2. Make `-std=` explicit in every compile path, including configure probes,
   generated build rules, host tools, and device compilation.
3. Clean all objects, modules, precompiled headers, LTO state, and generated
   configure results before evaluating failures.
4. Resolve hard errors and removed interfaces before globally suppressing new
   warnings.
5. Rebuild every object that crosses an affected ABI boundary with one
   compatible compiler configuration.
6. Run optimized tests with aliasing, pointer-overflow, undefined-behavior, and
   target-specific coverage where those contracts matter.
7. Verify proposed language or library facilities with feature-test macros,
   `__has_*` probes, and a compile test on the actual target.

## Breaking defaults first

### Pin language modes

GCC 15 changed the default C dialect to GNU C23. In that mode, `f()` declares a
function with no parameters, and `bool`, `true`, `false`, `nullptr`, and
`thread_local` are keywords. Pin an older mode or migrate declarations and
identifiers deliberately.

GCC 16 changed the default C++ dialect to GNU C++20. Always pass the intended
mode explicitly. Autoconf older than 2.73 can instead mis-detect GCC 16 and
inject GNU C++11; regenerate affected configure machinery rather than patching
around apparently missing library features.

```sh
cc -std=gnu17 -c legacy.c
c++ -std=gnu++20 -c app.cc
```

### Expect stricter source diagnostics

Clang 22 makes `-Wincompatible-pointer-types` an error by default in C. Clang 21
makes chained comparisons errors by default. Demote a diagnostic only as a
short migration step:

```sh
clang -Wno-error=incompatible-pointer-types -c legacy.c
clang++ -Wno-error=parentheses -c comparisons.cc
```

Clang 20 also hardened several C++ compatibility diagnostics: out-of-range enum
constant expressions cannot be restored with the removed
`-Wenum-constexpr-conversion`, and extraneous template heads require
`-Wno-error=extraneous-template-head` if they cannot be fixed immediately.

### Do not rely on implicit representation or headers

Automatic unions initialized with `{0}` need not have zeroed padding. Do not
hash, serialize, compare, or expose the full object representation on that
assumption; clear representation storage explicitly or use the controlled GCC
compatibility switch.

Include each libstdc++ name's owning header. In particular, use `<stdint.h>` for
global fixed-width typedefs, `<cstdint>` for `std::` typedefs, and `<ostream>`
for stream declarations and manipulators. Remove compatibility headers such as
`<cstdbool>` and `<cstdalign>`.

## ABI triage

Never mix objects across a documented ABI transition unless every producer and
consumer is compiled with a matching compatibility mode.

| Change | Migration control |
|---|---|
| Clang 20 Microsoft placeholder-return mangling | `-fms-compatibility-version=19.14` for the older form |
| Clang 20 Itanium construction-vtable and friend-template mangling | `-fclang-abi-compat=19` |
| Clang 20 32-bit Arm empty-struct passing | `-fclang-abi-compat=19` |
| Clang 21 large C++ record returns | `-fclang-abi-compat=20` |
| Clang 22 Windows deleting destructors | `-fclang-abi-compat=21` for scalar compatibility; avoid mixed vtables |
| GCC 16 Solaris 8-bit typedef identity | Rebuild all objects or temporarily define `_LEGACY_INT8_T` |
| GCC 16 C++17 `variant` layout correction | `_GLIBCXX_USE_VARIANT_CXX17_OLD_ABI` |

GCC 16 also changed ABI in formerly experimental C++20 library components.
Rebuild objects that exchange atomic-waiting state, semaphores, syncstream or
format state, stop tokens, variants, or affected ranges across a binary
boundary.

## Optimizer and pointer contracts

Clang's alias analysis now distinguishes incompatible pointer types by default.
Fix strict-aliasing violations; use `-fno-pointer-tbaa` only as a bounded
compatibility measure.

Pointer-addition overflow is undefined and can cause checks such as
`ptr + offset < ptr` to fold away. Compare the integer offset before addition
or use checked integer arithmetic. `-fwrapv` covers signed integers,
`-fwrapv-pointer` covers pointers, and `-fno-strict-overflow` implies both.

Request diagnostics where useful:

```sh
clang -fsanitize=pointer-overflow -O2 tests.c
```

GCC 15 enables `-fassume-sane-operators-new-delete` by default. Replacement
global allocation functions that expose observable global state may require
`-fno-assume-sane-operators-new-delete`.

## Modules and standard-library boundaries

Clang 22 enables Reduced BMI mode by default for C++20 modules. Two-phase module
builds must preserve the reduced-BMI workflow, and source must not depend on
implementation details discarded from the BMI.

GCC can build the standard header unit and `std`/`std.compat` modules before
other inputs with `--compile-std-module` when its experimental module support is
enabled. Keep module artifacts compiler-, flag-, target-, and library-specific.

## Sanitizer quick reference

- `-fsanitize=undefined` no longer implies `-fsanitize=vptr` in Clang 21; add
  `vptr` explicitly when required.
- `-fsanitize=realtime` detects unsafe allocation and locking in
  `[[clang::nonblocking]]` execution.
- Experimental `-fsanitize=type` detects type-based aliasing violations.
- `-fsanitize=alloc-token` instruments allocations for allocator-level token
  organization.
- Trapping UBSan can emit `basic` or `detailed` trap reasons in DWARF; disable
  merging or use `-O0` when distinct sites must remain distinguishable.

## Standards and feature checks

Use the standard mode that matches the project contract, then probe the actual
facility:

```c
#if defined(__has_feature)
#  if __has_feature(c_countof)
     /* _Countof is available */
#  endif
#endif
```

Do not infer implementation from a status-table `Unknown` entry. Clang's C23,
C2y, C++23, C++26, and C++29 modes each retain documented gaps, and C++20
coroutines remain problematic on Windows targets. Consult the language
references before depending on a recently adopted proposal.

## Tooling hygiene

Machine-readable GCC diagnostics should use SARIF; the old JSON format was
deprecated in GCC 15 and removed in GCC 16. Consumers that parse display text
should also account for GCC 16's nested C++ diagnostics or request flat output.

After a Clang upgrade, update checker names, clang-format keys, libclang link
dependencies, and binding null/error handling together. Do not assume
`clangFrontend` or another library still supplies driver symbols transitively.

## Target and offloading checks

Target feature probes must run for the currently active compilation target.
During Clang offloading, `__has_builtin` now reports only the active target.
Compile host and device probes separately.

Treat target ABI changes, default CPU changes, and offload-driver changes as
clean-rebuild events. CUDA's newer Clang RDC format is not interchangeable with
NVIDIA's, AMDGPU code object defaults impose runtime requirements, and
WebAssembly target names and baseline features have changed.

## Final validation

- Compile all supported language modes with explicit flags.
- Perform a clean, whole-program rebuild for any ABI-sensitive transition.
- Test optimized and sanitized configurations separately.
- Re-run generated configuration with the current compiler.
- Validate tooling output formats and analyzer checker names.
- Compile feature probes on every supported host and device target.
