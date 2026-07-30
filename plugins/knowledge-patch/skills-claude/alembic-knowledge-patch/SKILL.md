---
name: alembic-knowledge-patch
description: Alembic
version: 1.18.0
license: MIT
metadata:
  author: Nevaberry
---


# Alembic Knowledge Patch

## When to use this skill

Load this skill when an Alembic task involves any of the following:

- upgrading Python, SQLAlchemy, Alembic, or source-build tooling;
- configuring Alembic with `pyproject.toml` or cross-platform paths;
- diagnosing autogenerate output, custom comparators, or rewriters;
- writing conditional, inline, or backend-specific migration DDL;
- extending the CLI or operation implementation registry;
- creating nested, merge, or dependency-aware revision layouts;
- checking whether every database head has been applied.

Project manifests, configuration, migration scripts, and observed behavior remain
authoritative. Apply the notes here only when they match the installed Alembic
version and database backend.

## Reference index

| Reference | Topics |
| --- | --- |
| [compatibility-and-configuration.md](references/compatibility-and-configuration.md) | Python and SQLAlchemy floors, yanked package, `pyproject.toml`, path parsing, `PathLike`, source builds |
| [autogenerate-and-rendering.md](references/autogenerate-and-rendering.md) | Expression indexes, constraints, rewriters, naming conventions, server defaults, filtered objects, dialect arguments, MySQL `ENUM` |
| [operations-and-ddl.md](references/operations-and-ddl.md) | Conditional DDL, SQL Server comments, replacement implementations, inline primary and foreign keys |
| [revisions-cli-and-extensions.md](references/revisions-cli-and-extensions.md) | Custom commands, revision IDs, head checks, nested revisions, plugins, splice merges, dependency-aware heads |

## Breaking compatibility checks

### Runtime requirements

- Alembic 1.15 requires Python 3.9 or newer and SQLAlchemy 1.4 or
  newer. It no longer supports Python 3.8 or SQLAlchemy 1.3.
- Alembic 1.17 raises the Python floor to 3.10, so update runtime and CI
  environments before upgrading from a Python 3.9 deployment.
- Alembic 1.18 raises the SQLAlchemy floor to 1.4.23. Check direct and
  transitive pins before resolving the upgrade.

### Avoid the yanked package

Do not install Alembic 1.15.0. Its wheel omitted the migration template files
after the packaging move to PEP 621. Use 1.15.1 or later in that series, where
the wheel was corrected.

### Revise path and revision-ID assumptions

Use `path_separator = os` for cross-platform splitting of
`version_locations` and `prepend_sys_path`. The setting supersedes
`version_path_separator`. Omitting it retains legacy splitting and emits a
deprecation warning.

Custom revision IDs cannot contain `:` as of Alembic 1.17 because the colon is
reserved for revision ranges. Replace IDs such as `REV:1` with `REV_1`.

### Source-build requirement

Building Alembic from source after the PEP 639 metadata change requires
setuptools 77.0.3 or newer. This does not describe the runtime dependency.

## Configuration quick reference

### Split source and deployment settings deliberately

Alembic can store source-code and generation settings in `pyproject.toml`,
including local paths and post-write hooks. TOML lists avoid separator
ambiguity for `version_locations` and `prepend_sys_path`, and `%(here)s`
resolves relative to the TOML file's parent.

Keep database connectivity and logging in `alembic.ini` or `env.py`. When
`env.py` supplies those settings, the `pyproject` initialization template can
operate without an `alembic.ini` file.

```ini
[alembic]
path_separator = os
```

### Use path objects at public boundaries

Public command, configuration, and script APIs that accept string paths also
accept `os.PathLike` values. Public path accessors still return strings. Do not
rely on private underscored APIs returning strings; the path refactor allows
those APIs to return `pathlib.Path` objects.

## Migration-operation quick reference

### Conditional column and constraint DDL

On supporting backends, opt into guarded DDL directly on the operation:

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

Custom autogenerate `Rewriter` recipes can place the same flags into generated
migrations.

### Inline primary keys require explicit opt-in

`Column(primary_key=True)` alone does not make `ADD COLUMN` emit an inline
primary key. Pass `inline_primary_key=True` when the backend supports the
syntax:

```python
op.add_column(
    "account",
    sa.Column("id", sa.Integer, primary_key=True),
    inline_primary_key=True,
)
```

The explicit flag preserves PostgreSQL `SERIAL` behavior while permitting
inline `PRIMARY KEY` rendering.

### Inline foreign-key references

Pass `inline_references=True` to render `REFERENCES` inside `ADD COLUMN`
instead of emitting a separate `ADD CONSTRAINT` on PostgreSQL, Oracle, MySQL
5.7+, and MariaDB 10.5+:

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

Inline rendering retains foreign-key actions and attributes such as
`ON DELETE`, `ON UPDATE`, `DEFERRABLE`, `INITIALLY`, and `MATCH`.

### SQL Server column comments

`op.alter_column(..., comment=...)` can add, update, or delete SQL Server
column comments:

```python
op.alter_column("account", "name", comment="Display name")
```

## Autogenerate quick reference

### Renames still require a rewriter

Alembic does not detect column renames by itself. A rewriter may set
`AlterColumnOp.modified_name`; as of 1.15.2, operation rendering honors that
attribute and emits the renamed column correctly.

### Preserve finalized names

Reflected constraint and index names are rendered through `Operations.f()` so
an active naming convention does not transform a final database name again:

```python
op.drop_constraint(op.f("uq_account_name"), "account", type_="unique")
```

This is especially important for conventions using `%(constraint_name)s`.

### Account for filtered foreign-key targets

Foreign keys whose target table is excluded by `include_name` or absent from
reflected schemas no longer cause `NoReferencedTableError`. An
`include_object` callback can receive a placeholder target table carrying only
its name, schema, and referenced columns, so callback code must not assume a
fully reflected table.

### Select comparison plugins per environment

Use `EnvironmentContext.configure(autogenerate_plugins=...)` to choose built-in
and extension comparison plugins for an environment. Existing add-ons do not
need to adopt plugin entry points to keep working.

## Revision and CLI quick reference

### Verify all heads are applied

Use the CLI head check in deployment validation:

```console
alembic current --check-heads
```

The command exits nonzero if any head is missing. Programmatic callers can use
`command.current(..., check_heads=True)` and handle `DatabaseNotAtHead`.

### Create date-organized revision paths safely

`file_template` can contain directory separators and creates directories
automatically. Enable recursive version locations so later commands discover
the nested revision files:

```toml
[tool.alembic]
file_template = "%(year)d/%(month).2d/%(day).2d_%(rev)s_%(slug)s"
recursive_version_locations = true
```

### Splice a merge from non-head revisions

Use the merge command's splice option when the selected revisions are not
heads:

```console
alembic merge --splice rev_a rev_b
```

Programmatic callers can pass the corresponding `splice` argument to
`command.merge()`.

### Resolve effective heads through dependencies

Call `ScriptDirectory.get_heads(consider_depends_on=True)` when nominal heads
linked through `depends_on` should collapse to the effective heads represented
in `alembic_version` after all upgrades:

```python
heads = script.get_heads(consider_depends_on=True)
```
