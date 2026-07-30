# Data, CLI, and testing

## Use the current data-table surface

The typed relational toolkit includes adapters for MySQL, PostgreSQL, and
SQLite. Import adapter-specific functionality from domain subpaths such as
`remix/data-table/postgres`.

The SQL and query API changed:

- `SqlStatement`, `sql`, and `rawSql` moved from `remix/data-table/sql` to
  `remix/data-table`.
- `Database` is a constructible runtime class.
- `QueryBuilder` is removed; use `Query` and `query`.
- `db.exec(...)` accepts raw SQL or `Query` values.
- Unbound terminal methods such as `first()`, `count()`, `insert()`, and
  `update()` return `Query` objects.

Check whether a terminal operation is bound to a database before assuming it
executes immediately.

## Use action-oriented generated structure

The CLI commands `remix doctor` and `remix routes`, as well as generated
applications, expect action files under `app/actions`.

- Root actions belong in `app/actions/controller.tsx`.
- Nested maps require explicit `router.map(...)` calls in `app/router.ts`.
- Controller middleware applies only to the controller's direct actions.

The `app/controllers` convention belongs to earlier beta code.

## Run the current CLI

The package exposes the `remix` binary and the `remix/cli` entrypoint. Its CLI
metadata requires Node.js 24.3.0 or later.

The separate `remix-test` binary has been removed. Invoke the test command
through:

```sh
remix test
```

## Use the Node TypeScript/JSX loader

`remix test` uses the internal `node-tsx` loader. Run application entrypoints
with:

```sh
node --import remix/node-tsx app.tsx
```

The loader retains Node module resolution and handles TypeScript syntax that
requires emitted JavaScript, including:

- Enums.
- Runtime namespaces.
- Parameter properties.

This loader is the planned exception to Remix 3's no-required-compilation
runtime direction.

## Make tests cancellation-aware

Tests and hooks accept `{ timeout, signal }`. The test context's `t.signal`
aborts when a timeout fires.

Use that signal to cancel pending work and avoid state changes or resource
activity after the test has timed out.

## Do not depend on the removed native server

Early betas exposed `remix/node-serve` and `@remix-run/node-serve`. Beta 2
removed both because their native transport dependency was unavailable in
npm-compatible packaging.

The package remains absent through beta 5, with restoration deferred to a
later beta. Use the available Fetch server integration rather than adding an
import for either removed entrypoint.

