# Lint rules and behavior

Use this reference to decide whether a rule is stable or preview-only and to
account for expanded matching behavior. Rule availability and rule behavior
are separate concerns: a long-stable code can gain new coverage.

## Rules promoted to stable

### Python modernization, typing, and stubs

- `UP044` (`non-pep646-unpack`) became stable in 0.10.0.
- `UP045` (`non-pep604-annotation-optional`), `UP046`
  (`non-pep695-generic-class`), `UP047` (`non-pep695-generic-function`), and
  `UP049` (`private-type-parameter`) became stable in 0.12.0.
- `PYI059` (`generic-not-last-base-class`), `PYI061`
  (`redundant-none-literal`), and `UP050`
  (`useless-class-metaclass-type`) became stable in 0.13.0.
- `UP042` (`replace-string-enum`) became stable in 0.15.0.
- `RUF036` (`none-not-at-end-of-union`) and `RUF063`
  (`access-annotations-from-class-dict`) became stable in 0.16.0.

### Bugbear, comprehensions, performance, and simplification

- `FURB188` (`slice-to-remove-prefix-or-suffix`), `PLR1716`
  (`boolean-chained-comparison`), and `RUF034` (`useless-if-else`) became
  stable in 0.9.0.
- `B911` (`batched-without-explicit-strict`), `C420`
  (`unnecessary-dict-comprehension-for-iterable`), `PLC1802` (`len-test`),
  `RUF041` (`unnecessary-nested-literal`), `RUF046`
  (`unnecessary-cast-to-int`), `RUF048` (`map-int-version-parsing`), `RUF051`
  (`if-key-in-dict-del`), and `SIM905` (`split-static-string`) became stable
  in 0.10.0.
- `FURB122` (`for-loop-writes`), `FURB132`
  (`check-and-remove-from-set`), `FURB157` (`verbose-decimal-constructor`),
  `FURB162` (`fromisoformat-replace-z`), `FURB166` (`int-on-sliced-str`),
  `PLR1733` (`unnecessary-dict-index-lookup`), `RUF057`
  (`unnecessary-round`), and `RUF058` (`starmap-zip`) became stable in 0.12.0.
- `FURB116` (`f-string-number-format`) became stable in 0.13.0.
- `FURB110` (`if` expression instead of `or`), `FURB171`
  (`single-item-membership-test`), `PLC0207` (`missing-maxsplit`), `PLW0108`
  (`unnecessary-lambda`), `RUF037` (`empty-iterable-in-deque`), `RUF060`
  (`membership-in-empty-collection`), and `RUF064`
  (`non-octal-permissions`) became stable in 0.15.0.
- `FURB164` (`unnecessary-from-float`), `FURB192` (`sorted-min-max`),
  `PLR1708` (`stop-iteration-return`), and `ISC004`
  (`implicit-string-concatenation-in-collection-literal`) became stable in
  0.16.0.

### Framework, logging, paths, and security

- `A005` (`stdlib-module-shadowing`) and `A006`
  (`builtin-lambda-argument-shadowing`) became stable in 0.9.0; `A005` ignores
  stub files.
- `DTZ901` (`datetime-min-max`), `FAST003`
  (`fast-api-unused-path-parameter`), `LOG015` (`root-logger-call`), `PLW1507`
  (`shallow-copy-environ`), `PTH208` (`os-listdir`), `PTH210`
  (`invalid-pathlib-with-suffix`), and `S704` (`unsafe-markup-use`) became
  stable in 0.10.0.
- `LOG014` (`exc-info-outside-except-handler`), `PLC0415`
  (`import-outside-top-level`), `PLW0177` (`nan-comparison`), `PLW1641`
  (`eq-without-hash`), `PT028` (`pytest-parameter-with-default-argument`),
  `PT030` (`pytest-warns-too-broad`), and `PT031`
  (`pytest-warns-with-multiple-statements`) became stable in 0.12.0.
- Airflow rules `AIR002`, `AIR301`, `AIR302`, `AIR311`, and `AIR312`, plus
  `ASYNC116` (`long-sleep-not-forever`), `PTH211` (`os-symlink`), and `RUF043`
  (`pytest-raises-ambiguous-pattern`) became stable in 0.13.0.
- `ASYNC212` (blocking `httpx` call), `ASYNC240` (blocking path method),
  `ASYNC250` (blocking `input`), `B912` (`map` without explicit `strict`), and
  `RUF061` (legacy `pytest.raises`) became stable in 0.15.0.
- `AIR303` (`airflow3-incompatible-function-signature`), `CPY001`
  (`missing-copyright-notice`), and `LOG004`
  (`log-exception-outside-except-handler`) became stable in 0.16.0.

### Correctness and Ruff diagnostics

- `RUF032` (`decimal-from-float-literal`) and `RUF033`
  (`post-init-default`) became stable in 0.9.0.
- `RUF040` (`invalid-assert-message-literal-argument`) became stable in
  0.10.0.
- `RUF028` (`invalid-formatter-suppression-comment`), `RUF049`
  (`dataclass-enum`), and `RUF053` (`class-with-mixed-type-vars`) became stable
  in 0.12.0.
- `RUF059` (`unused-unpacked-variable`) became stable in 0.13.0.
- `RUF102` (`invalid-rule-code`), `RUF103` (`invalid-suppression`), and
  `RUF104` (`unmatched-suppression`) became stable in 0.15.0.
- `PLE0304` (`invalid-bool-return-type`), `PLR0917`
  (`too-many-positional-arguments`), and `RUF068`
  (`duplicate-entry-in-dunder-all`) became stable in 0.16.0.

### Runtime typing and test conventions

- `TC006` (`runtime-cast-value`) and `TC007` (`unquoted-type-alias`) became
  stable in 0.10.0.

## Preview-only additions at introduction

The following rules require preview mode at the point they are introduced;
some are promoted later as listed above.

### General correctness and modernization

- `B903` (`class-as-data-structure`) and `RUF049` (class that is both enum and
  dataclass) were added in 0.9.0.
- `RUF102` (`invalid-rule-code`), `RUF060` (`in-empty-collection`, later with
  recursive empty-collection checking), `PLC0207` (`missing-maxsplit-arg`),
  `UP050` (`useless-class-metaclass-type`), and `PTH211` (`os.symlink` to
  `Path.symlink_to`) were added in 0.11.0.
- `RUF061`, detecting calls to `pytest.raises`, `pytest.warns`, or
  `pytest.deprecated_call` not used as context managers, was added in 0.12.0.
- `DOC102` (`docstring-extraneous-parameter`), including NumPy-style
  comma-separated entries; `PLR1708` (`stop-iteration-return`); `RUF066`
  (unnecessary class property); `ISC004`; `RUF067` (non-empty initialization
  modules); and `RUF068` were added in 0.14.0.
- `RUF069` (float equality), `D420` (docstring section order), `PLR1712`
  (temporary-variable swap), `RUF070` (assignment immediately before `yield`),
  `B043` (constant-name `delattr`), and `RUF071` (`os.path.commonprefix`) were
  added in 0.15.0.
- Later 0.15.0 additions were `RUF050` (unnecessary `if`), `RUF072` (useless
  `finally`), `RUF073` (percent-formatting an f-string), `PLW0717` (too many
  `try` statements), `RUF074` (incorrect decorator order), `RUF075` (fallible
  context manager), `ASYNC119` (yield in a context manager implemented as an
  async generator), `D421` (property docstring beginning with a verb), and
  `UP051` (deprecated `abc` decorators).

### Airflow preview coverage

In 0.15.0, preview added `AIR321` for Airflow 3.1 imports; `AIR003` and `AIR304`
for parse-time or runtime-varying DAG values; `AIR201` for XCom pulls in
template strings; `AIR004` for branch tasks used as short-circuit tasks; and
`AIR202` for implicit multiple outputs.

## Stabilized matching and diagnostic behavior

### Typing, imports, and annotations

- `TC008` more eagerly applies `quoted-type-alias` inside `TYPE_CHECKING` but
  ignores stubs; `PLW1641` also ignores stubs (preview, 0.9.0).
- `PYI006` applies to non-stub files; fixes for `PYI041`
  (`redundant-numeric-union`) and `PYI016` (`duplicate-union-members`) are
  stable (0.9.0).
- `RET503` understands return annotations of `typing.Never` (0.9.0).
- `__new__` is no longer flagged by `N804`; `PLW0211` handles it instead.
  `PYI019` better detects custom `TypeVar`s replaceable by `Self` and spans the
  full function header. `N803` ignores arguments under `typing.override`
  (0.10.0).
- `UP024` recognizes `resource.error` as a deprecated `OSError` alias. `UP035`
  rewrites `get_type_hints()` only for Python 3.13+ (0.11.0).
- `PYI019` checks string annotations, and `UP007`/`UP045` do not fix
  `Optional[None]` (0.12.0).
- Preview `UP043` runs in stubs; `UP008` only applies when a `__class__` cell
  exists (0.13.0). Stable `UP043` later runs on stubs before Python 3.13, and
  `PYI016` recognizes duplicates involving `typing.Optional` (0.15.0).
- `COM812` and `COM819` cover trailing commas in PEP 695 type-parameter lists;
  `PLC0414` skips `__init__.py`, avoiding conflict with an `F401` fix
  (0.13.0).
- `UP045` covers string arguments to `typing.cast`; `PIE794` detects duplicated
  declared class fields (0.14.0).
- Preview `F811` catches annotated redeclarations and duplicate imports inside
  `TYPE_CHECKING`. `PYI033`, now named `legacy-type-comment`, runs on Python
  files too (0.15.0).
- `FA102` recognizes more PEP 585-compatible APIs, including
  `collections.abc`, and `UP019` recognizes `typing_extensions.Text`
  (0.16.0).

### Tests and frameworks

- `PT006` covers `pytest.parametrize` outside decorators and calls using
  keyword arguments; `E402` permits `pytest.importorskip` between imports;
  `RUF008` and `RUF009` support `attrs` (0.9.0).
- `PT012` and `PT031` allow empty-bodied `for` statements inside
  `pytest.raises` and `pytest.warns` (0.10.0).
- `PT019` does not suggest `usefixtures` for `parametrize` values, and
  `FAST003` accepts class dependencies (0.11.0).
- `B017` checks direct calls to unittest and pytest exception assertions, not
  only context-manager forms. `PGH005` covers `AsyncMock` methods such as
  `not_awaited` (0.13.0).
- `DJ001` applies to annotated Django fields. Airflow checks cover
  `DatasetEvent`, `Param`, removed `DAG.create_dagrun`, deprecated
  `DAG(concurrency=...)`, and invalid positional arguments to
  `HookLineageCollector.create_asset` and `Asset`/`Dataset` (0.14.0).
- Pytest-style checks include `pytest_asyncio` fixtures (preview, 0.15.0).

### Expressions, strings, and control flow

- `PLE1310` applies to values known as `str` or `bytes`; `PLW1508` detects
  invalidly typed defaults to `os.environ.get`; `FURB169` recognizes
  `type(expr) is type(None)` for non-name expressions (0.10.0).
- `RUF005` covers slices, `FURB171` covers `set` and `frozenset`, `PERF401`
  can turn list constructor calls into comprehensions, and `PERF403` gains a
  fix (0.11.0).
- `S308` accepts raw strings. `S603` trusts `str`, `list[str]`, and tuples of
  string literals. `S608` skips expressionless f-strings (0.11.0).
- Stable `SIM108` can simplify further to `or`; `FBT001` covers annotations
  containing `bool`, such as `bool | int` and `Optional[bool]` (0.12.0).
- `EM101` checks byte-string exception messages; `PLE2502` detects U+061C
  Arabic Letter Mark; preview `RUF060` handles empty f-strings (0.13.0).
- `B006` recognizes more guaranteed-mutable defaults using tuples, generators,
  and assignment expressions. `RUF065` covers more eager logging conversions
  while excluding nonsimple `str()` calls and complex specifiers. `FURB105`
  detects empty f-strings, `B901` catches embedded `yield`, `RUF052` catches
  more dummy-variable uses, and `PLW0133` covers subclasses of built-in
  exceptions (0.14.0).
- `A003` finds built-in attribute shadowing in decorators, defaults, and other
  attribute definitions. `SIM905` fixes `split(maxsplit=...)` without an
  explicit separator; `SIM910` accepts more dictionary-key expressions
  (0.15.0).
- `RUF019` treats f-string interpolation as a possible side effect. Preview
  permits `FURB189` subclasses of built-ins in stubs and adds an `E402` fix
  (0.15.0).
- `BLE001` is suppressed when an exception is logged through methods other
  than `critical`, `error`, or `exception`. `INT001`–`INT003` recognize more
  common `gettext` patterns, including assignment to `builtins._`. `S310`
  resolves local string-literal bindings, and `S508`/`S509` support the
  recommended API in newer PySNMP (0.16.0).

### Fixes that became available or stable

- Stable fixes improved for `PLR1714` (`repeated-equality-comparison`),
  `SIM103` (`needless-bool`), and `PYI018` (`unused-private-type-var`)
  (0.10.0).
- `E712` and `ISC003` gained fixes; `SIM117` was enabled in preview
  (0.11.0).
- `FURB129` (`readlines-in-for`) became always safe (0.12.0).
- The `SIM117` fix became always safe (0.13.0).
- `UP008` gained a safe fix when comments are preserved (0.15.0).

Consult [suppressions-and-fixes.md](suppressions-and-fixes.md) for the unsafe
matrix and removed or conditional fixes.
