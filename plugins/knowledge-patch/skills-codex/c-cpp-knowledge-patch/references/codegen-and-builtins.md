# Code Generation and Builtins

Builtin names, constant-expression eligibility, optimization assumptions, and
debug/profile controls are compiler-version interfaces. Guard nonstandard
facilities and test optimized behavior on every supported compiler.

## Optimization and floating-point semantics

Clang 20's `-ffp-model=fast` no longer implies finite-only math and uses promoted
complex division when possible (`clang-20.1`). The new
`-ffp-model=aggressive` selects the former fast behavior.

GCC 15 enables `-fassume-sane-operators-new-delete` by default (`gcc-15.1`). If
replacement global allocation functions deliberately expose global state, use
`-fno-assume-sane-operators-new-delete` and audit the optimization impact.

Clang 22 adds `-fdevirtualize-speculatively` for normally disabled speculative
virtual-call devirtualization and `-fmatrix-memory-layout={column-major,row-major}`
for Clang matrix storage (`clang-22.1`).

## Elementwise, reduction, and low-level builtins

### Clang 20

Clang 20 adds (`clang-20.1`):

- `__builtin_elementwise_popcount` and `__builtin_elementwise_fmod`;
- `__builtin_elementwise_minimum` and `__builtin_elementwise_maximum`;
- `__builtin_elementwise_atan2`;
- `__builtin_common_type`.

Integer reductions, elementwise bit/count/saturating operations, floating-point
comparisons, `__builtin_signbit`, and `__builtin_abs` are now valid in constant
expressions.

For a flexible-array member with `counted_by`,
`__builtin_counted_by_ref(flexible_member)` accesses the named counter. Initialize it
before sanitizer-checked flexible-array access:

```c
*__builtin_counted_by_ref(p->items) = count;
```

### Clang 21

New builtins are `__builtin_elementwise_exp10`,
`__builtin_elementwise_minnum`, `__builtin_elementwise_maxnum`,
`__builtin_invoke`, and `__builtin_get_vtable_pointer` (`clang-21.1`).
`__builtin___clear_cache` now has the GCC-compatible signature
`void(void *, void *)`, not `char *` parameters.

### Clang 22

Clang 22 adds (`clang-22.1`):

- `__builtin_elementwise_ldexp`, `__builtin_elementwise_fshl`, and
  `__builtin_elementwise_fshr`;
- `__builtin_elementwise_minnumnum` and
  `__builtin_elementwise_maxnumnum`;
- generic `__builtin_bswapg` and `__builtin_stack_address()`;
- masked load, store, gather, and scatter builtins.

Integer elementwise min/max and abs work in constant expressions. Fixed boolean
vectors work with generic popcount/count-zero builtins and as `?:` conditions.
`__builtin_assume_dereferenceable` accepts runtime sizes.

## Attributes and annotations

### Clang source annotations

Clang 20 introduces (`clang-20.1`):

- `[[clang::no_specializations]]`;
- `[[clang::lifetime_capture_by(X)]]`;
- `[[clang::coro_await_elidable]]` and
  `[[clang::coro_await_elidable_argument]]`;
- `__attribute__((format(syslog, ...)))`.

`swift_attr` can annotate types. Attributes after a namespace name are no longer
accepted.

Clang 21 lets X86-64 globals select `__attribute__((model("small")))` or
`model("large")` independently of the translation unit's code model
(`clang-21.1`). `[[clang::atomic(...)]]` controls AMDGPU atomic metadata for a
statement.

On Itanium-ABI targets, Clang 22's `[[gnu::gcc_struct]]` requests Itanium record
layout even when Microsoft bit-field layout is active (`clang-22.1`).
`[[clang::cfi_unchecked_callee]]` propagates from declaration to definition and
also disables `-fsanitize=function` instrumentation for affected indirect calls.

## Assembly

GCC 15 permits restricted extended assembly at file scope (`gcc-15.1`). Inline
assembly that overwrites the stack red zone can declare the special `"redzone"`
clobber. In C++, assembly strings can be produced by constant evaluation.

Clang 21 accepts constant-expression GNU assembly strings, for example
(`clang-21.1`):

```cpp
asm((std::string_view("nop")) ::: (std::string_view("memory")));
```

## LTO, profiling, and debug information

GCC 15 offers incremental LTO through `-flto-incremental=` (`gcc-15.1`).

Clang 20 adds (`clang-20.1`):

- `-fc++-static-destructors={all,thread-local,none}`; `all` is the default and
  `none` is equivalent to `-fno-c++-static-destructors`;
- `-fextend-variable-liveness[=all]` for all user variables and `this`, and
  `=this` for only the implicit C++ object;
- the stable `-fmodules-reduced-bmi` spelling.

Clang 21 adds `-fprofile-continuous`, `-ignore-pch`, DWARF-only
`-gkey-instructions`, and
`-fthinlto-distributor=`/`-Xthinlto-distributor=` for externally distributed
ThinLTO backends (`clang-21.1`). On Windows, `-static-libclosure` changes only
Blocks code generation; it does not independently change linker behavior.
`-Og` now enables `-fextend-variable-liveness`, and the default
`-fbracket-depth` limit rises from 256 to 2048.

Clang 22 enables `-gkey-instructions` by default for optimized plain C/C++ with
DWARF (`clang-22.1`). `-fconstexpr-steps=0` removes the constant-evaluation step
limit.
