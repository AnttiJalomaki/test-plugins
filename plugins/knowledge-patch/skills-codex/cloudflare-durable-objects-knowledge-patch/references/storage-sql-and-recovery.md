# Storage, SQL, and recovery

## Use the SQLite and synchronous KV surfaces

A SQLite-backed object's private database is available through
`ctx.storage.sql`. The backend also exposes synchronous KV operations through
`ctx.storage.kv`: `get`, `put`, `delete`, and `list` return immediately rather
than promises.

```ts
this.ctx.storage.kv.put("profile:42", { name: "Ada" });
const profile = this.ctx.storage.kv.get("profile:42");
```

`list()` returns key-value pairs in UTF-8 key order. It supports inclusive
`start`, exclusive `startAfter` and `end`, plus `prefix`, `reverse`, and
`limit`. An unbounded call loads every result into memory.

Embedded SQLite includes FTS5 and `fts5vocab`, JSON functions and operators,
and math functions.

## Consume SQL cursors before yielding

A `SqlStorageCursor` is not a stable snapshot across an `await`. Resumed
iteration may observe later mutations, including writes made in a later
implicit transaction that eventually rolls back. Materialize results
synchronously before yielding.

```ts
const rows = this.ctx.storage.sql.exec(
  "SELECT * FROM users",
).toArray();
await sendRows(rows);
```

One `exec()` call may contain semicolon-separated statements. Bindings apply
only to the last statement, and the returned cursor represents only that
statement. The cursor and its `raw()` iterator share and consume one position.
`one()` throws unless exactly one row is returned.

## Use storage transaction callbacks

`sql.exec()` rejects transaction statements such as `BEGIN TRANSACTION` and
`SAVEPOINT`. Use `transactionSync()` for synchronous SQL or KV work. Its
callback must not return a promise, its return value passes through, and an
exception rolls back the transaction.

```ts
const result = this.ctx.storage.transactionSync(() => {
  this.ctx.storage.sql.exec(
    "UPDATE counters SET value = value + 1 WHERE id = ?",
    counterId,
  );
  return "updated";
});
```

For a SQLite-backed object, direct `ctx.storage` operations, including SQL
queries, participate in `transaction()`. The older `txn` wrapper is obsolete
for this backend.

## Restore the full database with PITR

Point-in-time recovery bookmarks cover the entire SQLite database, including
values written through the KV API. A bookmark can target approximately any
time in the preceding 30 days.

`onNextSessionRestoreBookmark()` schedules an exact restore for the next
restart and returns a pre-restore bookmark that can undo it. Bookmark strings
compare chronologically using ordinary lexical comparison.

```ts
const DAY_MS = 24 * 60 * 60 * 1000;
const target = await this.ctx.storage.getBookmarkForTime(
  Date.now() - 2 * DAY_MS,
);
const undo = await this.ctx.storage.onNextSessionRestoreBookmark(target);
this.ctx.abort("restore requested");
```

PITR is unavailable in local development. `ctx.abort()` forcibly resets the
object and logs an uncatchable error carrying its optional message; it is also
unavailable under `wrangler dev`.

## Reset storage and alarms deliberately

With compatibility date `2026-02-24` or later, `ctx.storage.deleteAll()`
deletes both stored data and the alarm on KV- and SQLite-backed objects. A
separate `deleteAlarm()` is unnecessary at those dates. Earlier compatibility
dates still require it for a complete reset.

## Call the Worker's own exports through the context

`ctx.exports` contains loopback bindings to the Worker's top-level exports and
has the same semantics as `ExecutionContext.ctx.exports`. Do not confuse this
runtime property with the deployment-time class lifecycle map also named
`exports`.
