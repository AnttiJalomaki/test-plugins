---
name: ruff-knowledge-patch
description: Ruff
version: 0.16.0
license: MIT
metadata:
  author: Nevaberry
---


# Ruff Knowledge Patch

Use this skill when configuring, upgrading, integrating, or debugging Ruff,
especially when the result depends on current lint rules, formatter style,
Python syntax support, suppressions, autofix safety, CLI output, or server
behavior.

## Working method

1. Determine the installed Ruff version from the project manifest, lockfile,
   CI image, pre-commit revision, or `ruff --version`.
2. Read the project configuration before recommending changes. Check
   `pyproject.toml`, `ruff.toml`, or `.ruff.toml`, including every file reached
   through `extend`.
3. Establish the intended Python version explicitly. An implicit
   `target-version` can affect syntax validation, lint selection, import
   rewrites, and formatting.
4. Separate stable behavior from preview behavior. Do not recommend a preview
   rule or formatter style without checking whether preview is enabled for the
   relevant command.
5. Preserve explicit `select`, `ignore`, per-file ignores, suppression
   comments, output formats, and unsafe-fix policy during migrations.
6. Run the narrow command first, inspect its diff and diagnostics, and only
   then apply broader formatting or fixes.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migration-and-configuration.md](references/migration-and-configuration.md) | Default-selection changes, deprecations, renamed rules and options, target-version inference, configuration inheritance |
| [references/formatting-and-syntax.md](references/formatting-and-syntax.md) | Stable and preview formatter styles, Markdown, Python syntax validation, language-version support |
| [references/lint-rules.md](references/lint-rules.md) | Newly stable and preview rules, expanded rule coverage, stabilized diagnostics |
| [references/suppressions-and-fixes.md](references/suppressions-and-fixes.md) | `noqa`, `ruff: ignore`, range suppressions, future imports, fix availability and safety |
| [references/cli-server-analysis-and-distribution.md](references/cli-server-analysis-and-distribution.md) | Check/format output, language server, dependency analysis, discovery, containers, binaries, source builds |

## Breaking changes first

### Pin lint selection when defaults matter

The default selection can change substantially across upgrades. Ruff now
enables 413 rules by default rather than 59, but the expanded set is not a
strict superset: 18 opinionated pycodestyle and Pyflakes rules are no longer
implicit. Preserve a known policy with an explicit selection:

```toml
[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
```

Earlier preview releases also broadened their defaults to hundreds of rules
while omitting several formerly implicit checks. Treat `preview = true` as a
lint-selection change as well as a feature gate.

### Expect Markdown writes

`ruff format` formats labeled Python code blocks in Markdown by default.
Include Markdown in upgrade diffs and CI checks. Quarto files no longer receive
an implicit special case; configure their extension when needed:

```toml
[tool.ruff]
extension = { qmd = "markdown" }
```

### Pin the Python target

Ruff's implicit default and latest Python baselines have advanced. Syntax
validation can use a different fallback from ordinary lint-rule application,
and inherited configuration can now supply the target after the complete
`extend` chain is resolved. Pin project reality:

```toml
[tool.ruff]
target-version = "py312"
```

The once-announced `requires-python` inference change was not present in
0.10.0; it first shipped in 0.11.0.

### Update machine-readable output consumers

JSON diagnostic fields including `filename`, `location`, `end_location`, and
fix-edit locations may be null. CI wrappers must also account for fix diffs in
human-readable `ruff check` and `ruff format --check` output.

## Deprecations and removals

- Replace `RUF035` selections and suppressions with `S704` for
  `unsafe-markup-use`.
- `RUF025` moved to `RUF037`; preview split `Optional` handling from `UP007`
  into `UP045`, and the split later became stable.
- Airflow preview codes were reorganized: the old `AIR301` became `AIR002`,
  `AIR302` became `AIR301`, and `AIR303` became `AIR302`; checks also split
  into `AIR311` and `AIR312`, with some moving back to `AIR302`.
- `S320` was deprecated and then removed. `PD901` and `UP038` were deprecated
  and later removed. Deprecated rules no longer activate through a group or
  prefix and, while present, require exact-code selection.
- Replace `builtins-allowed-modules` and other `builtins-`-prefixed
  `flake8-builtins` settings with their unprefixed names.
- On macOS, use the XDG configuration location, normally
  `~/.config/ruff/ruff.toml`; the Application Support fallback was removed.
- The `NPY201` fix for `np.in1d` no longer exists.

## Suppression quick reference

Canonical logical-line suppression syntax is:

```python
import math  # ruff: ignore[F401]

# ruff: ignore[F401]
import os
```

Keep the space in `ruff: ignore`. Block `ruff:disable` / `ruff:enable`
suppression is stable. Human-readable rule names, nested logical-line
suppression, `#ruff:file-ignore`, `#ruff:ignore`, and `--add-ignore` remain
preview-sensitive features where introduced.

File-level and inline `noqa` comments share one robust parser. Malformed legacy
forms can now error. `PGH004` detects blanket file-level `noqa`, and `RUF100`
detects unused file-level or range suppressions. Because some diagnostic spans
moved—for example `UP015` now highlights only the mode argument—an existing
`noqa` may need repositioning.

## Autofix safety quick reference

Do not equate “a fix exists” with “safe to apply.” Important unsafe cases
include:

- fixes that remove comments across many bugbear, simplify, upgrade, pathlib,
  refurb, and Ruff rules;
- `FURB171` with a string right-hand side, `FURB161` except for integers and
  booleans, and `FURB116` except for numeric literals;
- `FURB180` on a class with bases, `PIE804` on dictionaries with comments,
  and Airflow module moves;
- `B905`, `B912`, and `FURB192`; pathlib fixes that can change return or
  expression type; and `RUF036` outside typing-only code;
- future-annotation rewrites that insert a module-level
  `from __future__ import annotations`.

Conversely, the `FURB129` fix and the stabilized `SIM117` fix are always safe;
`UP008` is safe when it preserves comments. Read the full safety matrix before
enabling `--unsafe-fixes` or applying fixes repository-wide.

## Formatter quick reference

Upgrades can intentionally produce formatter-only diffs. The stable style now
includes fluent method-chain formatting, updated lambda layout, Python 3.14
exception-type formatting, triple-quoted-string spacing, blank lines before
decorated stub classes, and the `nested-string-quote-style` option. Earlier
stable changes affected f-string expressions and quotes, implicit string
concatenation, match cases, asserts, return annotations, comprehensions,
single-manager `with` statements, and dynamic docstring code-block widths.

For Python 3.13.4 compatibility, Ruff avoids a multiline f-string break after
a format specifier that would be a syntax error. Preview also validates
compile-time and version-gated syntax; regular checks later inherited that
validation.

## High-value current capabilities

- `ruff format --exit-non-zero-on-format` writes changes and exits nonzero if
  it modified files.
- `ruff format --check --output-format github .` emits a CI annotation format;
  linter output formats are accepted by format checks.
- `ruff analyze graph` can resolve dependencies from a supplied virtual
  environment, skip imports under `TYPE_CHECKING`, analyze notebooks, and use
  configured `src` directories.
- Set `lint.typing-extensions = false` to prevent generated fixes from adding
  imports from `typing_extensions`.
- Set `lint.future-annotations = true` when selected typing fixes may insert a
  future import and use that expanded rewrite space deliberately.
- `py314` is a supported target. Python 3.15 syntax, lazy imports, and related
  rules require the appropriate preview configuration.
- The server can use `uv` as a formatter backend. Logging is controlled only
  by `logLevel` (default `info`), not LSP `trace`.

## Verification checklist

- Confirm `ruff --version` matches the configuration and CI image being
  evaluated.
- Run `ruff check` without fixes and review newly selected, renamed, removed,
  or stabilized rules.
- Run `ruff format --check` over Python and Markdown inputs; inspect style-only
  changes separately from lint fixes.
- If applying fixes, start without unsafe fixes, then review every unsafe class
  relevant to the selected rules.
- Validate suppression comments after rule-code migrations and diagnostic-span
  changes.
- Exercise JSON parsers, annotation output, watch mode, and editor code actions
  when automation depends on their exact shape.
- Recheck import classification and dependency graphs against the configured
  project roots, `src` directories, notebooks, and virtual environment.
