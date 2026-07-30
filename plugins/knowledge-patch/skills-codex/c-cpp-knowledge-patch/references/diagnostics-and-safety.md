# Diagnostics and Safety

Review warning-group membership, default severity, analyzer checker names, and
sanitizer composition whenever the compiler changes. A stable build flag can
select different checks in a newer release.

## Diagnostic output and policy

GCC 15 deprecates `-fdiagnostics-format=json` in favor of SARIF
(`gcc-15.1`). It can emit multiple formats with
`-fdiagnostics-add-output=`, while `-fdiagnostics-set-output=` provides detailed
destination control. GCC 16 removes JSON output (`gcc-16.1`).

GCC 16 C++ diagnostics can contain nested explanations. Use
`-fno-diagnostics-show-nesting` or `-fdiagnostics-plain-output` when a consumer
requires the previous flat form (`gcc-16.1`).

## Warning changes

### Clang 20

The following changes are from `clang-20.1`:

- `-Warray-compare` diagnoses array comparisons before C++20;
  `-Warray-compare-cxx26` diagnoses them from C++26 and is an error by default.
- `-Wnontrivial-memcall` checks memory-function destinations that are not
  trivially copyable and is implied by `-Wnontrivial-memaccess`.
- `-Winvalid-gnu-asm-cast` is enabled and defaults to an error.
  `-fheinous-gnu-extensions` is deprecated as a demotion alias.
- `-Wdangling-assignment-gsl` is enabled by default.

### GCC 15

`-Wheader-guard` is new and enabled by `-Wall`. The
`-Wtrailing-whitespace=` and `-Wleading-whitespace=` families enforce whitespace
policy (`gcc-15.1`). `-Wdefaulted-function-deleted` reports explicitly defaulted
functions that become deleted; `flag_enum` can suppress unsuitable switch
warnings. C++11 attributes are also accepted in C++98 mode.

### Clang 21

In C, `-Wextra` now includes `-Wunterminated-string-initialization`
(`clang-21.1`). Mark an intentionally unterminated C array with `nonstring`; the
C++-compatibility form cannot be silenced that way. `-Wc++-compat` also diagnoses
implicit `void *` and integer-to-enum conversions, C++ keywords and hidden tags,
tentative definitions, default initialization of `const`, and jumps bypassing
initialization. `-Wundef-true` is enabled by default before C23.

New general warnings include `-Wunique-object-duplication`, `-Wshift-bool`, and
`-Wunnecessary-virtual-specifier` through `-Wextra`. Unsafe libc calls now use
`-Wunsafe-buffer-usage-in-libc-call`.

Thread-safety analysis adds opt-in `-Wthread-safety-pointer` and reentrant
capabilities. The pointer analysis does not perform alias analysis, so interpret
both misses and reports with that limitation.

### Clang 22

Pedantic function-effect redeclaration diagnostics moved to
`-Wfunction-effect-redeclarations`, and `-Wperf-constraint-implies-noexcept` is
no longer in `-Wall` (`clang-22.1`). New warnings include `-Walloc-size`,
`-Wenum-compare-typo`, and `-Wshadow-header`. `ACQUIRED_BEFORE` and
`ACQUIRED_AFTER` no longer require `-Wthread-safety-beta`.

`-Wformat-nonliteral` can find wrappers that lack `format` or `format_matches`
annotations. Overlapping rules in `--warning-suppression-mappings=` now resolve
to the last match, not the longest one.

## Format and source contracts

Clang 21's
`__attribute__((format_matches(printf, 1, "%x %s")))` says parameter 1 must be
equivalent to the reference format (`clang-21.1`). It validates wrappers that
receive a format without its arguments and avoids inappropriate
`-Wformat-nonliteral` within the wrapper.

GCC 15 adds (`gcc-15.1`):

- a `musttail` statement attribute to require a tail call;
- `nonnull_if_nonzero` to require a pointer parameter when a separate size or
  count parameter is nonzero.

Clang 22's `malloc_span` applies malloc-like semantics to a function returning a
pointer-and-size or pointer-pair structure. `modular_format` lets a cooperating
static libc select needed `printf` features at link time (`clang-22.1`).

## Clang Static Analyzer

### Checker and configuration migration

In Clang 20 (`clang-20.1`), the analyzer validates `nonblocking` and
`nonallocating` function effects and accepts `-warning-suppression-mappings` for
per-file suppression. The Z3 cross-check timeout returns to 15 seconds from
300 ms; rlimit and equivalence-class timeout defaults are disabled.

Several alpha checkers graduated:

- `alpha.unix.Chroot` became `unix.Chroot`;
- `alpha.core.PointerSub` became `security.PointerSub`;
- taint checkers moved to `optin.taint.*`.

The nondeterministic-pointer checks moved to clang-tidy as
`bugprone-nondeterministic-pointer-iteration-order`.

Clang 21 understands `[[clang::assume]]`, adds `core.FixedAddressDereference`,
and replaces `alpha.security.ArrayBoundV2` with `security.ArrayBound`
(`clang-21.1`). The deprecated `optin.cplusplus.VirtualCall:PureOnly` option is
removed.

Clang 22 adds `core.NullPointerArithm` and `alpha.core.StoreToImmutable`
(`clang-22.1`). All `valist.*` behavior moves to `security.VAList`, and
`alpha.core.CastSize` is removed. `[[clang::suppress]]` now works in primary
templates. Analyzer model paths and taint configurations honor virtual-file-
system overlays.

## Sanitizers

### Realtime and type checking

Clang 20 introduces `-fsanitize=realtime`, which reports allocation, mutex
locking, and other unsafe library calls while running a
`[[clang::nonblocking]]` function, then exits nonzero (`clang-20.1`). Experimental
`-fsanitize=type` checks C/C++ type-based aliasing rules.

### UBSan composition and tuning

Clang 20 adds
`-fsanitize-undefined-ignore-overflow-pattern=` values
`add-signed-overflow-test`, `add-unsigned-overflow-test`,
`negated-unsigned-const`, `unsigned-post-decr-while`, `all`, and `none`
(`clang-20.1`). Special-case lists gain the `type` prefix for integer-overflow,
truncation, and enum checks. It also adds
`-f[no-]sanitize-{trap,recover}=local-bounds` and
`-f[no-]sanitize-merge`; pointer-overflow checking no longer reports `NULL + 0`
in C.

Starting in Clang 21, `-fsanitize=undefined` no longer includes
`-fsanitize=vptr`; request it explicitly (`clang-21.1`). Ignorelists can contain
positive `src:*=sanitize` entries and equivalent `type`, `fun`, `global`, and
`mainfile` forms.

Clang 22 emits detailed UBSan trap reasons in DWARF by default
(`clang-22.1`). Select `basic` or `detailed` with
`-fsanitize-debug-trap-reasons=`, disable with
`-fno-sanitize-debug-trap-reasons`, and use `-fno-sanitize-merge=` or `-O0` if
optimization coalesces distinct reasons. `__builtin_allow_sanitize_check("name")`
tests whether supported address, hardware-address, memory, or thread checking is
active for the current function after inlining and respects `no_sanitize`.

### Allocation tokens

Clang 22's `-fsanitize=alloc-token` assigns token IDs to allocation functions
for allocator-level heap organization (`clang-22.1`). Tune it with
`-falloc-token-max=`, `-fsanitize-alloc-token-fast-abi`, and
`-fsanitize-alloc-token-extended`. At compile time,
`__builtin_infer_alloc_token(args...)` returns the token inferred for allocation
arguments.
