# Configuration, CLI, and Server

## Python targets and configuration inheritance

The `target-version` change announced around 0.10.0 was deferred: 0.10.0 does
not infer the Python version from `requires-python` when `target-version` is
unset. That inference begins in 0.11.0. Do not diagnose 0.10.0 using the later
behavior (0.10.0-guide).

Ruff accepts `py314` as a target, including preview handling for deferred
annotations and Python 3.14 unparenthesized exception tuples. Parser and
formatter support for template strings was present by 0.11.13 (0.11.0).

Syntax checks became stable in 0.12.0. With no target, version-related syntax
checks assume the latest supported Python, then 3.13, while other lint-rule
behavior still defaults to the minimum supported Python, then 3.9 (0.12.0).

Ruff later advanced its implicit default and latest baselines for Python 3.14.
Projects that need stable decisions should set their actual baseline:

```toml
[tool.ruff]
target-version = "py312"
```

Preview also accepts `py315` (0.14.0).

Ruff now resolves the entire `extend` chain before falling back to its default
Python version. A `target-version` inherited from an extended file can take
effect instead of an earlier default fallback (0.15.0).

## Rule selection and defaults

The split between `UP007` for `Union` and `UP045` for `Optional` is stable.
Explicit `select`, `ignore`, and `noqa` configuration may need to include
`UP045` (0.12.0).

Deprecated rules are no longer activated by selecting their group or prefix.
Select a still-existing deprecated rule only by exact code. `PD901` and
`UP038`, the remaining deprecated rules at that point, were removed and no
longer run (0.13.0-guide).

Preview mode expanded from the stable default of 59 rules to 412 rules starting
in 0.15.2. It was mostly a superset, but initially omitted `E401`, `E402`,
`E701`-`E703`, `E711`-`E714`, `E721`, `E731`, `E741`-`E743`, `F403`, `F405`,
`F406`, and `F722`; 0.15.6 removed a few more rules from the preview defaults.
Pin the former selection when needed (0.15.0):

```toml
[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
```

Ruff now enables 413 rules by default instead of 59. The new selection is
primarily an expansion but not a strict superset: 18 opinionated pycodestyle
and Pyflakes rules are not enabled implicitly. Configure a fixed selection
when policy must not drift (0.16.0-guide).

## Lint configuration

The `flake8-builtins` options with a `builtins-` prefix are deprecated. Replace,
for example, `builtins-allowed-modules` with `allowed-modules` (0.10.0).

`lint.flake8-builtins.strict-checking` changed its default from `true` to
`false` (0.10.0).

To prevent generated fixes from importing `typing_extensions`, set
`lint.typing-extensions = false` (0.11.0):

```toml
[tool.ruff.lint]
typing-extensions = false
```

When `lint.future-annotations` is enabled, fixes for `TC001`, `TC002`, `TC003`,
`RUF013`, and `UP037` may insert `from __future__ import annotations`. This
allows more imports to move into `TYPE_CHECKING`, PEP 604 unions before Python
3.10, and more annotation unquoting (0.13.0-guide).

Import sorting supports configurable section-heading comments (0.15.0).

The formatter gained `nested-string-quote-style` in 0.15.9 (0.15.0).

`numpy.typing as npt` is a default `flake8-import-conventions` alias, so
`ICN001` recognizes it without custom alias configuration (0.11.0).

## Suppression comments

File-level and inline suppression comments use one robust parser. It recognizes
more valid comments, but malformed forms that happened to work can now report
an error (0.10.0-guide).

`PGH004` stably checks blanket file-level `noqa`, not just blanket inline
comments. `RUF100` detects unused file-level `noqa` directives that name rule
codes (0.11.0).

Ruff treats `ty:` comments as pragma comments (0.12.0).

Block `ruff:disable` and `ruff:enable` suppressions are stable. Preview mode
adds:

- `#ruff:file-ignore` at file level;
- `#ruff:ignore` for a logical line;
- nested logical-line suppressions;
- human-readable rule names in suppressions and selectors; and
- `--add-ignore` to insert the newer comments.

Preview output, LSP hovers, and code actions prefer human-readable names.
`ruff rule` accepts them, and unknown selectors warn rather than fail.
Preview rules `RUF105`, `RUF106`, and `RUF201` migrate `noqa` to `ruff:ignore`,
codes to names in `ruff:ignore`, and configuration selectors to names
(0.15.0).

Ruff supports `ruff: ignore` either on a diagnostic line or on the preceding
line. The canonical form contains a space after the colon (0.16.0):

```python
import math  # ruff: ignore[F401]

# ruff: ignore[F401]
import os
```

## CLI behavior and output

`ruff format --exit-non-zero-on-format` writes formatting changes and exits
nonzero if it modified files (0.11.0):

```console
ruff format --exit-non-zero-on-format .
```

`ruff check --watch` respects `--output-format` and defaults to `full`
(0.15.0).

`ruff check` and `ruff format --check` now include fix diffs in human-readable
output. Update CI logs or wrappers that assumed the older text (0.16.0-guide).

`ruff format --check` accepts linter output formats, including CI annotations
such as `github` and `gitlab` (0.16.0-guide):

```console
ruff format --check --output-format github .
```

JSON consumers must allow `null`, rather than placeholder empty strings or
row-1/column-1 positions, in `filename`, `location`, `end_location`,
`fix.edits[].location`, and `fix.edits[].end_location` (0.16.0-guide).

## Server and editor integration

Server logging is controlled solely by `logLevel`, which defaults to `info`.
The LSP `trace` value no longer enables or disables logging. Code-action
requests ignore diagnostics emitted by other sources (0.9.0).

`ruff.printDebugInformation` no longer produces logging output (0.10.0).

The Ruff server can use `uv` as an alternate formatter backend (0.13.0).

## User configuration location

Ruff no longer searches
`~/Library/Application Support/ruff/ruff.toml` for user-level configuration on
macOS. Use the XDG location, normally `~/.config/ruff/ruff.toml`
(0.13.0-guide).
