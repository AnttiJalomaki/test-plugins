# Suppressions and autofixes

Use this reference when migrating suppression syntax, generating ignores, or
deciding whether fixes can run unattended.

## Suppression parsing and diagnostics

File-level and inline suppression comments use the same robust parser as of
0.10.0-guide. It recognizes more valid comments, but malformed legacy forms
that happened to work can now produce errors.

`PGH004` stably detects blanket file-level `noqa`, not just blanket inline
comments. `RUF100` detects unused file-level `noqa` directives that name
specific codes (0.11.0).

Ruff treats `ty:` comments as pragma comments (0.12.0).

Preview in 0.14.0 validates `ruff:disable` and `ruff:enable` range controls,
reports invalid or unmatched comments, consolidates diagnostics for a matched
pair, and lets `RUF100` report unused ranges. `ERA001` ignores the control
comments. `--add-noqa` can attach a reason to a generated suppression.

Block `ruff:disable` / `ruff:enable` suppressions became stable in 0.15.0.
Preview added `#ruff:file-ignore` at file scope, `#ruff:ignore` at logical-line
scope, nested logical-line suppression, human-readable rule names in
suppressions and selectors, and `--add-ignore` to insert the new comments.

Preview migration rules in 0.15.0 are:

- `RUF105`: migrate `noqa` to `ruff:ignore`;
- `RUF106`: migrate rule codes to names inside `ruff:ignore`; and
- `RUF201`: migrate configuration selectors to human-readable names.

## Stable `ruff: ignore` syntax

Ruff 0.16.0 supports a `ruff: ignore` either at the end of the diagnostic line
or on the preceding line, analogous to `noqa`. The canonical spelling includes
a space after the colon:

```python
import math  # ruff: ignore[F401]

# ruff: ignore[F401]
import os
```

## Diagnostic spans and moved rule codes

Suppression placement and codes can require migration even if the underlying
finding is unchanged:

- `UP015` now highlights only the redundant mode argument rather than the
  whole `open` call; a prior `noqa` may need to move (0.10.0).
- `unsafe-markup-use` moved from `RUF035` to `S704`
  (0.10.0-guide).
- Preview `Optional` handling moved from `UP007` to `UP045`, and `RUF025`
  moved to `RUF037` (0.9.0). The `UP007`/`UP045` split became stable in
  0.12.0.
- Airflow codes moved and split across `AIR002`, `AIR301`, `AIR302`, `AIR311`,
  and `AIR312` in 0.11.0.
- Removed rules such as `S320`, `PD901`, and `UP038` no longer produce a
  diagnostic to suppress.

## Fixes that may add imports

With the 0.13.0-guide setting below, fixes for `TC001`, `TC002`, `TC003`,
`RUF013`, and `UP037` can insert a module-level future import. This supports
moving more imports under `TYPE_CHECKING`, using PEP 604 unions before Python
3.10, and unquoting more annotations:

```toml
[tool.ruff.lint]
future-annotations = true
```

Preview fixes for `UP006`, `UP007`, and `UP045` can also insert
`from __future__ import annotations` (0.15.0).

To forbid generated fixes from importing `typing_extensions`, set the 0.11.0
option:

```toml
[tool.ruff.lint]
typing-extensions = false
```

## Unsafe-fix matrix

### Value- and type-sensitive fixes

- `FURB171` is unsafe when its right-hand side is a string. `RUF046`
  parenthesizes the argument when removing `int` would otherwise change
  semantics (0.9.0).
- `FURB161` is unsafe except on integers and booleans; `FURB116` is unsafe
  except on numeric literals; `FURB180` is unsafe when the class has bases;
  `PIE804` is unsafe when the dictionary contains comments (0.11.0).
- `FURB163` is unsafe for `log2`, `log10`, `*args`, or when it would remove
  comments (0.12.0).
- `B905` and `B912` fixes are unsafe. `PTH104`, `PTH105`, `PTH109`, and
  `PTH115` are unsafe when the change can alter return or expression type,
  including inside compound statements. `FURB192` is always unsafe (0.14.0).
- `RUF036` is limited to typing contexts and unsafe outside typing-only code.
  `PTH101` is unsafe for a class attribute annotated as `int`. `EXE004` fixes
  and `PYI061` fixes in Python files are unsafe (0.15.0).

### Comment-removing fixes

In 0.11.0, fixes for `PLR1730`, `UP004`, `UP010`, and `UP050` are unsafe when
they delete comments.

In 0.14.0, a fix that removes comments is unsafe for `B009`, `B010`, `B013`,
`B014`, `B033`, `SIM910`, `SIM911`, `UP007`, `UP039`, `UP041`, `UP045`,
`FURB105`, `FURB116`, `FURB136`, `FURB140`, `FURB145`, `FURB154`, `FURB157`,
`FURB164`, `FURB181`, `FURB188`, `RUF019`, and `RUF020`.

In 0.15.0, `UP017`, `UP020`, `UP033`, `FURB110`, and `RUF010` fixes are unsafe
when they remove comments.

### Framework and ambiguous fixes

- Airflow autofixes cover the reorganized rules, but module-move fixes are
  unsafe (0.11.0).
- `PT006` receives no fix when `argnames` or `argvalues` are ambiguous
  (0.15.0).
- The `NPY201` fix for `np.in1d` was removed (0.15.0).

## Fixes promoted or corrected

- Fixes for `PYI041` (`redundant-numeric-union`) and `PYI016`
  (`duplicate-union-members`) became stable in 0.9.0.
- Stable fixes or improvements landed for `PLR1714`
  (`repeated-equality-comparison`), `SIM103` (`needless-bool`), and `PYI018`
  (`unused-private-type-var`) in 0.10.0.
- `E712` and `ISC003` gained fixes, while `SIM117` was enabled in preview in
  0.11.0.
- The `FURB129` (`readlines-in-for`) fix is always safe as of 0.12.0.
- The `SIM117` fix is always safe as of 0.13.0.
- `UP008` has a safe fix when it preserves comments (0.15.0).

## Safe rollout

1. Run `ruff check` without fixes and save the diagnostic set.
2. Apply safe fixes only and inspect both semantic and formatting diffs.
3. Identify selected rules represented in the unsafe matrix.
4. Apply unsafe fixes in small groups, preserving comments and checking return
   types, expression types, imports, and module moves.
5. Rerun suppression diagnostics, because a moved span, renamed code, or
   removed rule can make an ignore ineffective or unused.
