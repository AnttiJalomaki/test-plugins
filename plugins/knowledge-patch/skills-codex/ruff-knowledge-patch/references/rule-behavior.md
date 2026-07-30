# Rule Behavior

## Type-checking guards and annotations

Ruff recognizes any local variable named `TYPE_CHECKING` as a type-checking
guard. Legacy `if 0:` and `if False:` guards are no longer recognized; use a
local `TYPE_CHECKING` variable (0.10.0-guide).

`TC008` preview behavior applies `quoted-type-alias` more eagerly inside
`TYPE_CHECKING` blocks but ignores it in stubs. Preview `PLW1641` also ignores
`eq-without-hash` in stubs (0.9.0).

`RET503` understands functions annotated with a `typing.Never` return
(0.9.0).

`PYI006` applies to non-stub files (0.9.0).

`PYI019` more accurately recognizes custom `TypeVar`s that can be replaced
with `Self`, and its diagnostic spans the full function header (0.10.0). It
also checks string annotations (0.12.0).

`N803` ignores arguments of functions decorated with `typing.override`
(0.10.0).

In preview, `UP043` runs in stub files (0.13.0); it later runs on stubs before
Python 3.13 as stable behavior (0.15.0).

`UP045` applies to string arguments of `typing.cast` (0.14.0).

Bandit import checks `S401`-`S415` allow imports inside `TYPE_CHECKING`.
`TC001`-`TC003` avoid strict behavior when `lint.future-annotations` is
enabled. Preview `F811` detects annotated redeclarations and duplicate imports
inside `TYPE_CHECKING` (0.15.0).

`PYI033` also runs on Python files and is named `legacy-type-comment`
(0.15.0).

Preview adds an `E402` fix and permits `FURB189` subclasses of built-ins in
stub files (0.15.0).

`UP019` recognizes `typing_extensions.Text` as well as `typing.Text`
(0.16.0).

`FA102` recognizes more PEP 585-compatible APIs, including APIs from
`collections.abc` (0.16.0).

## pytest behavior

`PT006` covers `pytest.parametrize` calls outside decorators and calls using
keyword arguments (0.9.0).

`E402` ignores `pytest.importorskip` between import statements (0.9.0).

`PT012` and `PT031` permit `for` statements with empty bodies inside
`pytest.raises` and `pytest.warns` context managers (0.10.0).

`PT019` does not recommend `usefixtures` for `parametrize` values, and
`FAST003` accepts class dependencies (0.11.0).

Pytest-style checks include `pytest_asyncio` fixtures (0.15.0).

`PT006` calls with ambiguous `argnames` or `argvalues` no longer receive a fix
(0.15.0).

## Names, attributes, and classes

`attrs` support was added to `RUF008` and `RUF009` (0.9.0).

`__new__` methods are no longer flagged by `N804`; `PLW0211` handles them
instead (0.10.0).

`PLW1641` (`eq-without-hash`) became stable (0.12.0).

`UP008` only applies when the `__class__` cell exists (0.13.0).

`B006` recognizes more guaranteed-mutable default expressions built from
tuples, generators, and assignment expressions (0.14.0).

`DJ001` applies to annotated Django fields. `PLW0133` recognizes subclasses of
built-in exceptions, and `PIE794` finds duplicated declared class fields
(0.14.0).

`A003` finds built-in attribute shadowing in decorators, default arguments, and
other attribute definitions (0.15.0).

## Strings, formatting expressions, and messages

`PLE1310` applies to objects known to have type `str` or `bytes` (0.10.0).

`EM101` checks byte strings in exception messages (0.13.0).

`PLE2502` detects U+061C, Arabic Letter Mark (0.13.0).

`FURB105` detects empty f-strings (0.14.0).

`RUF065` covers more eager logging conversions, while excluding `str()` calls
that are not simple conversions and complex conversion specifiers (0.14.0).

`RUF060` correctly handles empty f-strings (0.13.0).

`RUF019` treats f-string interpolation as a possible side effect (0.15.0).

`INT001`, `INT002`, and `INT003` recognize more common `gettext` patterns,
including assignment to `builtins._` (0.16.0).

## Collections, comparisons, and control flow

`RUF005` covers slices, and `FURB171` covers `set` and `frozenset` calls
(0.11.0).

`PERF401` can replace list-constructor calls with comprehensions, and
`PERF403` has a fix (0.11.0).

`SIM108` can further simplify a conditional expression to `or` (0.12.0).

`FBT001` covers annotations containing `bool`, including `bool | int` and
`Optional[bool]` (0.12.0).

`B017` checks direct calls to unittest and pytest exception assertions, not
only context-manager forms (0.13.0).

`COM812` and `COM819` check trailing commas in PEP 695 type-parameter lists
(0.13.0).

`PGH005` covers `AsyncMock` methods such as `not_awaited` (0.13.0).

`B901` catches `yield` expressions embedded in other statements, and `RUF052`
finds more uses of dummy variables (0.14.0).

`SIM905` can fix `split(maxsplit=...)` with no explicit separator. `SIM910`
accepts more kinds of dictionary key expressions (0.15.0).

`PYI016` recognizes duplicate union members involving `typing.Optional`
(0.15.0).

## Imports and APIs

`EXE003` accepts `uv run` shebangs (0.11.0).

`S308` accepts raw strings. `S603` treats `str`, `list[str]`, and tuples of
string literals as trusted input (0.11.0).

`UP024` recognizes `resource.error` as a deprecated alias of `OSError`.
`UP035` rewrites `get_type_hints()` only on Python 3.13 and newer (0.11.0).

`PLC0414` does not apply to `__init__.py`, avoiding conflict with a suggested
`F401` fix (0.13.0).

Airflow checks cover `DatasetEvent`, `Param`, removed `DAG.create_dagrun`, the
deprecated `DAG(concurrency=...)` argument, and invalid positional arguments
to `HookLineageCollector.create_asset` and to `Asset` or `Dataset` (0.14.0).

`S310` resolves local string-literal bindings. `S508` and `S509` support the
recommended APIs in newer PySNMP versions (0.16.0).

## Diagnostics, suppressions, and logging

`UP015` highlights only the redundant mode argument, rather than the entire
`open` call. Existing `noqa` comments may need to move (0.10.0).

`PLW1508` detects incorrectly typed default arguments to `os.environ.get`
(0.10.0).

`FURB169` recognizes `type(expr) is type(None)` even when `expr` is not a name
(0.10.0).

`S608` skips expressionless f-strings (0.11.0).

`ERA001` ignores `ruff:disable` and `ruff:enable` control comments
(0.14.0).

Preview validates suppression ranges, reports invalid or unmatched comments,
and consolidates diagnostics for matched pairs. `RUF100` reports unused range
suppressions, and `--add-noqa` can attach a reason to generated suppressions
(0.14.0).

`BLE001` is suppressed when an exception is logged through logging methods
other than `critical`, `error`, or `exception` (0.16.0).

## Stable fixes and corrected behavior

`FURB129` (`readlines-in-for`) has an always-safe stable fix (0.12.0).

The `SIM117` fix was enabled in preview in 0.11.0 and became always safe in
0.13.0.

`E712` and `ISC003` gained fixes (0.11.0).

`UP007` and `UP045` do not offer a fix for `Optional[None]` (0.12.0).

`UP008` has a safe fix when it preserves comments (0.15.0).
