# Optimization, Debugging, and Sanitizers

Optimizer contracts can expose latent undefined behavior. Reproduce upgrade
issues at the real optimization level, then add focused sanitizer coverage
without assuming one sanitizer group contains every relevant check.

## Alias and pointer semantics

### Type-based alias analysis (clang-20.1)

Clang emits distinct type-based alias-analysis tags for incompatible pointer
types by default. Strict-aliasing violations can therefore change behavior
after an upgrade. Correct the types; `-fno-pointer-tbaa` restores the older
behavior as a bounded compatibility measure.

### Pointer overflow (clang-20.1)

The optimizer treats pointer-addition overflow more aggressively as undefined,
so a check such as `ptr + offset < ptr` can fold to false. Compare the integer
offset before adding, use checked integer arithmetic, or diagnose with
`-fsanitize=pointer-overflow`.

`-fwrapv` covers signed integers only, `-fwrapv-pointer` covers pointers, and
`-fno-strict-overflow` implies both.

### Null-pointer arithmetic and indirect branch targets (clang-21.1)

LLVM optimizes arithmetic on null pointers more aggressively. Clang preserves
old-style `offsetof` idioms; `-fwrapv-pointer` or
`-fno-delete-null-pointer-checks` defines such arithmetic more generally.

With branch-target enforcement, `asm goto` labels are not guaranteed to begin
with `bti` or `endbr64`. Register-controlled branches must not target those
labels.

## Floating-point behavior

`-ffp-model=fast` no longer assumes finite-only math and uses promoted complex
division where possible. `-ffp-model=aggressive` selects the former `fast`
behavior (clang-20.1).

## Assembly, tail calls, and LTO

GCC 15 permits extended assembly outside functions, subject to restrictions.
Assembly that overwrites the stack red zone can declare the special
`"redzone"` clobber (gcc-15.1).

Use `-flto-incremental=` for incremental link-time optimization in GCC 15
(gcc-15.1).

A `musttail` statement attribute can require a tail call. A failed requirement
is a compiler diagnostic rather than a silent loss of the requested calling
shape (gcc-15.1).

## Destructors, modules, and variable liveness

Clang 20 adds the following controls (clang-20.1):

- `-fc++-static-destructors={all,thread-local,none}` chooses which C++ static
  destructors are registered; `all` is the default and `none` is equivalent to
  `-fno-c++-static-destructors`;
- `-fextend-variable-liveness[=all]` preserves user variables and `this` for
  optimized debugging;
- `-fextend-variable-liveness=this` limits preservation to the C++ implicit
  object; and
- `-fmodules-reduced-bmi` is the non-experimental reduced-BMI spelling.

## Profiles, ThinLTO, and debug information

Clang 21 adds `-fprofile-continuous`, `-ignore-pch`, DWARF-only
`-gkey-instructions`, and `-fthinlto-distributor=` or
`-Xthinlto-distributor=` for externally distributed ThinLTO backends
(clang-21.1).

On Windows, `-static-libclosure` changes Blocks-extension code generation only;
it does not itself change linker behavior.

Also in clang-21.1, the default `-fbracket-depth` rises from 256 to 2048, and
`-Og` enables `-fextend-variable-liveness`. AArch32 `-mtp` defaults to `auto`,
selecting `TPIDRURO` where available instead of calling `__aeabi_read_tp`; use
`-mtp=soft` when that call is required.

## Compiler controls in clang-22.1

`-gkey-instructions` is enabled by default for optimized plain C/C++ with
DWARF. `-fconstexpr-steps=0` removes the evaluation-step limit.
`-fdevirtualize-speculatively` enables otherwise-disabled speculative
virtual-call devirtualization.

`-fmatrix-memory-layout={column-major,row-major}` selects storage order for
Clang matrix types.

## Realtime and type sanitizers

`-fsanitize=realtime` reports unsafe library calls, such as allocation or mutex
locking, while executing a `[[clang::nonblocking]]` function and exits nonzero.
Experimental `-fsanitize=type` detects C and C++ type-based aliasing violations
(clang-20.1).

## UBSan selection and suppression

### Granular overflow and bounds controls (clang-20.1)

`-fsanitize-undefined-ignore-overflow-pattern=` can exclude:

- `add-signed-overflow-test`;
- `add-unsigned-overflow-test`;
- `negated-unsigned-const`;
- `unsigned-post-decr-while`;
- `all`; or
- `none`.

Sanitizer special-case lists support the `type` prefix for integer-overflow,
truncation, and enum checks. Other controls include
`-f[no-]sanitize-{trap,recover}=local-bounds` and
`-f[no-]sanitize-merge`. Pointer-overflow sanitization no longer reports
`NULL + 0` in C.

### `vptr` is no longer in the undefined group (clang-21.1)

`-fsanitize=undefined` no longer implies `-fsanitize=vptr`; request `vptr`
explicitly. Ignorelists can contain positive entries such as
`src:*=sanitize`, with equivalent `type`, `fun`, `global`, and `mainfile`
forms.

### Trap reasons and inlining-aware checks (clang-22.1)

Trapping UBSan emits detailed trap reasons into DWARF by default. Select
`basic` or `detailed` with `-fsanitize-debug-trap-reasons=`, or disable the
feature with `-fno-sanitize-debug-trap-reasons`.

Use `-fno-sanitize-merge=` or `-O0` when optimization would merge distinct
reasons. `__builtin_allow_sanitize_check("name")` reports whether a supported
address, hardware-address, memory, or thread sanitizer is active for the
current function after inlining and respects `no_sanitize`.

## Allocation-token instrumentation

`-fsanitize=alloc-token` adds token IDs to allocation functions for
allocator-level heap organization (clang-22.1). Related controls are:

- `-falloc-token-max=`;
- `-fsanitize-alloc-token-fast-abi`; and
- `-fsanitize-alloc-token-extended`.

`__builtin_infer_alloc_token(args...)` computes at compile time the token that
would be inferred for allocation arguments.

## Validation pattern

Keep optimized, debug-friendly, and sanitized configurations separate. A
debug-liveness flag can affect generated code; a sanitizer group can change
membership; and optimization can merge diagnostic sites. Verify each
configuration's flags from the actual compiler command line.
