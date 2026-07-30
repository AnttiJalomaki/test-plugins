# Migration and configuration

Use this reference when upgrading Ruff, preserving an existing lint policy, or
changing configuration files. The sections are organized by migration task,
not release chronology.

## Default lint selection

The preview defaults expanded to 412 rules in 0.15.2, versus the stable
default's 59. That set was mostly a superset, but initially omitted `E401`,
`E402`, `E701`–`E703`, `E711`–`E714`, `E721`, `E731`, `E741`–`E743`, `F403`,
`F405`, `F406`, and `F722`; 0.15.6 removed several more from the preview
defaults. Pin the former selection if preview must not broaden policy:

```toml
[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
```

The stable default expanded to 413 rules in 0.16.0-guide. It is not a strict
superset of the old 59-rule default: 18 opinionated pycodestyle and Pyflakes
rules are no longer implicit. Explicit `select` is the migration-safe choice.

## Rule-code migrations, deprecations, and removals

- `A005` is now named `stdlib-module-shadowing`, replacing
  `builtin-module-shadowing` (0.9.0).
- Preview split `UP007`: it handles `Union`, while `UP045` handles `Optional`.
  The split became stable in 0.12.0, so update explicit `select`, `ignore`, and
  `noqa` entries.
- Preview rule code `RUF025` moved to `RUF037` (0.9.0).
- `unsafe-markup-use` moved from `RUF035` to `S704`; update selections,
  ignores, and suppressions (0.10.0-guide).
- `UP038` (`non-pep604-isinstance`) and `S320`
  (`suspicious-xmle-tree-usage`) were deprecated in 0.10.0. `S320` was removed
  in 0.12.0. `PD901` (`pandas-df-variable-name`) was then deprecated.
- Deprecated rules stopped activating via a selected group or prefix; select
  one by exact code while it still exists. `PD901` and `UP038`, the remaining
  deprecated rules, were removed in 0.13.0-guide.
- The Airflow 3 preview codes changed in 0.11.0: the former `AIR301` moved to
  `AIR002`, `AIR302` moved to `AIR301`, and `AIR303` moved to `AIR302`.
  Additional checks split into `AIR311` and `AIR312`, and some `AIR312` checks
  later returned to `AIR302`. Update explicit selectors, ignores, and `noqa`
  codes. Fixes exist, but module-move fixes are unsafe.
- The `NPY201` fix for `np.in1d` was removed (0.15.0).

## Target Python version

The announced inference of Python version from `requires-python`, when
`target-version` is unset, did not ship in 0.10.0; it first shipped in 0.11.0
(0.10.0-guide).

Syntax validation introduced in preview later became part of regular checks.
When `target-version` was unset in 0.12.0, version-related syntax checks assumed
the latest supported Python, 3.13, while other behavior such as lint-rule
application still defaulted to the minimum supported Python, 3.9.

Ruff advanced its implicit default and latest baselines for Python 3.14 in
0.14.0. Pin the real project version to keep syntax, rule, and formatting
decisions stable:

```toml
[tool.ruff]
target-version = "py312"
```

Preview accepts `py315`:

```toml
[tool.ruff]
preview = true
target-version = "py315"
```

Starting in 0.15.0, Ruff resolves the entire chain of `extend`ed configuration
files before falling back to its default Python version. A target inherited
from an extended file can therefore win instead of an earlier default fallback.

## Type-checking and generated imports

Any local variable named `TYPE_CHECKING` is recognized as a type-checking guard.
Legacy `if 0:` and `if False:` guards are no longer recognized; use a local
`TYPE_CHECKING` variable (0.10.0-guide).

Prevent generated fixes from introducing `typing_extensions` imports with the
0.11.0 setting:

```toml
[tool.ruff.lint]
typing-extensions = false
```

With `lint.future-annotations` enabled, fixes for `TC001`, `TC002`, `TC003`,
`RUF013`, and `UP037` can insert `from __future__ import annotations`. This
lets them move more imports under `TYPE_CHECKING`, use PEP 604 unions before
Python 3.10, and unquote more annotations (0.13.0-guide):

```toml
[tool.ruff.lint]
future-annotations = true
```

Preview fixes for `UP006`, `UP007`, and `UP045` can also insert that future
import (0.15.0). Bandit import checks `S401`–`S415` allow imports inside
`TYPE_CHECKING`, while `TC001`–`TC003` avoid strict behavior when
`lint.future-annotations` is enabled.

## Plugin and rule-family settings

- `lint.flake8-builtins.strict-checking` defaults to `false`, not `true`
  (0.10.0).
- `builtins-`-prefixed `flake8-builtins` options are deprecated. For example,
  replace `builtins-allowed-modules` with `allowed-modules` (0.10.0).
- `numpy.typing as npt` is a default `flake8-import-conventions` alias, so
  `ICN001` recognizes it without custom configuration (0.11.0).
- Preview `RUF066` honors decorators listed under
  `lint.pydocstyle.property-decorators` (0.14.0).
- Import sorting supports configurable section-heading comments (0.15.0).
- Human-readable rule names can be used in preview suppressions and selectors.
  Preview output, LSP hovers, and code actions prefer them; `ruff rule` accepts
  them; unknown selectors warn rather than fail. Preview migration rules
  `RUF105`, `RUF106`, and `RUF201` convert `noqa` to `ruff:ignore`, codes to
  names inside `ruff:ignore`, and configuration selectors to names
  respectively (0.15.0).

## File types, discovery, and project roots

The isort implementation compares a module's full path with the configured
project root or roots when classifying first-party imports. Nested modules can
therefore move between import sections (0.11.0).

Preview input discovery includes `*.pyw` by default (0.14.0).

Preview formatting handles labeled Python blocks in Markdown, including
`pycon` and Quarto language markers, while leaving unlabeled blocks unchanged.
Markdown discovery became a preview default in 0.15.5. `.qmd` no longer has an
implicit special case and must be mapped when desired:

```toml
[tool.ruff]
extension = { qmd = "markdown" }
```

Configured extensions participate in discovery, code-block language choice,
and later server handling (0.15.0). Markdown formatting became a default,
non-preview behavior in 0.16.0-guide.

## User-level configuration

Ruff no longer searches `~/Library/Application Support/ruff/ruff.toml` on
macOS. Put user-level configuration in the XDG location, normally
`~/.config/ruff/ruff.toml` (0.13.0-guide).
