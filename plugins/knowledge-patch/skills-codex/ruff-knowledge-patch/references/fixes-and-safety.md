# Fixes and Safety

Review unsafe fixes separately. A reclassification normally reflects a real
risk of semantic change, lost comments, changed types, or broader code motion.

## Comment-preservation risks

Fixes are unsafe when they delete comments in these cases:

- `PLR1730`, `UP004`, `UP010`, and `UP050` (0.11.0)
- `B009`, `B010`, `B013`, `B014`, `B033`, `SIM910`, `SIM911`, `UP007`,
  `UP039`, `UP041`, `UP045`, `FURB105`, `FURB116`, `FURB136`, `FURB140`,
  `FURB145`, `FURB154`, `FURB157`, `FURB164`, `FURB181`, `FURB188`, `RUF019`,
  and `RUF020` (0.14.0)
- `UP017`, `UP020`, `UP033`, `FURB110`, and `RUF010` (0.15.0)

`PIE804` is unsafe when the dictionary contains comments (0.11.0).

`UP008` has a safe fix when it preserves comments (0.15.0).

## Input- and expression-sensitive fixes

- `FURB171` is unsafe when the right-hand side is a string (0.9.0).
- `RUF046` parenthesizes the argument when removing `int(...)` would otherwise
  change semantics (0.9.0).
- `FURB161` is unsafe except on integers and booleans; `FURB116` is unsafe
  except on numeric literals (0.11.0).
- `FURB163` is unsafe for `log2`, `log10`, `*args`, or if it deletes comments
  (0.12.0).
- `PTH104`, `PTH105`, `PTH109`, and `PTH115` are unsafe when they can change
  return or expression types, including in compound statements (0.14.0).
- `PTH101` is unsafe for a class attribute annotated as `int` (0.15.0).

## Class, typing, and import risks

- `FURB180` is unsafe when the class has bases (0.11.0).
- Airflow module-move fixes are unsafe, even though autofixes cover the
  reorganized Airflow rules (0.11.0).
- `UP007` and `UP045` do not offer a fix for `Optional[None]` (0.12.0).
- `RUF036` is limited to typing contexts, and its fix is unsafe outside
  typing-only code (0.15.0).
- The `EXE004` fix and `PYI061` fixes in Python files are unsafe (0.15.0).

With `lint.future-annotations = true`, fixes for `TC001`, `TC002`, `TC003`,
`RUF013`, and `UP037` may insert a module-level future import
(0.13.0-guide):

```python
from __future__ import annotations
```

In preview, fixes for `UP006`, `UP007`, and `UP045` may also insert that import
(0.15.0).

## Always-unsafe and always-safe classifications

- `B905` and `B912` are unsafe (0.14.0).
- `FURB192` is always unsafe (0.14.0).
- `FURB129` is always safe (0.12.0).
- `SIM117` is always safe (0.13.0).

## Fix availability and corrected output

- `E712` and `ISC003` gained fixes, and the `SIM117` fix became available in
  preview (0.11.0).
- `PERF403` gained a fix, while `PERF401` can replace list constructors with
  comprehensions (0.11.0).
- `SIM905` can fix `split(maxsplit=...)` without an explicit separator
  (0.15.0).
- The `NPY201` fix for `np.in1d` was removed (0.15.0).
- `PT006` receives no fix when `argnames` or `argvalues` are ambiguous
  (0.15.0).
