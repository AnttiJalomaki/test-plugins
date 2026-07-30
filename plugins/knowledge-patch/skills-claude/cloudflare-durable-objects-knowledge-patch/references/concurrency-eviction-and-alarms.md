# Concurrency, eviction, and alarms

## Know when hibernation is possible

An idle object cannot hibernate while it has a timer, an in-progress awaited
`fetch()` handler, standard WebSocket API use, or any unfinished request or
event. A plain `fetch()` subrequest does not keep the object alive merely
because its returned body is still streaming.

An otherwise eligible object currently hibernates after 10 seconds without an
event. Hibernation discards memory and runs the constructor again on the next
event while hibernatable client WebSockets remain connected.

## Account for outbound connection eviction deferral

An active connection created with `connect()` or an outbound WebSocket keeps
the object alive. After every such connection closes, the normal 70–140 second
inactivity window begins.

Each connection prevents eviction for at most 15 minutes. After that, normal
eviction rules resume even if the connection remains open.

## Revalidate after non-storage I/O

Awaited Durable Object storage operations receive input-gate protection.
Awaiting `fetch()`, R2, or other non-storage I/O lets another request
interleave. Revalidate a version or other precondition after external I/O
before committing a dependent storage change.

## Use output gates and implicit transactions

Outgoing responses and network requests are held until pending storage writes
complete.

Consecutive writes without an intervening `await` are coalesced into one atomic
implicit transaction. An intervening `await`, including between legacy KV
writes, ends that coalescing boundary.

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

## Version the SQLite schema in a table

Durable Object SQLite does not support `PRAGMA user_version`. Track applied
migrations in an ordinary table. Run constructor migrations inside
`blockConcurrencyWhile()` so no request observes a partial schema.

```ts
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);
  ctx.blockConcurrencyWhile(async () => {
    await this.migrateUsingVersionTable();
  });
}
```

## Rely on storage boundaries for uniqueness

Global uniqueness is enforced when an event starts and whenever it accesses
storage. A stale HTTP or RPC event that never touches storage may finish after
a replacement instance starts; a later storage access stops that stale event
with an error.

WebSocket requests are terminated during shutdown. Requests affected by a
runtime update have at most 30 seconds to finish.

## Persist without a finalizer

Durable Objects have no shutdown hook or lifecycle callback before deployment,
eviction, or runtime-driven replacement. Persist checkpoints incrementally
instead of relying on an end-of-process flush.

## Keep adjacent deployments compatible

Worker and Durable Object code roll out with eventual consistency. New Worker
code can call an older object version for seconds to minutes, and a gradual
deployment extends the overlap. Keep request and RPC contracts forward- and
backward-compatible across adjacent releases.

## Make alarms idempotent and self-rescheduling

Alarms are non-recurring and may be delivered more than once. An alarm handler
must schedule its next run and be idempotent.

Use `AlarmInvocationInfo.retryCount` to decide whether to schedule a fresh
alarm before the remaining retries are exhausted. Under local `wrangler dev`,
alarm methods may fail after a hot reload; restart the command after editing
alarm code.

## Understand local persistence with `script_name`

By default, `wrangler dev` can read Durable Object storage, but keeps writes in
memory and does not change persistent data.

If a binding explicitly sets `script_name`, development writes do affect
persistent storage. Wrangler emits a warning for this mode.
