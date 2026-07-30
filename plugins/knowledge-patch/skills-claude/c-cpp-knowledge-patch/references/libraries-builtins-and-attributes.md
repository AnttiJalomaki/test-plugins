# Libraries, Builtins, and Attributes

Prefer standard facilities when they meet the requirement. Guard compiler
builtins and attributes with the appropriate `__has_builtin`,
`__has_attribute`, or language feature probe.

## Header ownership and libstdc++ migration

### Direct includes are required (gcc-15.1-porting)

Do not rely on libstdc++ transitive includes. Use `<stdint.h>` for global
fixed-width integer typedefs, `<cstdint>` for their `std::` forms, and
`<ostream>` for stream declarations, `std::endl`, and `std::flush`.

Compatibility headers now warn: remove `<cstdbool>` and `<cstdalign>`, replace
`<ccomplex>` with `<complex>`, and replace `<ctgmath>` with `<cmath>`,
`<complex>`, or both.

## libstdc++ behavior and facilities

### Assertions and allocation assumptions (gcc-15.1)

Unoptimized builds enable libstdc++ debug assertions by default. Define
`_GLIBCXX_NO_ASSERTIONS` to disable them deliberately.

GCC also enables `-fassume-sane-operators-new-delete` by default. Programs with
replacement global allocation functions whose observable global state matters
may need `-fno-assume-sane-operators-new-delete`.

### C++23 library additions (gcc-15.1)

libstdc++ adds the `std` and `std.compat` modules, flat associative containers,
range constructors and modifiers, and range and tuple formatting.

### Experimental C++26 library additions (gcc-15.1)

Additions include `views::concat`, `views::to_input`, `views::cache_latest`,
constexpr sorting and raw-memory algorithms, `<stdbit.h>`, `<stdckdint.h>`,
`std::is_virtual_base_of`, member `visit`, and type checking for `std::format`
arguments.

### Random-number sequence compatibility (gcc-16.1)

`std::generate_canonical` adopts P0952R2 behavior, changing result sequences.
Define `_GLIBCXX_USE_OLD_GENERATE_CANONICAL` when reproducing the old sequence
is temporarily required.

### C++23 additions (gcc-16.1)

libstdc++ adds `std::mdspan`, starts/ends-with range algorithms, shift
algorithms, and `allocate_at_least`.

### C++26 additions (gcc-16.1)

New support includes `std::simd`, `std::inplace_vector`,
`std::optional<T&>`, `std::copyable_function`, `std::function_ref`,
`std::indirect`, `std::polymorphic`, `std::owner_equal`, `<debugging>`,
string-view overloads, padded `mdspan` layouts, `std::philox_engine`, and
`std::atomic_ref::address()`.

## Clang elementwise and type builtins

### Numeric and type operations (clang-20.1)

New builtins are:

- `__builtin_elementwise_popcount`;
- `__builtin_elementwise_fmod`;
- `__builtin_elementwise_minimum` and
  `__builtin_elementwise_maximum`;
- `__builtin_elementwise_atan2`; and
- `__builtin_common_type`.

Integer reduction and elementwise bit, count, and saturating builtins; floating
comparison builtins; `__builtin_signbit`; and `__builtin_abs` can be used in
constant expressions.

### Invocation, vtables, and cache flushing (clang-21.1)

Clang adds `__builtin_elementwise_exp10`,
`__builtin_elementwise_minnum`, `__builtin_elementwise_maxnum`,
`__builtin_invoke`, and `__builtin_get_vtable_pointer`.
`__builtin___clear_cache` now has signature `void(void *, void *)`, matching GCC
instead of accepting `char *`.

### Template and comparison queries (clang-22.1)

`__builtin_{lt,gt,le,ge}_synthesizes_from_spaceship` reports whether a
relational operator is synthesized from `<=>`. Within template arguments or
base specifiers, `__builtin_dedup_pack<Ts...>...` creates an unexpanded pack
with duplicate types removed.

### Vector and low-level operations (clang-22.1)

New operations include:

- `__builtin_elementwise_ldexp`;
- `__builtin_elementwise_fshl` and `__builtin_elementwise_fshr`;
- `__builtin_elementwise_minnumnum` and
  `__builtin_elementwise_maxnumnum`;
- generic `__builtin_bswapg`;
- `__builtin_stack_address()`; and
- masked load, store, gather, and scatter builtins.

Integer elementwise min, max, and abs support constant evaluation. Fixed
boolean vectors work with generic popcount/count-zero builtins and as `?:`
conditions. `__builtin_assume_dereferenceable` accepts run-time sizes.

## Intrinsic compatibility

The `*mmintrin.h` intrinsics on `__m64` use SSE2 and XMM registers in
clang-20.1. They do not work for MMX-only targets or with
`-mmmx -mno-sse2`; MMX inline assembly remains supported. The former
`__builtin_ia32_*` implementation builtins were removed, so direct callers must
use the header intrinsics.

## Function and statement contracts

### Tail calls and size-dependent nullability (gcc-15.1)

A `musttail` statement attribute can require a tail call.
`nonnull_if_nonzero` expresses that a pointer parameter must be non-null when a
separate size or count parameter is nonzero.

### Attribute availability and diagnostics (gcc-15.1)

C++11 attributes are accepted in C++98 mode. `flag_enum` can suppress
inappropriate switch warnings. `-Wdefaulted-function-deleted` diagnoses
explicitly defaulted functions that become deleted.

## Lifetime and specialization annotations

`[[clang::lifetimebound]]` is rejected on types, unnamed parameters,
explicit-object member functions, and parameters or implicit objects of
void-returning functions instead of being ignored. Clang also infers it for
`std::span` and `std::string_view` constructor parameters, enabling more
dangling-reference diagnostics (clang-20.1).

Clang 20 adds:

- `[[clang::no_specializations]]`;
- `[[clang::lifetime_capture_by(X)]]`;
- `[[clang::coro_await_elidable]]`;
- `[[clang::coro_await_elidable_argument]]`; and
- `__attribute__((format(syslog, ...)))`.

`swift_attr` can annotate types. Attributes after a namespace name are no
longer accepted.

## Format and target-specific annotations

`__attribute__((format_matches(printf, 1, "%x %s")))` declares that a format
parameter is equivalent to a reference format. This validates wrappers without
the corresponding arguments and suppresses inappropriate
`-Wformat-nonliteral` inside them (clang-21.1).

X86-64 globals can select `__attribute__((model("small")))` or
`model("large")` independently of the translation unit. AMDGPU statements can
use `[[clang::atomic(...)]]` to control atomic metadata. Attributes before an
`extern template` declaration are rejected (clang-21.1).

## Allocation, formatting, layout, and CFI annotations

Clang 22 adds several contracts (clang-22.1):

- `malloc_span` applies malloc-like semantics to a function returning a
  pointer-and-size or pointer-pair structure;
- `modular_format` lets a cooperating static libc select required `printf`
  features at link time;
- `[[gnu::gcc_struct]]` requests Itanium record layout on Itanium-ABI targets
  even when Microsoft bit-field layout is active; and
- `[[clang::cfi_unchecked_callee]]` propagates from declaration to definition
  and suppresses `-fsanitize=function` on affected indirect calls.

Keep annotations on public declarations consistent across translation units,
and include them in ABI and static-analysis review.
