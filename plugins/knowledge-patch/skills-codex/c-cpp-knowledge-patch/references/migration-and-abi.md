# Migration and ABI

Use this reference before changing compiler major versions or linking objects
built by different toolchains. Pin the source dialect first, then separate
source, optimizer, library, and binary-interface failures.

## Changed language defaults

### GCC C default

GCC 15 defaults C compilation to `-std=gnu23` instead of `-std=gnu17`
(`gcc-15.1-porting`). Pin an older mode or migrate affected source. In C23:

- `f()` declares a function taking no arguments, not unspecified parameters.
- `bool`, `true`, `false`, `nullptr`, and `thread_local` are keywords; rename
  legacy identifiers.
- Public C `bool` must not be assumed ABI-identical to `int`.

Declare the real callback signature, for example:

```c
void (*handler)(int);
```

### GCC C++ default

GCC 16 defaults C++ to `-std=gnu++20` instead of `-std=gnu++17`
(`gcc-16.1-porting`). Set `-std=` explicitly in builds and probes. Autoconf
before 2.73 can mis-detect GCC 16 in `AC_PROG_CXX` and inject
`-std=gnu++11`, making facilities such as `std::make_unique` appear absent;
regenerate with a corrected setup instead of working around individual missing
facilities.

## Optimizer-sensitive source

### Pointer aliasing and overflow

Clang emits distinct type-based alias-analysis tags for incompatible pointer
types by default (`clang-20.1`). Fix strict-aliasing violations;
`-fno-pointer-tbaa` restores the prior behavior only as a migration aid.

Pointer addition that overflows is treated more aggressively as undefined, so
`ptr + offset < ptr` can fold to false. Compare or validate the offset before
addition, use suitable integer arithmetic, or diagnose with
`-fsanitize=pointer-overflow`. `-fwrapv` covers signed integers only,
`-fwrapv-pointer` covers pointers, and `-fno-strict-overflow` implies both.

Clang 21 extends the optimization around null-pointer arithmetic
(`clang-21.1`). Old-style `offsetof` idioms remain preserved, while
`-fwrapv-pointer` or `-fno-delete-null-pointer-checks` defines the arithmetic
more generally. With branch-target enforcement, an `asm goto` label is no
longer guaranteed to begin with `bti` or `endbr64`; do not use such a label as
the destination of a register-controlled branch.

### Constant-expression pointer rules

In Clang 20, comparing separate evaluations of the same string literal is not a
constant expression; literals that cannot overlap may compare false. Forming a
field address through null, such as `&((S *)nullptr)->member`, is also rejected
during constant evaluation (`clang-20.1`).

### Union representation

For an automatic C or C++ union, `{0}` initializes the first member but does not
guarantee zeroed padding (`gcc-15.1-porting`). Do not hash, serialize, compare,
or expose the full representation on that assumption. Use `{}` where supported,
clear representation storage explicitly, or deliberately use
`-fzero-init-padding-bits=unions` as a compatibility measure.

## Source compatibility changes

### Clang C and C++ diagnostics

Clang 20 removed `__is_nullptr`; use
`__is_same(__remove_cv(T), decltype(nullptr))`. `__is_referenceable` is
deprecated for removal in Clang 21. Out-of-range enum constants cannot be
accepted by disabling `-Wenum-constexpr-conversion`, because the control was
removed. Extraneous template headers are errors, but can be demoted with
`-Wno-error=extraneous-template-head` (`clang-20.1`).

Clang 21 makes chained comparisons such as `a < b < c`, including folds over
comparison operators, errors by default; `-Wno-error=parentheses` demotes them
(`clang-21.1`). Attributes before an `extern template` declaration are rejected.

Clang 22 makes `-Wincompatible-pointer-types` an error by default; use
`-Wno-error=incompatible-pointer-types` only for a staged migration. A trailing
null statement makes a GNU statement expression `void`, so `({ 1;; })` is no
longer an `int` expression (`clang-22.1`).

### Lifetime annotations

Clang 20 rejects `[[clang::lifetimebound]]` on types, unnamed parameters,
explicit-object members, and the parameters or implicit object of void-returning
functions instead of ignoring it. The compiler also infers the annotation for
`std::span` and `std::string_view` constructor parameters, which can expose new
dangling-reference diagnostics (`clang-20.1`).

### Headers and iterator adaptors

GCC 15's libstdc++ exposes fewer transitive headers (`gcc-15.1-porting`). Include
`<stdint.h>` for global fixed-width types, `<cstdint>` for their `std::` forms,
and `<ostream>` for stream declarations, `std::endl`, and `std::flush`. Remove
`<cstdbool>` and `<cstdalign>`; replace `<ccomplex>` with `<complex>`, and
`<ctgmath>` with `<cmath>`, `<complex>`, or both.

The `std::vector` range constructor recognizes C++20 iterator concepts and may
select a stronger optimized path. An adaptor must constrain operations to the
wrapped iterator's actual capabilities (or use equivalent SFINAE in older
modes):

```cpp
iterator_adaptor& operator--()
  requires std::bidirectional_iterator<Iter>
{
  --iter;
  return *this;
}
```

### Unused-but-set warnings

GCC 16 makes `-Wunused-but-set-variable` and
`-Wunused-but-set-parameter` sensitivity level 3 the default, including through
`-Wall` and `-Wextra` (`gcc-16.1-porting`). Level 2 stops treating increment and
decrement as uses; level 3 also stops treating compound assignment as a use when
the old value is absent from the right-hand side. Level 1 approximates the old
behavior:

```text
-Wunused-but-set-variable=1 -Wunused-but-set-parameter=1
```

## ABI transition matrix

| Boundary | Impact | Migration control |
| --- | --- | --- |
| Clang 20 Microsoft placeholder, `auto`, and `decltype(auto)` returns | Mangling now matches MSVC 1920+ | `-fms-compatibility-version=19.14` for older Clang objects |
| Clang 20 Itanium construction vtables and member-like friend templates | Mangled names changed | `-fclang-abi-compat=19` |
| Clang 20 32-bit Arm empty C++ structs | Passed as one-byte objects to match AAPCS32/GCC | `-fclang-abi-compat=19` restores ignored arguments |
| Clang 21 larger C++ record returns | Returned in memory instead of AVX registers | Compile new objects with `-fclang-abi-compat=20` when old objects cannot yet be rebuilt |
| Clang 22 MSVC scalar deleting destructor | `::delete` invokes the scalar deleting destructor; a mixed build can select the wrong deallocator | `-fclang-abi-compat=21` retains old scalar behavior |
| Clang 22 Windows vector deleting destructor | Vtables use the differently named and linked MSVC form; virtual-destructor classes are incompatible across the boundary | Rebuild the complete boundary |
| Clang 22 AArch64 aligned empty classes | Argument passing changed for empty C++ classes with large explicit alignment | Rebuild callers and callees together |
| GCC 16 Solaris 8-bit typedefs | `int8_t`, `int_fast8_t`, and `int_least8_t` use `signed char`, which mangles differently from `char` | Rebuild all boundary objects; `_LEGACY_INT8_T` is temporary |
| GCC 16 C++17 `std::variant` corner case | Layout changed when an empty base and first member interact | `_GLIBCXX_USE_VARIANT_CXX17_OLD_ABI` temporarily restores it |
| GCC 16 formerly experimental C++20 library components | ABI changes affect atomic waiting, semaphores, syncstream, format arguments, partial ordering, some stop-token/variant combinations, and range adaptors | Rebuild objects exchanging affected types or state |

The Clang entries are from `clang-20.1`, `clang-21.1`, and `clang-22.1`; the GCC
entries are from `gcc-16.1-porting` and `gcc-16.1`.

## Removed and deprecated compatibility paths

- Clang 20 removed `le32`, `le64`, `clang-rename`, and RenderScript target
  support. SPARC Linux `clang -m32` now defaults to `-mcpu=v9`; distributions needing
  V8 must add `-mcpu=v8` (`clang-20.1`).
- Clang 20 `*mmintrin.h` MMX intrinsics using `__m64` require SSE2/XMM and no longer
  work on MMX-only or `-mmmx -mno-sse2` targets. MMX assembly remains; direct
  users of removed `__builtin_ia32_*` implementations must use the header
  intrinsics (`clang-20.1`).
- GCC 15 removed Nios II and Solaris 11.3, deprecated AArch64 ILP32
  (`-mabi=ilp32`), and is the last GCC with the old `reload` register allocator;
  targets without LRA are affected in GCC 16 (`gcc-15.1`).
- Clang 21 removed the Objective-C ARC migrator and its libstdc++ 4.7 workaround,
  making 4.8.3 the oldest supported libstdc++. It also removed
  `-frelaxed-template-template-args` and its negative spelling (`clang-21.1`).
- GCC 15 removed Concepts TS behavior and `-fconcepts-ts` (`gcc-15.1`).
- Clang 22 warns with `-Wgcc-install-dir-libstdcxx` when automatic GCC discovery selects the highest-version tree
  without libstdc++ headers while another complete tree exists. Fix the install,
  select it with `--gcc-install-dir`, or suppress it with
  `-Wno-gcc-install-dir-libstdcxx` (`clang-22.1`).

## Other platform migration notes

On Solaris, GCC 16 ignores `-pthread` and `-pthreads`; they no longer define
`_REENTRANT` or `_PTHREADS`. Define an application-specific feature macro rather
than deriving application behavior from those flags (`gcc-16.1-porting`).

Clang 22's warning-suppression mappings choose the last overlapping match rather
than the longest one. Reorder existing files deliberately (`clang-22.1`).

GCC 16 removed `json` from `-fdiagnostics-format=` after its GCC 15 deprecation;
machine consumers must use SARIF (`gcc-15.1`, `gcc-16.1`).
