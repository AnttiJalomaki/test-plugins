# C Language and Standards

Use explicit language modes and feature probes. Draft modes expose selected
facilities; they do not imply complete conformance.

## C23 mode and compatibility

### Mode selection and version macros

GCC 15 implements `#embed`, the `unsequenced` and `reproducible` attributes,
and reports `__STDC_VERSION__` as `202311L` in `-std=c23` and `-std=gnu23`
(gcc-15.1).

Clang accepts `-std=iso9899:2024` as a C23 alias, and clang-cl accepts
`/std:clatest` (clang-21.1).

### Enumerations, tags, and temporary lifetime

Clang implements Improved Normal Enumerations (N3029) and makes `__nullptr`
available as an alias for `nullptr` in every C mode; it also rejects
`register void` parameters (clang-20.1).

Clang 21 permits structurally equivalent tag definitions in one translation
unit in C23. Structure or union rvalues containing array members follow C11
temporary-lifetime rules through the containing full expression, including in
older modes (clang-21.1).

Clang 22 treats enumeration constants of fixed-underlying enums as having the
enumerated type. Distinct unnamed tag types with identical fields are no longer
accepted as compatible (clang-22.1).

### Headers and dependency scanning

`<float.h>` defines `FLT_SNAN`, `DBL_SNAN`, and `LDBL_SNAN` in C23 and later.
During dependency scanning, `-MG` suppresses missing-file errors from `#embed`
(clang-22.1).

## C2y and draft C facilities

### GCC feature set (gcc-15.1)

`-std=c2y` and `-std=gnu2y` add:

- type operands in generic selections;
- complex increment/decrement and literals;
- byte-array access and `alignof` on incomplete array types;
- delimited escapes and named loops;
- rotate and non-undefined absolute-value builtins;
- case-range expressions and declarations in `if`; and
- zero-length operations on null pointers.

### Clang syntax and counting (clang-21.1)

`0o` and `0O` introduce octal literals. Older nonzero leading-zero octal
literals are deprecated. Delimited escapes such as `\x{12}` and `\o{12}` are
also extensions in older modes.

`_Countof(array)` is an extension in older C modes. `<stdcountof.h>` provides
`countof`; probe with `__has_feature(c_countof)` or
`__has_extension(c_countof)`.

### Clang warning changes (clang-20.1)

In C2y mode, imaginary suffixes and case ranges no longer trigger their GNU
extension warnings, and empty structures or unions no longer trigger
`-Wgnu-empty-struct`.

### Draft `defer`, named loops, and `__COUNTER__` (clang-22.1)

`-fdefer-ts` enables the draft C `defer` Technical Specification. C2y mode adds
named loops and permits static functions or variables inside `extern inline`
functions without `-Wstatic-in-inline`. `__COUNTER__` warns as an extension in
other modes and errors after 2,147,483,647 expansions.

### GCC expression and generic-selection support (gcc-16.1)

C2Y mode supports static assertions in expressions and permits unspecified and
variably modified array types in generic associations. It also diagnoses as
constraint violations cases that older standards classified only as undefined
behavior.

## GNU C facilities

### Counted pointer members (gcc-16.1)

`counted_by` can associate a pointer member with the member holding its element
count:

```c
struct buffer {
  unsigned count;
  int *data __attribute__((counted_by(count)));
};
```

For flexible array members, Clang's
`__builtin_counted_by_ref(flexible_member)` returns access to the named counter,
so allocation macros can initialize it before sanitizer-checked access
(clang-20.1):

```c
*__builtin_counted_by_ref(p->items) = count;
```

### Integer limits and variable-size literals (gcc-16.1)

GNU C provides `_Maxof(type)` and `_Minof(type)` for integer limits:

```c
int highest = _Maxof(int);
int lowest = _Minof(int);
```

An empty initializer is accepted for a variable-size compound literal; for
example, `(int[n]) {}` creates a run-time-sized, zero-initialized temporary.

### Nested functions without captures (gcc-16.1)

A nested function that does not capture its environment is guaranteed not to
need a run-time trampoline. This is useful where executable-stack-style
trampoline support is forbidden.

## C compatibility diagnostics

Clang 21 expands C and C++ compatibility checking (clang-21.1):

- `-Wextra` includes `-Wunterminated-string-initialization`;
- `nonstring` marks intentionally non-terminated C arrays, although it cannot
  suppress the C++-compatibility variant;
- `-Wc++-compat` checks implicit `void *` and integer-to-enum conversions, C++
  keywords and hidden tags, tentative definitions, default initialization of
  `const` objects, and jumps bypassing initialization; and
- `-Wundef-true` is enabled by default before C23.

Clang 22 makes `-Wincompatible-pointer-types` an error by default; migrate the
types or temporarily use `-Wno-error=incompatible-pointer-types`
(clang-22.1).

## Standards status and known limits

The `standards-status` inventory classifies C11, C17, C23, and C2y support as
partial. C11 and C23 proposal coverage is still under investigation, so
`Unknown` must not be read as implemented.

### C23 gaps

Clang accepts `-std=c23` from Clang 18, but unsequenced functions (N2956) and
storage-class specifiers for compound literals (N3038) remain unsupported.
Pointer-to-array qualifier compatibility is partial: C17 and earlier miss some
pedantic diagnostics, and `?:` can compute the wrong qualified-array result
type.

Improved tag compatibility is partial from Clang 21. Attributes and extensions
can make structurally similar definitions incorrectly accepted or rejected, so
code around these cases can break when the implementation is corrected.

### C2y gaps

Clang accepts `-std=c2y` from Clang 19; `if` declarations require Clang 24.
Static assertions in expressions, bit-precise enums, multidimensional-array
matching in generic selections, and array subscripting without decay are still
marked unsupported.

## Portable adoption pattern

Select the intended mode, then probe the feature and compile the exact construct
on every supported compiler and target. Keep GNU extensions behind explicit
guards, and do not infer proposal support solely from `__STDC_VERSION__`.
