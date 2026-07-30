# Libraries and Tooling

Pin the standard library, formatter, libclang, and compiler-plugin interfaces
separately from the compiler driver. Their source and ABI changes can affect a
build even when core language compilation succeeds.

## Standard-library behavior

### libstdc++ assertions and assumptions

Unoptimized GCC 15 builds enable libstdc++ debug assertions by default
(`gcc-15.1`). `_GLIBCXX_NO_ASSERTIONS` disables them, but first treat a new
assertion as evidence of invalid program behavior.

### Library additions

GCC 15's experimental C++26 library work includes (`gcc-15.1`):

- `views::concat`, `views::to_input`, and `views::cache_latest`;
- constexpr sorting and raw-memory algorithms;
- `<stdbit.h>` and `<stdckdint.h>`;
- `std::is_virtual_base_of` and member `visit`;
- compile-time checking of `std::format` arguments.

Its C++23 library adds the `std` and `std.compat` modules, flat associative
containers, range constructors and modifiers, and range and tuple formatting.

GCC 16 adds `std::mdspan`, starts/ends-with range algorithms, shift algorithms,
and `allocate_at_least` for C++23 (`gcc-16.1`). Its C++26 library adds:

- `std::simd`, `std::inplace_vector`, and `std::optional<T&>`;
- `std::copyable_function`, `std::function_ref`, `std::indirect`, and
  `std::polymorphic`;
- `std::owner_equal`, `<debugging>`, and string-view overloads;
- padded `mdspan` layouts, `std::philox_engine`, and
  `std::atomic_ref::address()`.

### Behavior and ABI changes

On targets supporting 128-bit integers, GCC 16 treats `__int128` as integral in
strict dialects as well as GNU modes (`gcc-16.1`). Traits such as
`std::is_integral<__int128>` can therefore change constraints and overload
selection.

`std::generate_canonical` adopts P0952R2 and produces changed sequences. Define
`_GLIBCXX_USE_OLD_GENERATE_CANONICAL` only when temporary reproduction of the
old sequence is necessary.

A C++17 `std::variant` layout correction affects the combination of an empty
base and first member; `_GLIBCXX_USE_VARIANT_CXX17_OLD_ABI` temporarily restores
the old layout. Formerly experimental C++20 components also change ABI,
including atomic waiting, semaphores, syncstream, format-argument
representation, partial ordering, some stop-token/variant combinations, and
some range adaptors. Rebuild objects that exchange those types or state.

## clang-format

### Clang 20 configuration

Clang 20 adds (`clang-20.1`):

- `BreakBinaryOperations`, `TemplateNames`, and
  `RemoveEmptyLinesInUnwrappedLines`;
- `KeepFormFeed` (enabled by GNU style) and
  `AllowShortNamespacesOnASingleLine`;
- `VariableTemplates`, `WrapNamespaceBodyWithEmptyLines`, and
  `IndentExportBlock`;
- `PenaltyBreakBeforeMemberAccess`;
- `AlignFunctionDeclarations` within `AlignConsecutiveDeclarations`.

`ReflowComments` adds `IndentOnly` and renames boolean values to
`Never`/`Always`. Ignore files support bash globstar. C has its own formatter
language, and a file can force its language in a leading comment such as:

```cpp
// clang-format Language: ObjC
```

### Clang 21 configuration

Clang 21 adds `BreakBeforeTemplateCloser`, `BinPackLongBracedList`,
`EnumTrailingComma`, `OneLineFormatOffRegex`, `SpaceAfterOperatorKeyword`, and
`MacrosSkippedByRemoveParentheses` (`clang-21.1`).

### Clang 22 configuration

`AlignAfterOpenBracket` is boolean in Clang 22; `AlwaysBreak` and `BlockIndent`
values are deprecated (`clang-22.1`). New controls include:

- `SpaceInEmptyBraces`, `NumericLiteralCase`, and
  `IndentPPDirectives: Leave`;
- the `BreakAfterOpenBracket*` and `BreakBeforeCloseBracket*` families;
- `AlignPPAndNotPP`.

Integer-separator `*MinDigits` keys are renamed to `*MinDigitsInsert`, and new
`*MaxDigitsSeparator` keys are available.

## libclang and Python bindings

### Layout and pretty-printing

Clang 20 adds `clang_isBeforeInTranslationUnit`, policy-controlled
`clang_getTypePrettyPrinted`, `clang_visitCXXBaseClasses`, and
`clang_getOffsetOfBase` (`clang-20.1`). Python bindings expose pretty printing,
base iteration, virtual-base queries, and base offsets. Affected string-returning
Python APIs return `""` instead of `None` when absent. Static access to
`CompletionChunk` or `CompletionString` properties is an error.

### Cursor and query behavior

Clang 21 adds libclang inline-assembly queries, `clang_visitCXXMethods`, and
`clang_getFullyQualifiedName`; duplicate binary-opcode APIs are deprecated
(`clang-21.1`). Python's `Cursor.from_location` returns `None` instead of a null
cursor, and most cursor methods reject null cursors. The bindings add hashable
cursors, attribute/template queries, method visitation, fully qualified names,
and `File` equality.

Clang 22 changes Python failure/null handling further (`clang-22.1`):

- `Token.cursor` returns `None` instead of a null cursor.
- `TypeKind.ELABORATED` is no longer emitted and `AccessSpecifier.NONE` is
  removed.
- `TranslationUnit.reparse()` raises on errors.
- `LIBCLANG_LIBRARY_PATH` and `LIBCLANG_LIBRARY_FILE` select libclang.
- Cursor language and inline-function queries, plus previously missing cursor,
  type, and exception kinds, are exposed.

## Embedding Clang and AST tooling

Clang 22 moves option code from `clangDriver` into `clangOptions`; downstream
tools may need both libraries (`clang-22.1`). `clangFrontend` no longer depends
transitively on `clangDriver`, so users of driver APIs must link it explicitly.

AST interface changes in Clang 22 include:

- `VarTemplateSpecializationDecl::getTemplateArgsAsWritten()` returns null for
  implicit instantiations.
- Conflicting anonymous-record members are injected as invalid
  `IndirectFieldDecl`s.
- Abbreviated function templates and generic lambdas have valid begin
  locations.
- `elaboratedType` and `dependentTemplateSpecializationType` matchers are
  removed.
- `MatchFinderOptions::IgnoreSystemHeaders`,
  `hasConditionVariableStatement` for `for`, `while`, and `switch`, and
  `arrayTypeLoc` are added.

## GCC plugins

GCC 16 moves diagnostic internals under `gcc/diagnostics/` and into the
`diagnostics::` namespace (`gcc-16.1-porting`). Plugins using context, paths,
sinks, buffering, SARIF, printing, or edit APIs need new headers and source
changes. Important replacements are:

| Old API | New API |
| --- | --- |
| `diagnostic_context` | `diagnostics::context` |
| `diagnostic_output_format` | `diagnostics::sink` |
| `diagnostic_path` | `diagnostics::paths::path` |
| `edit_context` | `diagnostics::changes::change_set` |
| `sarif_output_format` | `diagnostics::sarif_sink` |
