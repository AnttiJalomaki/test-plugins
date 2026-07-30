---
name: alembic-knowledge-patch
description: Alembic
version: 1.18.0
license: MIT
metadata:
  author: Nevaberry
---


# Alembic Knowledge Patch

Use this skill when upgrading Alembic, configuring migration environments,
writing operations, extending the command layer, or diagnosing autogenerate
output. Check the installed Alembic, Python, SQLAlchemy, and database versions
before applying version-sensitive advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [compatibility-and-configuration.md](references/compatibility-and-configuration.md) | Runtime and build requirements, yanked package, TOML and INI configuration, paths |
| [autogenerate-and-rendering.md](references/autogenerate-and-rendering.md) | Diff detection, operation rendering, naming conventions, dialect comparisons |
| [operations-and-ddl.md](references/operations-and-ddl.md) | Conditional DDL, column comments, inline primary and foreign keys |
| [revisions-cli-and-extensions.md](references/revisions-cli-and-extensions.md) | Revision layout, head checks, merge splicing, commands, plugins, implementations |

## Check installation prerequisites first

- Alembic 1.17 requires Python 3.10 or newer. Python 3.9 is supported by 1.15
  and 1.16, but Python 3.8 is not.
- Alembic 1.18 requires SQLAlchemy 1.4.23 or newer. Earlier 1.15-era
  installations accepted SQLAlchemy 1.4.0 or newer; SQLAlchemy 1.3 is not
  supported there.
- Do not pin Alembic 1.15.0. Its wheel omitted template files after the PEP 621
  packaging move. Use 1.15.1 or later in that series.
- Building from source requires setuptools 77.0.3 or newer.

## Update path configuration

Prefer the `path_separator` setting for INI configuration. It applies to both
`version_locations` and `prepend_sys_path`; `os` selects `os.pathsep` and is
portable across platforms.

```ini
[alembic]
path_separator = os
```

`path_separator` supersedes `version_path_separator`. Omitting the new setting
retains legacy splitting behavior but emits a deprecation warning.

Source and generation settings may instead live in `pyproject.toml`, including
local paths and post-write hooks. TOML lists avoid separator ambiguity, and
`%(here)s` is relative to the TOML file's parent. Keep database connectivity
and logging in `alembic.ini` or `env.py`. When `env.py` supplies them, the
`pyproject` initialization template can operate without `alembic.ini`.

## Guard revision identifiers and deployment state

Custom revision identifiers cannot contain `:` because it denotes revision
ranges. Replace identifiers such as `REV:1` with forms such as `REV_1`.

Use the head check in automation when merely printing the current revision is
not sufficient:

```console
alembic current --check-heads
```

The Python equivalent is `command.current(config, check_heads=True)`. It raises
`DatabaseNotAtHead` if any head is unapplied; the CLI exits nonzero.

## Add conditional schema operations

On backends that support the clauses, opt into conditional column and
constraint DDL:

```python
op.add_column(
    "account",
    sa.Column("nickname", sa.String()),
    if_not_exists=True,
)
op.drop_column("account", "legacy", if_exists=True)
op.drop_constraint(
    "uq_account_name",
    "account",
    type_="unique",
    if_exists=True,
)
```

Autogenerate `Rewriter` recipes can set the same flags so rendered migration
files carry the conditional behavior.

## Choose inline column constraints explicitly

`Column(primary_key=True)` does not make `ADD COLUMN` render an inline primary
key. Pass `inline_primary_key=True` explicitly:

```python
op.add_column(
    "account",
    sa.Column("id", sa.Integer, primary_key=True),
    inline_primary_key=True,
)
```

This preserves PostgreSQL `SERIAL` behavior while requesting inline
`PRIMARY KEY` syntax on a supporting backend.

To render a foreign key inside `ADD COLUMN` rather than as a separate
`ADD CONSTRAINT`, pass `inline_references=True`:

```python
op.add_column(
    "child",
    sa.Column(
        "parent_id",
        sa.Integer,
        sa.ForeignKey("parent.id", ondelete="CASCADE"),
        nullable=True,
    ),
    inline_references=True,
)
```

This is supported on PostgreSQL, Oracle, MySQL 5.7+, and MariaDB 10.5+.
Actions and attributes including `ON DELETE`, `ON UPDATE`, `DEFERRABLE`,
`INITIALLY`, and `MATCH` remain part of the inline reference.

## Work with corrected autogenerate output

Expect generated migrations to preserve several details that older recipes
may have worked around:

- Index expressions retain labels, allowing PostgreSQL operator classes keyed
  by a label name to render correctly.
- `UniqueConstraint.deferrable` renders as the Python booleans `True` or
  `False`, not quoted text.
- `AlterColumnOp.modified_name` set by a rewriter is honored. Rename detection
  is still not automatic.
- Embedded `ExecuteSQLOp` directives use the configured
  `alembic_module_prefix` rather than a hardcoded `op.` prefix.
- Reflected constraint and index names are wrapped with `Operations.f()` so a
  naming convention does not transform an already-final name again.
- Sequence-valued PostgreSQL arguments such as `postgresql_include` render
  `Column` objects as string column names.

When consuming `AutogenerateDiffsDetected`, use its revision context to format
or route detected diffs from a command wrapper.

## Account for dialect-specific comparison behavior

- PostgreSQL comparison ignores server-added `::regclass` casts on sequence
  defaults for non-primary-key columns, preventing recurring unchanged diffs.
- MySQL `ENUM` comparison detects values present on only one side but ignores
  ordering-only changes.
- Excluding a foreign-key target with `include_name`, or omitting its schema
  from reflection, no longer raises `NoReferencedTableError`. An
  `include_object` callback may receive a placeholder target table containing
  only its name, schema, and referenced columns.
- SQL Server supports `Operations.alter_column(comment=...)` for adding,
  changing, or deleting column comments.

## Organize revision files by date

`file_template` can contain directory separators, and Alembic creates the
directories. Enable recursive version locations so subsequent commands
discover the nested revisions:

```toml
[tool.alembic]
file_template = "%(year)d/%(month).2d/%(day).2d_%(rev)s_%(slug)s"
recursive_version_locations = true
```

## Extend commands and operations

Register application-specific CLI functions with the public command API:

```python
command_line.register_command(my_command)
```

To override an existing operation implementation, pass `replace=True`:

```python
@Operations.implementation_for(CreateTableOp, replace=True)
def create_table(operations, operation):
    ...
```

Use the `Plugin` interface for automatically loaded third-party extensions.
Plugins can register operations and implementations and add comparators with
`Plugin.add_autogenerate_comparator()`. Select comparison plugins per migration
environment through
`EnvironmentContext.configure(autogenerate_plugins=...)`. Existing add-ons do
not need to adopt plugin entry points.

## Handle nonstandard revision graphs

Create a merge revision from non-head revisions with:

```console
alembic merge --splice rev_a rev_b
```

The programmatic `command.merge()` API accepts the matching `splice`
parameter.

For the effective heads that would be represented after all upgrades, account
for `depends_on` relationships:

```python
heads = script.get_heads(consider_depends_on=True)
```

This excludes nominal heads that are dependencies of another revision.

## Respect public path API boundaries

Public command, configuration, and script APIs that accept paths also accept
`os.PathLike`. Public path-returning accessors still return strings. Do not
depend on the representation of private underscored APIs; they may return
`pathlib.Path` after the path-handling refactor.
