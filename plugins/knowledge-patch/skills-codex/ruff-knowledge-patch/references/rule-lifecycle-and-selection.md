# Rule Lifecycle and Selection

Use exact codes in configuration and suppression comments. The catalogs below
are grouped by what the checks cover.

## Renamed, recoded, deprecated, and removed

- `A005` is now `stdlib-module-shadowing`, renamed from
  `builtin-module-shadowing`; it ignores stub files (0.9.0).
- Preview code `RUF025` moved to `RUF037` (0.9.0).
- `unsafe-markup-use` moved from `RUF035` to `S704`; update configuration and
  suppressions (0.10.0-guide).
- `UP038` (`non-pep604-isinstance`) and `S320`
  (`suspicious-xmle-tree-usage`) were deprecated (0.10.0).
- `S320` was removed, and `pandas-df-variable-name` became deprecated
  (0.12.0).
- Deprecated prefix or group selection stopped activating deprecated rules;
  `PD901` (`pandas-df-variable-name`) and `UP038` were then removed
  (0.13.0-guide).
- The Airflow 3 preview rules were reorganized: old `AIR301` became `AIR002`,
  old `AIR302` became `AIR301`, and old `AIR303` became `AIR302`. Additional
  checks were split into `AIR311` and `AIR312`, with some `AIR312` checks later
  moved back to `AIR302`. Update selections, ignores, and `noqa` codes
  (0.11.0).

## Stable framework, security, and runtime checks

- Airflow rules `AIR002`, `AIR301`, `AIR302`, `AIR311`, and `AIR312`
  (0.13.0); `AIR303` (`airflow3-incompatible-function-signature`) (0.16.0)
- `ASYNC116` (`long-sleep-not-forever`) (0.13.0); `ASYNC212` (blocking
  `httpx` call), `ASYNC240` (blocking path method), and `ASYNC250` (blocking
  `input`) (0.15.0)
- `B911` (`batched-without-explicit-strict`) (0.10.0) and `B912` (`map`
  without explicit `strict`) (0.15.0)
- `CPY001` (`missing-copyright-notice`) (0.16.0)
- `DTZ901` (`datetime-min-max`) (0.10.0)
- `FAST003` (`fast-api-unused-path-parameter`) (0.10.0)
- `LOG015` (`root-logger-call`) (0.10.0), `LOG014`
  (`exc-info-outside-except-handler`) (0.12.0), and `LOG004`
  (`log-exception-outside-except-handler`) (0.16.0)
- `PTH208` (`os-listdir`) and `PTH210`
  (`invalid-pathlib-with-suffix`) (0.10.0), plus `PTH211` (`os-symlink`)
  (0.13.0)
- `PT028` (`pytest-parameter-with-default-argument`), `PT030`
  (`pytest-warns-too-broad`), and `PT031`
  (`pytest-warns-with-multiple-statements`) (0.12.0)
- `S704` (`unsafe-markup-use`) (0.10.0)

## Stable modernization, typing, and naming checks

- `A005` (`stdlib-module-shadowing`) and `A006`
  (`builtin-lambda-argument-shadowing`) (0.9.0)
- `PYI059` (`generic-not-last-base-class`) and `PYI061`
  (`redundant-none-literal`) (0.13.0)
- `TC006` (`runtime-cast-value`) and `TC007` (`unquoted-type-alias`) (0.10.0)
- `UP044` (`non-pep646-unpack`) (0.10.0)
- `UP045` (`non-pep604-annotation-optional`), `UP046`
  (`non-pep695-generic-class`), `UP047` (`non-pep695-generic-function`), and
  `UP049` (`private-type-parameter`) (0.12.0)
- `UP050` (`useless-class-metaclass-type`) (0.13.0)
- `UP042` (replace string enum) (0.15.0)
- `RUF036` (`none-not-at-end-of-union`) and `RUF063`
  (`access-annotations-from-class-dict`) (0.16.0)

The fixes for `PYI041` (`redundant-numeric-union`) and `PYI016`
(`duplicate-union-members`) became stable in 0.9.0. Fixes or fix improvements
for `PYI018` (`unused-private-type-var`) became stable in 0.10.0.

## Stable collection, simplification, and performance checks

- `C420` (`unnecessary-dict-comprehension-for-iterable`) (0.10.0)
- `FURB188` (`slice-to-remove-prefix-or-suffix`) (0.9.0)
- `FURB122` (`for-loop-writes`), `FURB132`
  (`check-and-remove-from-set`), `FURB157`
  (`verbose-decimal-constructor`), `FURB162`
  (`fromisoformat-replace-z`), and `FURB166` (`int-on-sliced-str`) (0.12.0)
- `FURB116` (`f-string-number-format`) (0.13.0)
- `FURB110` (`if` expression instead of `or`) and `FURB171` (single-item
  membership test) (0.15.0)
- `FURB164` (`unnecessary-from-float`) and `FURB192` (`sorted-min-max`)
  (0.16.0)
- `PLC1802` (`len-test`) (0.10.0) and `PLC0207` (missing `maxsplit`) (0.15.0)
- `PLR1716` (`boolean-chained-comparison`) (0.9.0) and `PLR1733`
  (`unnecessary-dict-index-lookup`) (0.12.0)
- `PLW0108` (`unnecessary-lambda`) (0.15.0)
- `RUF037` (empty iterable in `deque`), `RUF060` (membership in an empty
  collection), `RUF061` (legacy `pytest.raises`), and `RUF064` (non-octal
  permissions) (0.15.0)
- `RUF057` (`unnecessary-round`) and `RUF058` (`starmap-zip`) (0.12.0)
- `SIM905` (`split-static-string`) (0.10.0)

The fixes for `PLR1714` (`repeated-equality-comparison`) and `SIM103`
(`needless-bool`) became stable in 0.10.0.

## Stable correctness, structure, and policy checks

- `ISC004` (`implicit-string-concatenation-in-collection-literal`) (0.16.0)
- `PLC0415` (`import-outside-top-level`) (0.12.0)
- `PLE0304` (`invalid-bool-return-type`) (0.16.0)
- `PLR0917` (`too-many-positional-arguments`) and `PLR1708`
  (`stop-iteration-return`) (0.16.0); `PLR1708` detects raising
  `StopIteration` from a generator when `return` should be used
- `PLW1507` (`shallow-copy-environ`) (0.10.0)
- `PLW0177` (`nan-comparison`) and `PLW1641` (`eq-without-hash`) (0.12.0)
- `RUF032` (`decimal-from-float-literal`), `RUF033`
  (`post-init-default`), and `RUF034` (`useless-if-else`) (0.9.0)
- `RUF040` (`invalid-assert-message-literal-argument`), `RUF041`
  (`unnecessary-nested-literal`), `RUF046` (`unnecessary-cast-to-int`),
  `RUF048` (`map-int-version-parsing`), and `RUF051` (`if-key-in-dict-del`)
  (0.10.0)
- `RUF028` (`invalid-formatter-suppression-comment`), `RUF049`
  (`dataclass-enum`), and `RUF053` (`class-with-mixed-type-vars`) (0.12.0)
- `RUF043` (`pytest-raises-ambiguous-pattern`) and `RUF059`
  (`unused-unpacked-variable`) (0.13.0)
- `RUF102` (invalid rule code), `RUF103` (invalid suppression), and `RUF104`
  (unmatched suppression) (0.15.0)
- `RUF068` (`duplicate-entry-in-dunder-all`) (0.16.0)

## Preview data-modeling, typing, and API checks

- `B903` (`class-as-data-structure`) and `RUF049` for classes that are both
  enums and dataclasses (0.9.0)
- The split of `UP007` for `Union` and `UP045` for `Optional` (0.9.0)
- `UP050` (`useless-class-metaclass-type`) (0.11.0)
- `DOC102` (`docstring-extraneous-parameter`), including NumPy-style
  comma-separated parameter entries, and `RUF066` (unnecessary class
  properties), honoring `lint.pydocstyle.property-decorators` (0.14.0)
- `D420` (docstring section order), `D421` (property docstring starting with a
  verb), `RUF074` (incorrect decorator order), and `UP051` (deprecated `abc`
  decorators) (0.15.0)

## Preview collections, paths, and control-flow checks

- `RUF060` (`in-empty-collection`), later extended to recursive
  empty-collection checking, `PLC0207` (`missing-maxsplit-arg`), and `PTH211`,
  which replaces `os.symlink` with `Path.symlink_to` (0.11.0)
- `RUF061`, which finds calls to `pytest.raises`, `pytest.warns`, and
  `pytest.deprecated_call` not used as context managers (0.12.0)
- `PLR1708` (`stop-iteration-return`) and `ISC004` (implicit string
  concatenation inside collections) (0.14.0)
- `PLR1712` (swap via a temporary), `RUF050` (unnecessary `if`), `RUF070`
  (assignment immediately before `yield`), `RUF071`
  (`os.path.commonprefix`), `RUF072` (useless `finally`), `RUF075` (fallible
  context manager), and `ASYNC119` (yield in an async-generator context
  manager) (0.15.0)

## Preview diagnostics and module-policy checks

- `RUF102` (`invalid-rule-code`) (0.11.0)
- `RUF067` (non-empty initialization modules) and `RUF068` (duplicate entries
  in `__all__`) (0.14.0)
- `RUF069` (float equality), `RUF073` (percent-formatting an f-string),
  `PLW0717` (too many `try` statements), and `B043` (constant-name
  `delattr`) (0.15.0)

Example collection and export findings:

```python
items = ["left" "right"]  # ISC004
__all__ = ["run", "run"]  # RUF068
```

## Preview Airflow checks

Airflow preview coverage includes (0.15.0):

- `AIR321` for Airflow 3.1 imports
- `AIR003` and `AIR304` for parse-time or runtime-varying DAG values
- `AIR201` for XCom pulls in template strings
- `AIR004` for branch tasks used as short-circuit tasks
- `AIR202` for implicit multiple outputs
