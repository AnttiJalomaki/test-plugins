---
name: cloudflare-durable-objects-knowledge-patch
description: Cloudflare Durable Objects
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Cloudflare Durable Objects

Use this skill when designing, implementing, migrating, or testing Cloudflare
Durable Objects. Start with the quick reference, then load the topic file that
matches the work. Treat Worker configuration, exported classes, namespace
state, compatibility dates, and tests as one system.

## Reference index

| Reference | Topics |
| --- | --- |
| [Deployment and class lifecycle](references/deployment-lifecycle.md) | SQLite adoption, `exports`, tombstones, renames, transfers, environments, legacy migrations |
| [Storage and SQL](references/storage-and-sql.md) | `deleteAll()`, SQL cursors, transactions, extensions, PITR, synchronous KV, loopback exports |
| [Identity and placement](references/identity-and-placement.md) | names, jurisdictions, ID spaces, location hints |
| [Concurrency, eviction, and alarms](references/concurrency-eviction-and-alarms.md) | hibernation, gates, atomicity, schema migration, uniqueness, shutdown, rollout compatibility, alarms, local persistence |
| [WebSocket hibernation](references/websocket-hibernation.md) | acceptance, handlers, attachments, tags, auto-responses, event limits, close handling |
| [Testing](references/testing.md) | Vitest configuration, typing, HTTP and stub calls, persistence, alarms, eviction |

## Breaking changes and deprecations

### Prefer SQLite for every new class

Declare new namespaces with `new_sqlite_classes`, or use a live declarative
`exports` entry with `"storage": "sqlite"`. SQLite-backed objects provide a
10 GB private database and are the only backend with SQL and point-in-time
recovery. Existing KV-backed namespaces remain supported, but they cannot be
migrated in place to SQLite.

Accounts without an existing KV-backed namespace cannot create one with
`new_classes`. Do not design a new deployment around gaining access to the
legacy backend.

### Choose one lifecycle configuration model

Wrangler's declarative `exports` map and the ordered, tagged `migrations` array
are mutually exclusive. Existing Workers may retain migrations. Moving a
Worker to `exports` preserves its namespace data, but is one-way:

1. Replace the complete migrations array.
2. Add a live entry for every active namespace.
3. Use `sqlite` for `new_sqlite_classes` namespaces.
4. Use `legacy-kv` for `new_classes` namespaces.
5. Deploy with `wrangler deploy`.

Never change an existing namespace's storage type in place.

### Treat lifecycle edits as irreversible deploy boundaries

Only `wrangler deploy` applies an `exports` lifecycle operation.
`wrangler versions upload` rejects it, gradual deployment is unsupported, and
rollback cannot cross the lifecycle change. Named environments have separate
namespace state even when they inherit the same top-level configuration.

### Keep tombstones until reconciliation says they are safe

A lifecycle tombstone records an operation that must be reconciled across the
account. Keep it until Wrangler lists it under
`Safe to remove from exports`. A `deleted` entry is rejected while the class
still exists in code or any Worker still binds to its namespace. Renamed and
transferred tombstones remain necessary while `referencing_scripts` is
non-empty.

### Account for the alarm behavior of `deleteAll()`

At compatibility date `2026-02-24` or later,
`ctx.storage.deleteAll()` removes the alarm as well as all stored data on both
storage backends. Do not add a separate `deleteAlarm()` when implementing a
complete reset at that date.

## Lifecycle quick reference

### Reconcile code, configuration, and namespace state

A code export alone is ignored and does not create a namespace. Every desired
or provisioned namespace needs a matching `exports` entry. Live entries require
`storage`; tombstones must omit it. A live entry whose class is absent from the
Worker code fails deployment.

Live class entries default to the `created` state. Other states are `deleted`,
`renamed`, `transferred`, and the target-side `expecting-transfer`.

### Stage a zero-downtime rename

1. Deploy the new class and re-export it under the old name without changing
   `exports`.
2. Deploy an old-name `renamed` tombstone plus the new live entry while keeping
   the code alias.
3. Remove the alias after the rollout completes.

The target must be a different valid identifier, must be live in the same map,
and must not already own a namespace.

```ts
export class NewName extends DurableObject {
  // ...
}
export { NewName as OldName };
```

### Transfer a namespace target-first

For declarative cross-Worker transfer, deploy `expecting-transfer` on the target
first, with the source name and storage but without a self-referencing binding.
Then deploy `transferred` on the source to commit the handoff atomically. After
rollout, make the target entry live and add its binding; remove or redirect the
source binding with `script_name`.

Both Workers must share an account and dispatch-namespace context. Removing or
replacing the target's pending entry before source commit cancels the transfer.

## Storage quick reference

### Materialize SQL results before yielding

A `SqlStorageCursor` is not a stable snapshot across an `await`; later
iteration can observe mutations, even from a later implicit transaction that
eventually rolls back.

```ts
const rows = this.ctx.storage.sql.exec(
  "SELECT * FROM users",
).toArray();
await sendRows(rows);
```

One `exec()` call may contain multiple statements, but bindings apply only to
the last statement and only its cursor is returned. Cursor iteration and
`raw()` share a position. `one()` requires exactly one result row.

### Use storage transaction callbacks

Do not send `BEGIN TRANSACTION` or `SAVEPOINT` through `sql.exec()`. Use
`transactionSync()` for synchronous SQL or KV work. Its callback cannot return
a promise; its return value passes through, and throwing rolls back.

For SQLite-backed objects, direct storage operations—including SQL—participate
in `transaction()`. The older transaction wrapper is obsolete there.

### Make constructor schema changes atomic

SQLite Durable Objects do not support `PRAGMA user_version`. Record schema
versions in an ordinary table and run constructor migrations inside
`blockConcurrencyWhile()` so requests cannot observe a partial schema.

## Concurrency and lifecycle quick reference

### Recheck state after external I/O

Awaited storage operations receive input-gate protection. Awaiting `fetch()`,
R2, or another non-storage operation allows a different request to interleave.
After external I/O, revalidate the version or precondition on which a write
depends.

Pending writes hold outgoing responses and network requests at the output
gate. Consecutive writes without an intervening `await` are coalesced into one
atomic implicit transaction; an intervening `await` ends that boundary.

### Design for eviction and mixed versions

There is no shutdown hook before eviction, deployment, or runtime replacement,
so persist checkpoints incrementally. Worker and object code roll out with
eventual consistency: adjacent API versions can overlap for seconds to minutes,
and longer during a gradual deployment. Keep their request or RPC contract
forward- and backward-compatible.

Active `connect()` connections and outbound WebSockets temporarily defer
eviction. Each connection does so for at most 15 minutes; after all close, the
ordinary 70–140 second inactivity window begins.

### Make alarms idempotent and self-rescheduling

Alarms are non-recurring and may be delivered more than once. Schedule the next
run from the handler, tolerate retries, and use
`AlarmInvocationInfo.retryCount` when deciding whether to schedule a fresh
alarm before retries are exhausted.

## WebSocket quick reference

For hibernation, accept the server socket with `ctx.acceptWebSocket()` and
handle events in Durable Object methods such as `webSocketMessage()` and
`webSocketClose()`. Calling the standard `ws.accept()` does not enable
hibernation. Only inbound WebSockets served by the object can hibernate.

Store small per-socket state with `serializeAttachment()`. It saves a
structured-clone snapshot of at most 16,384 bytes; call it again after any
mutation that must persist. The attachment disappears when either side closes,
so durable or larger state belongs in object storage.

Use tags and auto-responses for low-cost routing and keepalive:

- At most 10 tags per socket and 256 characters per tag.
- At most one auto-response pair, with 2,048 characters per request and reply.
- Auto-responses do not wake a hibernating object.

## Identity and placement quick reference

`getByName()` and `idFromName()` objects can read their name from `ctx.id.name`;
unique IDs, IDs re-opened only from a string, and names over 1,024 bytes report
`undefined`.

Use `namespace.jurisdiction("eu" | "us" | "fedramp")` for residency. Each
jurisdiction is a distinct ID space, while an unscoped namespace can resolve a
restricted ID. A `locationHint` affects only best-effort first placement and
never relocates an existing object.

## Testing quick reference

Use Vitest 4 with `@cloudflare/vitest-pool-workers` and the
`cloudflareTest()` Vite plugin. Point it at the Wrangler configuration that
declares the bindings and lifecycle configuration. Type test bindings by adding
the pool's types and augmenting `cloudflare:workers` with `ProvidedEnv`.

Exercise Worker routing through `exports.default.fetch()`; call a namespace
binding directly when the object itself is the test boundary. Storage for a
named object persists across tests in the same file. Eviction resets memory,
not durable storage.
