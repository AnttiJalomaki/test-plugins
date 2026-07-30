# Operations and DDL

## Conditional column and constraint DDL

The following operation flags render guarded DDL on backends that support the
corresponding clause: (since 1.16.0)

- `Operations.add_column(..., if_not_exists=True)` renders `IF NOT EXISTS`;
- `Operations.drop_column(..., if_exists=True)` renders `IF EXISTS`;
- `Operations.drop_constraint(..., if_exists=True)` renders `IF EXISTS`.

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

Support remains backend-dependent. A custom autogenerate `Rewriter` can set
the same flags so generated migrations retain the guarded behavior.

## Replacing registered operation implementations

`Operations.implementation_for(..., replace=True)` allows an extension to
replace an existing operation implementation, including the implementation for
an operation such as `CreateTableOp`. Without `replace=True`, registration is
limited to operation types that do not already have an implementation. (since
1.17.0)

Use replacement deliberately: it changes how an existing Alembic operation is
executed for the extension's registration context.

## SQL Server column comments

`Operations.alter_column(comment=...)` emits SQL Server DDL to add, update, or
delete a column comment instead of raising `UnsupportedCompilationError`.
(since 1.18.0)

```python
op.alter_column("account", "name", comment="Display name")
```

Use the same operation for setting a new comment, changing an existing comment,
or deleting it through the appropriate `comment` value.

## Inline primary keys on added columns

`Operations.add_column(..., inline_primary_key=True)` renders `PRIMARY KEY`
inside `ADD COLUMN`. A `Column` with `primary_key=True` does not opt in by
itself. The separate flag preserves PostgreSQL `SERIAL` behavior while allowing
inline primary-key syntax on a backend that supports it. (since 1.18.0)

```python
op.add_column(
    "account",
    sa.Column("id", sa.Integer, primary_key=True),
    inline_primary_key=True,
)
```

## Inline foreign-key references

`Operations.add_column(..., inline_references=True)` renders the foreign-key
`REFERENCES` clause inside `ADD COLUMN` rather than emitting a separate
`ADD CONSTRAINT`. The inline form is supported for PostgreSQL, Oracle, MySQL
5.7 and newer, and MariaDB 10.5 and newer. (since 1.18.0)

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

Inline rendering includes foreign-key actions and attributes such as:

- `ON DELETE`;
- `ON UPDATE`;
- `DEFERRABLE`;
- `INITIALLY`;
- `MATCH`.
