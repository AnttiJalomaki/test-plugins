# Diagnostics and Tooling

Audit warning policy, machine-readable output, formatter configuration,
analyzer checker names, library dependencies, and binding behavior as one
upgrade surface.

## Diagnostic output consumers

### GCC machine-readable output (gcc-15.1 and gcc-16.1)

GCC 15 deprecated the `json` value of `-fdiagnostics-format=`. It can emit
multiple formats from one compilation with `-fdiagnostics-add-output=`, while
`-fdiagnostics-set-output=` provides detailed output control.

GCC 16 removed JSON diagnostic output. Migrate machine-readable consumers to
SARIF.

### Nested C++ diagnostics (gcc-16.1)

C++ diagnostics can contain nested explanations. Use
`-fno-diagnostics-show-nesting` or `-fdiagnostics-plain-output` when a tool or
user requires the former flat presentation.

## Warning-policy changes

### Clang comparison, memory, assembly, and lifetime warnings (clang-20.1)

- `-Warray-compare` diagnoses array comparisons before C++20.
- `-Warray-compare-cxx26` diagnoses them from C++26 and is an error by default.
- `-Wnontrivial-memcall` checks memory-function destinations that are not
  trivially copyable and is implied by `-Wnontrivial-memaccess`.
- `-Winvalid-gnu-asm-cast` is enabled and defaults to an error.
- `-fheinous-gnu-extensions` is deprecated as an alias for demoting that
  assembly diagnostic.
- `-Wdangling-assignment-gsl` is enabled by default.

### Header guards and whitespace (gcc-15.1)

`-Wheader-guard` is enabled by `-Wall`.
`-Wtrailing-whitespace=` and `-Wleading-whitespace=` enforce whitespace
policies.

### Chained comparisons (clang-21.1)

Expressions such as `a < b < c`, including fold expressions over comparison
operators, are errors by default. `-Wno-error=parentheses` demotes them during
migration.

### Clang warning and thread-safety expansion (clang-21.1)

New warnings include `-Wunique-object-duplication`, `-Wshift-bool`, and the
`-Wextra` member `-Wunnecessary-virtual-specifier`. Unsafe libc calls use
`-Wunsafe-buffer-usage-in-libc-call`.

Thread-safety analysis adds opt-in `-Wthread-safety-pointer` and reentrant
capabilities. The pointer check performs no alias analysis.

### Warning-suppression mappings (clang-22.1)

`--warning-suppression-mappings=` resolves overlapping entries by taking the
last match rather than the longest match. Reorder existing files if they relied
on longest-match precedence.

### Clang diagnostic groups (clang-22.1)

Pedantic function-effect redeclaration checks moved to
`-Wfunction-effect-redeclarations`, and
`-Wperf-constraint-implies-noexcept` left `-Wall`. New warnings include
`-Walloc-size`, `-Wenum-compare-typo`, and `-Wshadow-header`.
`ACQUIRED_BEFORE` and `ACQUIRED_AFTER` no longer require
`-Wthread-safety-beta`. `-Wformat-nonliteral` can detect wrappers missing
`format` or `format_matches` annotations.

### Unused-but-set sensitivity (gcc-16.1-porting)

`-Wunused-but-set-variable` and `-Wunused-but-set-parameter`, including their
`-Wall` or `-Wextra` forms, default to level 3. Level 2 stops counting
increment/decrement as uses; level 3 also stops counting compound assignment
when the old value is not used on the right-hand side. Level 1 is closest to
the prior behavior:

```text
-Wunused-but-set-variable=1 -Wunused-but-set-parameter=1
```

## clang-format

### Policy and language controls (clang-20.1)

New options include `BreakBinaryOperations`, `TemplateNames`,
`RemoveEmptyLinesInUnwrappedLines`, `KeepFormFeed`,
`AllowShortNamespacesOnASingleLine`, `VariableTemplates`,
`WrapNamespaceBodyWithEmptyLines`, `IndentExportBlock`, and
`PenaltyBreakBeforeMemberAccess`. GNU style enables `KeepFormFeed`.

`AlignConsecutiveDeclarations` adds `AlignFunctionDeclarations`.
`ReflowComments` adds `IndentOnly` and renames its boolean values to
`Never`/`Always`. Ignore files support bash globstar. C is formatted as its own
language, and a header can force its language with a first-line comment such as
`// clang-format Language: ObjC`.

### Additional layout policies (clang-21.1)

New settings are `BreakBeforeTemplateCloser`, `BinPackLongBracedList`,
`EnumTrailingComma`, `OneLineFormatOffRegex`, `SpaceAfterOperatorKeyword`, and
`MacrosSkippedByRemoveParentheses`.

### Renamed and new keys (clang-22.1)

`AlignAfterOpenBracket` is boolean; `AlwaysBreak` and `BlockIndent` are
deprecated. New controls include `SpaceInEmptyBraces`, `NumericLiteralCase`,
`IndentPPDirectives: Leave`, `BreakAfterOpenBracket*`,
`BreakBeforeCloseBracket*`, and `AlignPPAndNotPP`.

Integer-separator `*MinDigits` keys were renamed to `*MinDigitsInsert`, and
`*MaxDigitsSeparator` keys were added.

## Embedding and AST tooling

### Clang library linkage (clang-22.1)

Options code moved from `clangDriver` to the new `clangOptions` library, so
downstream tools may need both. `clangFrontend` no longer depends transitively
on `clangDriver`; consumers of driver APIs must link it explicitly.

### AST matcher and declaration changes (clang-22.1)

`VarTemplateSpecializationDecl::getTemplateArgsAsWritten()` returns null for
implicit instantiations. Anonymous-record members are injected as invalid
`IndirectFieldDecl`s even on name conflicts. Abbreviated function templates and
generic lambdas have valid begin locations.

The `elaboratedType` and `dependentTemplateSpecializationType` matchers were
removed. Additions include `MatchFinderOptions::IgnoreSystemHeaders`,
`hasConditionVariableStatement` support for `for`, `while`, and `switch`, and
the `arrayTypeLoc` matcher.

## libclang and Python bindings

### Layout APIs and empty strings (clang-20.1)

libclang adds `clang_isBeforeInTranslationUnit`, policy-controlled
`clang_getTypePrettyPrinted`, `clang_visitCXXBaseClasses`, and
`clang_getOffsetOfBase`. Python exposes pretty-printing, base iteration,
virtual-base queries, and base offsets.

Affected Python string-returning interfaces return `""` rather than `None`
when absent. Static access to `CompletionChunk` or `CompletionString`
properties is an error.

### Cursor and method APIs (clang-21.1)

libclang adds inline-assembly queries, `clang_visitCXXMethods`, and
`clang_getFullyQualifiedName`; duplicate binary-opcode APIs are deprecated.

Python's `Cursor.from_location` returns `None` instead of a null cursor, and
most cursor methods reject null cursors. Bindings add hashable cursors,
attribute/template queries, method visits, fully qualified names, and `File`
equality.

### Failure, null, and library selection behavior (clang-22.1)

`Token.cursor` returns `None` instead of a null cursor.
`TypeKind.ELABORATED` is no longer produced, `AccessSpecifier.NONE` was
removed, and `TranslationUnit.reparse()` raises on errors.

`LIBCLANG_LIBRARY_PATH` and `LIBCLANG_LIBRARY_FILE` select libclang. Bindings
also expose cursor language, inline-function queries, and previously missing
cursor, type, and exception kinds.

## Static Analyzer

### Effects, suppression, and checker renames (clang-20.1)

The analyzer verifies `nonblocking` and `nonallocating` effects and accepts
`-warning-suppression-mappings` for per-file suppression. The Z3 cross-check
timeout returned from 300 ms to 15 seconds; rlimit and equivalence-class
timeout defaults are disabled.

Checker migrations include:

- `alpha.unix.Chroot` to `unix.Chroot`;
- `alpha.core.PointerSub` to `security.PointerSub`;
- taint checkers to `optin.taint.*`; and
- nondeterministic-pointer checks to clang-tidy
  `bugprone-nondeterministic-pointer-iteration-order`.

### Assumptions and array bounds (clang-21.1)

The analyzer understands `[[clang::assume]]` and adds
`core.FixedAddressDereference`. `alpha.security.ArrayBoundV2` graduated to
`security.ArrayBound`, replacing the alpha checker. The deprecated
`optin.cplusplus.VirtualCall:PureOnly` option was removed.

### Checker consolidation and VFS support (clang-22.1)

New checkers are `core.NullPointerArithm` and
`alpha.core.StoreToImmutable`. All `valist.*` functionality moved to
`security.VAList`, and `alpha.core.CastSize` was removed.
`[[clang::suppress]]` works in primary templates. Analyzer model paths and
taint configurations honor virtual-file-system overlays.

## GCC plugin API migration

GCC 16 moved diagnostic internals below `gcc/diagnostics/` and into the
`diagnostics::` namespace (gcc-16.1-porting). Plugins using context, path, sink,
buffering, SARIF, printing, or edit APIs require new headers and names:

| Old | New |
|---|---|
| `diagnostic_context` | `diagnostics::context` |
| `diagnostic_output_format` | `diagnostics::sink` |
| `diagnostic_path` | `diagnostics::paths::path` |
| `edit_context` | `diagnostics::changes::change_set` |
| `sarif_output_format` | `diagnostics::sarif_sink` |
