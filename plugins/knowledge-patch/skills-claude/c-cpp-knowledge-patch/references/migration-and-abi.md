# Migration and ABI

Use this reference for clean-build planning, source migrations, compatibility
switches, and binary-boundary decisions.

## Pin changed default dialects

### GCC C defaults (gcc-15.1-porting)

GCC 15 changed the default C mode from `-std=gnu17` to `-std=gnu23`. In the new
mode, `f()` means that `f` takes no arguments; declare the actual parameter
types instead of relying on the old unspecified-parameter form. `bool`, `true`,
`false`, `nullptr`, and `thread_local` are keywords, so rename conflicting
identifiers. Do not treat public C `bool` as ABI-identical to `int`.

### GCC C++ defaults (gcc-16.1-porting)

GCC 16 changed the default from `-std=gnu++17` to `-std=gnu++20`. Pass the
intended dialect explicitly in every build and probe.

Autoconf before 2.73 can mis-detect GCC 16 in `AC_PROG_CXX` and inject
`-std=gnu++11`, making default-mode facilities such as `std::make_unique`
appear unavailable. Regenerate with a corrected Autoconf setup rather than
adding feature-by-feature workarounds.

## Resolve new source incompatibilities

### C pointer and statement-expression errors (clang-22.1)

`-Wincompatible-pointer-types` is an error by default; use
`-Wno-error=incompatible-pointer-types` only while correcting the types. A
trailing null statement makes a GNU statement expression `void`, so
`({ 1;; })` no longer has type `int`.

### Hardened C++ compatibility diagnostics (clang-20.1)

`__is_nullptr` was removed; use
`__is_same(__remove_cv(T), decltype(nullptr))`. `__is_referenceable` is
deprecated for removal in Clang 21. Out-of-range enum values in constant
expressions cannot be restored with the removed
`-Wenum-constexpr-conversion`. Extraneous template headers are errors unless
temporarily demoted with `-Wno-error=extraneous-template-head`.

### Iterator adaptors must advertise real capabilities (gcc-15.1-porting)

The `std::vector` range constructor recognizes C++20 iterator concepts and can
select a stronger optimized path. An adaptor that exposes invalid operations
unconditionally can now fail during instantiation. Constrain each operation to
the wrapped iterator's capability, using SFINAE in older modes:

```cpp
iterator_adaptor& operator--()
  requires std::bidirectional_iterator<Iter>
{
  --iter;
  return *this;
}
```

### Union initialization does not clear padding (gcc-15.1-porting)

For an automatic C or C++ union, `{0}` initializes the first member but need
not zero every padding byte. Do not hash, serialize, compare, or expose the
whole representation on that assumption. Use `{}` where supported, clear the
storage explicitly, or deliberately opt into
`-fzero-init-padding-bits=unions`.

### Removed Concepts TS behavior (gcc-15.1)

Concepts TS support and the behavior selected by `-fconcepts-ts` are gone.
Migrate to standard concepts.

### Removed compatibility paths (clang-21.1)

The Objective-C ARC migrator is gone. The libstdc++ 4.7 workaround was removed,
making 4.8.3 the oldest supported version. The deprecated
`-frelaxed-template-template-args` spellings were removed.

## Rebuild across ABI transitions

### C++ mangling (clang-20.1)

Microsoft mangling for placeholder, `auto`, and `decltype(auto)` return types
now matches MSVC 1920+. Use `-fms-compatibility-version=19.14` when old object
compatibility is required. Itanium construction-vtable names and member-like
friend function-template mangling also changed; `-fclang-abi-compat=19`
selects the old forms.

### 32-bit Arm empty structures (clang-20.1)

Empty C++ structures are passed as one-byte objects to match AAPCS32 and GCC.
`-fclang-abi-compat=19` restores the old ignored-argument behavior. SME
function-type attributes now participate in mangling. Separately,
`-fno-omit-frame-pointer` retains frame pointers in leaf functions unless
paired with `-momit-leaf-frame-pointer`.

### C++ record returns (clang-21.1)

Larger C++ records are returned in memory rather than AVX registers. Objects
built by older Clang releases are incompatible unless new compilation uses
`-fclang-abi-compat=20`.

### Windows deleting destructors (clang-22.1)

Under the MSVC ABI, `::delete` now invokes the scalar deleting destructor.
Mixing Clang 21-or-earlier and Clang 22 objects can select the wrong deallocator
and corrupt memory; `-fclang-abi-compat=21` retains the older scalar behavior.
Windows vtables now use the differently named and linked MSVC vector deleting
destructor, which is another mixed-version incompatibility for classes with
virtual destructors.

### Solaris 8-bit typedef identity (gcc-16.1-porting)

On Solaris, `int8_t`, `int_fast8_t`, and `int_least8_t` now use `signed char`
instead of plain `char`. Those types mangle differently in C++, so rebuild all
objects across the boundary. `_LEGACY_INT8_T` is temporary compatibility when a
complete rebuild is impossible.

### C++17 `std::variant` layout (gcc-16.1)

A layout correction affects the narrow case where an empty base and first
member interact in C++17 mode. `_GLIBCXX_USE_VARIANT_CXX17_OLD_ABI` restores
the old layout for migration.

### Formerly experimental C++20 library components (gcc-16.1)

The C++20 library is no longer experimental. ABI changed in atomic waiting,
semaphores, syncstream, format-argument representation, partial ordering, some
stop-token/variant combinations, and some range adaptors. Rebuild objects that
exchange affected types or state across binary boundaries.

## Platform migration details

### Solaris pthread flags (gcc-16.1-porting)

On Solaris, `-pthread` and `-pthreads` are ignored and no longer define
`_REENTRANT` or `_PTHREADS`. If application code used those macros for feature
selection, define an application-specific macro explicitly.

### GCC discovery with incomplete libstdc++ (clang-22.1)

Clang warns with `-Wgcc-install-dir-libstdcxx` when automatic discovery chooses
the highest-version GCC installation, that installation lacks libstdc++
headers, and another complete installation exists. Install or remove headers
consistently, select the intended tree with `--gcc-install-dir`, or suppress
the warning with `-Wno-gcc-install-dir-libstdcxx`.

## Clean-rebuild rule

Compatibility switches are migration tools, not a durable mixed-ABI design.
Clean module caches, PCH files, LTO state, generated configuration, and every
object that owns or exchanges affected layouts, names, calling conventions, or
library state.
