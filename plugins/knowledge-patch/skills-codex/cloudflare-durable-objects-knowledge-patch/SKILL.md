---
name: cloudflare-durable-objects-knowledge-patch
description: Cloudflare Durable Objects
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Cloudflare Durable Objects

Use this skill when designing, implementing, migrating, testing, or debugging
Durable Object classes, bindings, storage, alarms, placement, or hibernating
WebSockets.

Inspect the Worker configuration, compatibility date, class exports, bindings,
and storage backend before giving advice. Treat deployment configuration and
namespace state as part of the program: code alone neither creates nor safely
removes a Durable Object namespace.

## Reference index

| Reference | Topics |
| --- | --- |
| [deployment-and-class-lifecycle.md](references/deployment-and-class-lifecycle.md) | SQLite versus legacy KV, `exports`, migrations, rename, delete, and cross-Worker transfer |
| [storage-sql-and-recovery.md](references/storage-sql-and-recovery.md) | SQL cursors, transactions, synchronous KV, PITR, `deleteAll()`, and loopback exports |
| [identity-and-placement.md](references/identity-and-placement.md) | Names, jurisdictions, ID spaces, and location hints |
| [concurrency-alarms-and-runtime.md](references/concurrency-alarms-and-runtime.md) | Gates, atomic writes, hibernation, eviction, uniqueness, alarms, and rollout safety |
| [websocket-hibernation.md](references/websocket-hibernation.md) | Hibernatable handlers, attachments, tags, auto-responses, event limits, and closing |
| [testing-and-local-development.md](references/testing-and-local-development.md) | Workers Vitest configuration, typed bindings, HTTP tests, alarms, eviction, and persistence |

## Resolve lifecycle and backend choices first

For a new class, create a SQLite-backed namespace. In legacy migrations, use
`new_sqlite_classes`; in declarative configuration, use a live `exports` entry
with `"storage": "sqlite"`.

```jsonc
{
  "exports": {
    "ChatRoom": {
      "type": "durable-object",
      "storage": "sqlite"
    }
  }
}
```

Do not mix `exports` and `migrations` in one Worker. Existing Workers may retain
ordered, tagged migrations, but deployment of `exports` makes the conversion
one-way. Preserve each existing backend exactly: `sqlite` for a namespace made
by `new_sqlite_classes`, and `legacy-kv` for one made by `new_classes`.

An exported class without configuration is ignored. Conversely, every live
entry needs matching code and a `storage` value. The `deleted`, `renamed`, and
`transferred` tombstones must not specify `storage`; the receiving
`expecting-transfer` state does require the namespace's backend.

Lifecycle operations require `wrangler deploy`. They are incompatible with
gradual deployment and `wrangler versions upload`, and rollback cannot cross a
lifecycle change.

## Make destructive lifecycle changes in stages

Before deleting a class, remove all account-wide bindings to its namespace and
remove the class from code. Keep the `deleted` tombstone until reconciliation
explicitly reports it safe to remove.

For a zero-downtime rename:

1. Deploy the new class while re-exporting it under the old name; do not change
   `exports` yet.
2. Deploy an old-name `renamed` tombstone and a live new-name entry while the
   code alias remains.
3. Remove the alias only after rollout completes.

```ts
export class NewName extends DurableObject {
  // ...
}
export { NewName as OldName };
```

For a cross-Worker declarative transfer, deploy the target's
`expecting-transfer` entry first without a self-binding. Then deploy the
source's `transferred` tombstone. Only afterward make the target entry live,
add its binding, and remove or redirect the source binding. Both Workers must
share the account and dispatch-namespace context.

## Use SQLite storage with synchronous boundaries in mind

SQLite-backed objects expose both `ctx.storage.sql` and synchronous
`ctx.storage.kv`. The SQL database is private to the object, has a 10 GB limit,
and includes FTS5, JSON, and math functions.

Materialize a SQL cursor before any `await`; iteration resumed afterward is not
a stable snapshot and may observe later writes, even writes later rolled back.

```ts
const rows = this.ctx.storage.sql
  .exec("SELECT * FROM users")
  .toArray();
await sendRows(rows);
```

Do not send `BEGIN`, `SAVEPOINT`, or other transaction statements through
`sql.exec()`. Use `transactionSync()` for synchronous SQL or KV work. Its
callback must not return a promise, its value is returned to the caller, and
throwing rolls the transaction back.

```ts
this.ctx.storage.transactionSync(() => {
  this.ctx.storage.sql.exec(
    "UPDATE counters SET value = value + 1 WHERE id = ?",
    counterId,
  );
});
```

On a SQLite-backed object, direct storage operations also participate in
`transaction()`; do not use the obsolete `txn` wrapper for this backend.

## Respect gate boundaries

Awaited Durable Object storage operations receive input-gate protection.
External I/O such as `fetch()` or R2 does not: another request may interleave
while it is awaited. Re-read a version or precondition after external I/O
before writing dependent state.

Pending storage writes hold outgoing responses and network requests behind the
output gate. Consecutive writes with no intervening `await` are coalesced into
one atomic implicit transaction.

```ts
this.ctx.storage.sql.exec(
  "UPDATE accounts SET balance = balance - ? WHERE id = ?",
  amount,
  from,
);
this.ctx.storage.sql.exec(
  "UPDATE accounts SET balance = balance + ? WHERE id = ?",
  amount,
  to,
);
return "transferred";
```

Run SQLite schema changes from the constructor under
`blockConcurrencyWhile()`, tracking versions in an ordinary table.
`PRAGMA user_version` is unavailable.

## Treat memory as disposable

There is no shutdown hook before eviction, deployment, or runtime replacement.
Persist checkpoints incrementally.

An otherwise quiescent object may hibernate after about 10 seconds, lose all
memory, and run its constructor again on the next event. Timers, unfinished
events, an awaited in-progress `fetch()`, or standard WebSocket API use prevent
hibernation. Hibernatable client WebSockets remain connected.

Active outbound `connect()` or WebSocket connections defer eviction, but each
connection does so for at most 15 minutes. When all close, the normal
70–140-second inactivity window begins.

## Make alarms and resets idempotent

Alarms are non-recurring and can be delivered more than once. Schedule the
next alarm explicitly, make the handler idempotent, and use
`AlarmInvocationInfo.retryCount` when arranging a fresh alarm before retries
are exhausted.

With compatibility date `2026-02-24` or later, `ctx.storage.deleteAll()` also
deletes the alarm on both storage backends. Earlier dates need an explicit
`deleteAlarm()` for a full reset.

## Use the hibernation WebSocket API end to end

Accept the server socket with `ctx.acceptWebSocket()` and implement class
handlers such as `webSocketMessage()` and `webSocketClose()`. Calling
`ws.accept()` selects the standard API and prevents hibernation. Only inbound
WebSockets served by the object can hibernate.

```ts
const [client, server] = Object.values(new WebSocketPair());
this.ctx.acceptWebSocket(server, ["room:42"]);
return new Response(null, { status: 101, webSocket: client });
```

Persist small per-connection state with `serializeAttachment()`. It stores a
structured-clone snapshot of at most 16,384 bytes; call it again after a
mutation. Store larger or longer-lived state in Durable Object storage.

## Test process boundaries, not only methods

Configure Workers Vitest 4 with `vitest@^4.1.0`,
`@cloudflare/vitest-pool-workers`, and the `cloudflareTest()` Vite plugin
pointing to the Wrangler configuration.

Exercise the Worker's HTTP route through `exports.default.fetch()` and call a
stub from `env` for direct object tests. Use `runDurableObjectAlarm()` for
scheduled alarms and the eviction helpers from `cloudflare:test` to verify
constructor re-entry and loss of in-memory state.

Eviction preserves durable storage, and storage for a named object persists
across tests in the same file. Use distinct IDs when tests need isolation.

## Review checklist

- Confirm the Worker compatibility date and the namespace's existing backend.
- Confirm that code, bindings, and either `exports` or `migrations` agree.
- Stage renames and transfers in the documented deployment order.
- Materialize SQL cursors before yielding and use storage transaction APIs.
- Revalidate state after non-storage I/O.
- Persist progress without relying on shutdown or in-memory lifetime.
- Make alarm handlers and mixed-version Worker/object calls tolerant of retry
  and rollout overlap.
- Test eviction, alarm execution, storage persistence, and WebSocket wake-up.
