# Operations and DDL

## Conditional column and constraint DDL

Since 1.16.0, column and constraint operations can request conditional DDL on
backends that support it:

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

- `Operations.add_column(..., if_not_exists=True)` renders `IF NOT EXISTS`.
- `Operations.drop_column(..., if_exists=True)` renders `IF EXISTS`.
- `Operations.drop_constraint(..., if_exists=True)` renders `IF EXISTS`.

Custom autogenerate `Rewriter` recipes may set these flags on operation
directives so generated migrations retain the requested clauses.

## SQL Server column comments

As of 1.18.0, `Operations.alter_column(comment=...)` emits Microsoft SQL Server
DDL that can add, change, or delete a column comment:

```python
op.alter_column("account", "name", comment="Display name")
```

This operation no longer fails with `UnsupportedCompilationError` merely
because the target dialect is SQL Server.

## Opt in to inline primary keys

Since 1.18.0, `Operations.add_column(..., inline_primary_key=True)` renders
`PRIMARY KEY` within `ADD COLUMN`:

```python
op.add_column(
    "account",
    sa.Column("id", sa.Integer, primary_key=True),
    inline_primary_key=True,
)
```

Setting `Column(primary_key=True)` alone does not opt in. Explicit selection
preserves PostgreSQL `SERIAL` behavior while permitting inline primary-key
syntax when the backend supports it.

## Opt in to inline foreign-key references

Since 1.18.0, `Operations.add_column(..., inline_references=True)` can render
`REFERENCES` inside `ADD COLUMN` rather than issue a separate
`ADD CONSTRAINT`:

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

Inline references are supported on PostgreSQL, Oracle, MySQL 5.7 and newer,
and MariaDB 10.5 and newer. Rendering carries through `ON DELETE`, `ON UPDATE`,
`DEFERRABLE`, `INITIALLY`, and `MATCH` attributes.
