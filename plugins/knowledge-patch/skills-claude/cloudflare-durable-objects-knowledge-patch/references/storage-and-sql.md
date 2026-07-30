# Storage and SQL

## Reset storage and alarms together

At compatibility date `2026-02-24` or later, `ctx.storage.deleteAll()` deletes
the object's alarm as well as its stored data on both KV- and SQLite-backed
objects. A complete reset no longer needs a separate `deleteAlarm()`.

```js
await this.ctx.storage.deleteAll();
```

## Exhaust SQL cursors before awaiting

A `SqlStorageCursor` is not a stable snapshot across an `await`. Resumed
iteration may see later mutations, including writes from a later implicit
transaction that eventually rolls back. Materialize the rows synchronously
before yielding.

```ts
const rows = this.ctx.storage.sql.exec(
  "SELECT * FROM users",
).toArray();
await fetch("https://example.com", {
  method: "POST",
  body: JSON.stringify(rows),
});
```

## Understand `exec()` cursor semantics

One `exec()` call may contain semicolon-separated statements, but bindings
apply only to the last statement and the returned cursor represents only that
statement.

The object cursor and its `raw()` iterator consume the same cursor position.
Calling `one()` throws unless exactly one row is returned.

## Use storage transaction callbacks

`sql.exec()` rejects transaction statements such as `BEGIN TRANSACTION` and
`SAVEPOINT`. Use `transactionSync()` for synchronous SQL or synchronous KV
work. Its callback must not return a promise. Its return value passes through,
and an exception rolls the transaction back.

```ts
this.ctx.storage.transactionSync(() => {
  this.ctx.storage.sql.exec(
    "UPDATE counters SET value = value + 1 WHERE id = ?",
    counterId,
  );
});
```

For SQLite-backed objects, direct `ctx.storage` operations, including SQL
queries, participate in `transaction()`. The older `txn` wrapper is obsolete
for that backend.

## Use embedded SQLite capabilities

The private SQLite database includes selected extensions:

- FTS5, including `fts5vocab`
- JSON functions and operators
- Math functions

These features can be called directly from SQL in the object.

## Restore with point-in-time recovery

PITR bookmarks cover the complete SQLite database, including values written
through the KV API. A bookmark can target approximately any time during the
preceding 30 days.

`onNextSessionRestoreBookmark()` schedules an exact restore for the next
restart. It returns a pre-restore bookmark that can undo the restore.
Bookmarks compare chronologically with ordinary lexical string comparison.

```ts
const DAY_MS = 24 * 60 * 60 * 1000;
const target = await this.ctx.storage.getBookmarkForTime(
  Date.now() - 2 * DAY_MS,
);
await this.ctx.storage.onNextSessionRestoreBookmark(target);
this.ctx.abort("restore requested");
```

PITR is unavailable in local development. `ctx.abort()` forcibly resets the
object and logs an uncatchable error carrying its optional message; it is also
unavailable under `wrangler dev`.

## Use synchronous KV on SQLite-backed objects

`ctx.storage.kv` provides immediate `get`, `put`, `delete`, and `list`
operations without promises.

```ts
this.ctx.storage.kv.put("profile:42", { name: "Ada" });
const profile = this.ctx.storage.kv.get("profile:42");
```

`list()` returns key-value pairs in UTF-8 key order. It supports inclusive
`start`, exclusive `startAfter` and `end`, plus `prefix`, `reverse`, and
`limit`. An unbounded list loads all matching data into memory.

## Distinguish runtime loopback exports

`ctx.exports` on Durable Object state contains loopback bindings to the
Worker's own top-level exports. It has the same semantics as
`ExecutionContext.ctx.exports`.

This runtime property is distinct from the deployment-time Durable Object
class lifecycle map that is also named `exports`.
