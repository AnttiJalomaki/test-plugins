# Concurrency, alarms, and runtime lifecycle

## Know when an object can hibernate

An otherwise eligible object currently hibernates after about 10 seconds with
no event. Its memory is discarded and its constructor runs on the next event,
while hibernatable client WebSockets remain connected.

Timers, an unfinished request or event, an in-progress awaited `fetch()`, or
standard WebSocket API use prevent hibernation. A plain `fetch()` subrequest
does not keep the object alive merely because its returned body is still
streaming.

Active connections made with `connect()` or an outbound WebSocket keep the
object alive. Once all close, the usual 70–140-second inactivity window begins.
Each connection prevents eviction for at most 15 minutes; afterward normal
eviction rules resume even if the connection stays open.

## Distinguish storage gates from external I/O

Awaited Durable Object storage operations receive input-gate protection.
Awaiting `fetch()`, R2, or other non-storage I/O allows another request to
interleave. Revalidate a version or other precondition after external I/O
before committing dependent changes.

Output gates hold outgoing responses and network requests until pending
storage writes complete. Consecutive writes with no intervening `await` are
coalesced into one atomic implicit transaction. An `await` between legacy KV
writes ends that coalescing boundary.

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

## Migrate schemas before requests enter

Durable Object SQLite does not support `PRAGMA user_version`. Track applied
migrations in an ordinary table and run constructor migrations inside
`blockConcurrencyWhile()` so no request can observe a partial schema.

```ts
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);
  ctx.blockConcurrencyWhile(async () => {
    await this.migrateUsingVersionTable();
  });
}
```

## Expect stale events to be stopped at storage

Global uniqueness is checked when an event starts and again whenever it
accesses storage. A stale HTTP or RPC event that never touches storage may
finish after a replacement instance starts, but a later storage access fails.
WebSocket requests are terminated during shutdown. Requests interrupted by a
runtime update have at most 30 seconds to finish.

No shutdown hook or lifecycle callback runs before deployment, eviction, or
runtime replacement. Persist checkpoints incrementally rather than relying on
a final flush.

## Keep adjacent deployments compatible

Worker and Durable Object code rolls out with eventual consistency. New Worker
code can call an older object version for seconds to minutes, and a gradual
deployment lengthens the overlap. Keep HTTP and RPC contracts forward- and
backward-compatible across adjacent releases.

## Treat alarms as retryable one-shots

Alarms are non-recurring and may be delivered more than once. Each handler must
schedule its own next run and be idempotent.
`AlarmInvocationInfo.retryCount` can guide scheduling a fresh alarm before the
remaining retries are exhausted.

Under local `wrangler dev`, alarm methods may fail after a hot reload. Restart
the command after changing alarm code.

## Understand local persistence with `script_name`

By default, `wrangler dev` can read Durable Object storage but keeps writes in
memory without changing persistent data. When a binding explicitly sets
`script_name`, development writes do affect persistent storage, and Wrangler
emits a warning.
