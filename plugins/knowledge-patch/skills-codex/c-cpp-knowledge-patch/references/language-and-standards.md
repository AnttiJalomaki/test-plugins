# Language and Standards

Language-mode acceptance is not a conformance guarantee. Probe the exact core
or library facility, retain compiler/target guards where necessary, and account
for defect-report behavior that can change without a new `-std=` value.

## C language facilities

### C23

GCC 15 completes several important pieces (`gcc-15.1`):

- `#embed` is implemented.
- The `unsequenced` and `reproducible` attributes are available.
- `__STDC_VERSION__` is `202311L` in `-std=c23` and `-std=gnu23`.

Clang 20 supports Improved Normal Enumerations (N3029), exposes `__nullptr` as
an alias for `nullptr` in every C mode, and rejects `register void` parameters
(`clang-20.1`). Clang 21 accepts `-std=iso9899:2024` as a C23 alias and clang-cl
accepts `/std:clatest`. It permits structurally equivalent tag definitions in
one translation unit; structure or union rvalues containing arrays obey C11
temporary lifetime through the full expression, including in older modes
(`clang-21.1`).

Clang 22 adds `FLT_SNAN`, `DBL_SNAN`, and `LDBL_SNAN` to `<float.h>` in C23 and
later. With dependency generation, `-MG` suppresses missing-file errors from
`#embed`. Constants of a fixed-underlying enumeration now have the enumeration
type, and distinct unnamed tags with identical fields are no longer compatible
(`clang-22.1`).

### C2y and GNU C extensions

GCC 15's explicit `-std=c2y` and `-std=gnu2y` modes support (`gcc-15.1`):

- type operands in generic selections;
- complex increment/decrement and literals;
- byte-array access and `alignof` on incomplete arrays;
- delimited escapes and case-range expressions;
- named loops and declarations in `if`;
- rotate and defined absolute-value builtins;
- zero-length operations on null pointers.

Clang 20 stops GNU-extension warnings in C2y for imaginary suffixes and case
ranges, and stops `-Wgnu-empty-struct` for empty structures/unions
(`clang-20.1`). Clang 21 adds `0o`/`0O` octal syntax
and deprecates nonzero leading-zero octal literals. It accepts `\x{12}` and
`\o{12}` as extensions in older modes. `_Countof(array)` is also an older-mode
extension, with `countof` in `<stdcountof.h>` and feature tests
`__has_feature(c_countof)` and `__has_extension(c_countof)` (`clang-21.1`).

Clang 22 adds named loops in C2y and offers the draft C `defer` TS behind
`-fdefer-ts`. Static functions and variables inside `extern inline` no longer
trigger `-Wstatic-in-inline` in C2y. `__COUNTER__` warns as an extension in other
modes and errors after 2,147,483,647 expansions (`clang-22.1`).

GCC 16 supports static assertions in expressions and unspecified or variably
modified array types in generic associations; it diagnoses some formerly
undefined cases as constraint violations (`gcc-16.1`). GNU C also gains:

- `_Maxof(type)` and `_Minof(type)` integer limits;
- empty initialization of variable-size compound literals, such as
  `(int[n]) {}`;
- a guarantee that noncapturing nested functions do not require runtime
  trampolines;
- `counted_by` on pointer members:

```c
struct buffer {
  unsigned count;
  int *data __attribute__((counted_by(count)));
};
```

## C++ core language

### C++20 and C++23

Clang 20 performs module-level lookup for C++20 modules (`clang-20.1`). Its C++23
mode fully implements range-for temporary lifetime extension, allows unknown
pointers and references in constant expressions, removes the literal-type
restriction from constexpr functions, and defines
`__cpp_explicit_this_parameter`.

Defect-report behavior in Clang 20 also changes these cases:

- For a `T` prvalue `e`, `T{e}` prefers a viable initializer-list constructor
  before guaranteed copy elision.
- Suitably narrow bit-fields are non-narrowing.
- `nullptr` promotes to `void *` through C-style varargs.
- `void{}` is accepted.
- Explicit deduction guides may have trailing requires-clauses, and constructor
  constraints propagate into CTAD.

Clang 22 makes Reduced BMI the default for C++20 modules (`clang-22.1`).
Two-phase module builds must support reduced BMIs, and implementation details
discarded from them are unavailable. Clang 20's stable opt-in spelling was
`-fmodules-reduced-bmi`.

GCC 16 can prebuild `<bits/stdc++.h>`, `std`, and `std.compat` with
`--compile-std-module` before explicit inputs; once the header unit exists,
eligible standard-header includes can become imports (`gcc-16.1`). This remains
an experimental modules workflow.

### C++26 language work

Clang 20 implements variadic friends (P2893R3), constexpr placement new
(P2747R2), and the Oxford variadic comma (P3176R1). User-defined `static_assert`
messages are accepted as an extension back to C++11; implementation builtins
include `__builtin_is_virtual_base_of` and `__builtin_is_within_lifetime`
(`clang-20.1`).

GCC 15 adds pack indexing, attributes on structured bindings, reason strings in
`= delete("reason")`, and structured bindings as conditions (`gcc-15.1`). It also
implements standard-attribute ignorability, rejects macro-produced module
declarations, makes deletion through an incomplete type ill-formed, removes
deprecated array comparisons, and deprecates the notion of trivial types. The
basic character set includes `@`, `$`, and backtick.

Clang 21 adds structured-binding packs, trivial relocatability,
structured-binding conditions, and attachment of `main` to the global module
(`clang-21.1`). `__builtin_structured_binding_size(T)` reports a destructuring
binding count. A perfect identity-conversion match from a non-template overload
can suppress template-candidate instantiation, so diagnostics produced only by
an unused template candidate can disappear.

Clang 22 supports constexpr structured bindings for arrays and aggregates, not
references or tuple-like decomposition (`clang-22.1`). It normalizes constraints
before satisfaction checks, rejects sibling-member pointer access during
constant evaluation, checks constant template parameter types in template
definitions, and disallows immediate escalation in destructors.

GCC 16 adds expansion statements, contracts, erroneous behavior for
uninitialized reads, constexpr exceptions, constexpr virtual inheritance,
partial program correctness, and defined preprocessing behavior (`gcc-16.1`).
P2996R13 reflection is separate from the dialect switch:

```sh
g++ -std=c++26 -freflection source.cc
```

The reflection work covers annotations, parameter reflection, base-class
subobject splicing, reflection errors, `define_static_string`,
`define_static_object`, and `define_static_array`.

### Templates, allocation, and relocation

GCC 15 diagnoses invalid current-instantiation lookup when parsing a template,
not only at instantiation (`gcc-15.1`). It permits constexpr-generated C++
inline-assembly strings. Clang 21 likewise permits GNU `asm` strings as constant
expressions and implements P2719R5 type-aware allocation/deallocation as an
extension in every C++ mode (`clang-21.1`).

Clang 21 introduces `__builtin_is_cpp_trivially_relocatable`,
`__builtin_is_replaceable`, and `__builtin_trivially_relocate`, while deprecating
`__is_trivially_relocatable`. A relocatable, non-trivially-copyable object must
be moved with `__builtin_trivially_relocate`, not copied with `memcpy`.

After P2786 left C++26, Clang 22 removed `__builtin_is_replaceable`,
`trivially_relocable_if_eligible`, and `replaceable_if_eligible`
(`clang-22.1`). `__builtin_is_cpp_trivially_relocatable` and
`__builtin_trivially_relocate` remain with P2786-style semantics, but source can
no longer explicitly mark a type relocatable.

Clang 22 adds relational builtins
`__builtin_{lt,gt,le,ge}_synthesizes_from_spaceship` to report whether an
operator is synthesized from `<=>`. Inside template arguments
or base specifiers, `__builtin_dedup_pack<Ts...>...` produces an unexpanded pack
with duplicate types removed.

## Current conformance cautions

The compiler status data is `standards-status`; treat `Unknown` as unknown, not
implemented. C11, C17, C23, C2y, C++20, C++23, C++2c, and C++2d tables all
contain partial support.

### C23 gaps

Clang accepts `-std=c23` from Clang 18, but unsequenced functions (N2956) and
storage-class specifiers on compound literals (N3038) remain unsupported.
Pointer-to-array qualifier compatibility is partial: C17 and older modes miss
some pedantic diagnostics, and `?:` can derive an incorrect qualified-array
result. Improved tag compatibility is partial from Clang 21 because attributes
and extensions can make equivalent definitions incorrectly accepted or
rejected; expect further source changes near these cases.

### C++20 and C++23 gaps

C++20 coroutines are fully supported except on Windows, where ABI and stability
issues remain. On 32-bit x86 Windows, `__cpp_impl_coroutine` is absent and using
coroutines warns. Module proposal P1857R3 arrives only in Clang 23 and P1815R2
remains partial. Scalar non-type template parameters still have limitations for
references to instantiation-dependent objects or subobjects. Alias-template
CTAD exists, but `__cpp_deduction_guides` does not advertise it.

The C++23 table still lacks constexpr `<cmath>`/`<cstdlib>`, standard extended
floating-point types, CTAD from inherited constructors, and explicit lifetime
management. Unicode identifiers are accepted without NFC normalization checks.

### Draft-language gaps

Use `-std=c2y` from Clang 19. `if` declarations need Clang 24, while static
assertions in expressions, bit-precise enums, multidimensional-array generic
matching, and array subscripting without decay remain unsupported.

Use `-std=c++2c` for C++26. Clang 23 rejects macro module declarations and only
partially supports expansion statements because iterating expansions are
diagnosed. Clang 24 adds constexpr virtual inheritance, but contracts,
reflection, `#embed`, constexpr exceptions, trivial unions, and other adopted
facilities remain unsupported. The table advances unevenly through Clang 24.

Clang accepts `-std=c++2d`, but C++29 support is preliminary and most listed
proposals are absent. Clang 23 adds more named universal-character escapes; do
not treat mode acceptance as broad C++29 availability.
