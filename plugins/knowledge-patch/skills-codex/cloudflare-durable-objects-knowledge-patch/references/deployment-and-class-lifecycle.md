# Deployment and class lifecycle

## Choose and preserve the storage backend

SQLite-backed Durable Objects became generally available in 2025. Each object
has a 10 GB database, and new classes should use this backend through
`new_sqlite_classes` or a live declarative entry with `"storage": "sqlite"`.
Only SQLite-backed objects expose SQL and point-in-time recovery. Existing
KV-backed objects remain supported, but there is no migration path that
converts an existing KV-backed namespace to SQLite.

An account with no existing KV-backed Durable Object namespace can no longer
create one with `new_classes`; create a SQLite-backed namespace instead.
Accounts that already have a KV-backed namespace may still create more for
now, and existing namespaces are unaffected.

```toml
[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyDurableObject"]
```

## Reconcile code, configuration, and namespace state

Wrangler's `exports` map is a declarative alternative to the ordered, tagged
`migrations` array. The two forms are mutually exclusive in one Worker.
Entries are keyed by class name and use `"type": "durable-object"`. A live
entry defaults to `created`; other lifecycle states are `deleted`, `renamed`,
`transferred`, and the receiving state `expecting-transfer`.

A class exported only by Worker code is ignored and gets no namespace. Every
desired or provisioned namespace needs a corresponding declarative entry. Live
entries require `storage`; `deleted`, `renamed`, and `transferred` tombstones
forbid it; and `expecting-transfer` declares the receiving backend. Deployment
fails when a live entry's class is absent from code.

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

Wrangler also reports other Workers whose bindings still reference a class
being renamed or deleted.

## Keep tombstones until reconciliation clears them

A `deleted` tombstone is rejected while the class remains in code or any Worker
in the account still binds to its namespace. Once an operation lands, keep its
stale tombstone until reconciliation lists it under
`Safe to remove from exports`. A renamed or transferred tombstone is not safe
to remove while `referencing_scripts` is non-empty.

## Stage a zero-downtime rename

First deploy the new class while re-exporting it under the old name, leaving
the declarative map unchanged. Next, deploy the old-name `renamed` tombstone
and the new live entry while retaining the alias. Remove the alias after
rollout.

```ts
export class NewName extends DurableObject {
  // ...
}
export { NewName as OldName };
```

The rename target must be a different valid identifier, must be live in the
same map, and must not already own a namespace.

## Transfer between Workers target-first

For a declarative cross-Worker transfer:

1. On the target, deploy `expecting-transfer` with the source class name,
   backend, and `transfer_from`, but no self-referencing binding.
2. On the source, deploy `transferred` with `transferred_to`. This atomically
   commits the handoff.
3. Make the target entry live and add its binding.
4. Remove the source binding or redirect it using `script_name`.

```jsonc
{
  "exports": {
    "MyDO": {
      "type": "durable-object",
      "state": "expecting-transfer",
      "storage": "sqlite",
      "transfer_from": "source-worker"
    }
  }
}
```

```jsonc
{
  "exports": {
    "MyDO": {
      "type": "durable-object",
      "state": "transferred",
      "transferred_to": "target-worker"
    }
  }
}
```

Both Workers must be in the same account and dispatch-namespace context.
Removing or replacing the target's pending entry before the source commits
cancels the transfer without moving the namespace.

## Apply lifecycle state with a full deploy

Named environments inherit top-level declarative entries unless they override
them, but each environment owns separate namespaces and its tombstones affect
only that environment.

Only `wrangler deploy` applies declarative lifecycle changes.
`wrangler versions upload` rejects them, gradual deployment is unsupported,
and rollback cannot cross a lifecycle change.

## Convert legacy migrations once

To move from legacy migrations, replace the entire migration array with live
entries for every active namespace. Use `sqlite` for classes originally made
by `new_sqlite_classes` and `legacy-kv` for classes made by `new_classes`.
Namespace data does not move.

```jsonc
{
  "exports": {
    "ExistingKvClass": {
      "type": "durable-object",
      "storage": "legacy-kv"
    }
  }
}
```

Storage cannot be changed in place. After deploying the declarative map, the
Worker cannot return to legacy migrations.

## Use the destination migration for a legacy transfer

In the legacy cross-script flow, put `transferred_classes` in the destination
Worker's migration and export the destination class. Do not create its
namespace first; the transfer creates it. `from`, `from_script`, and `to` can
also rename the class during transfer.

```jsonc
{
  "migrations": [{
    "tag": "v4",
    "transferred_classes": [{
      "from": "OldClass",
      "from_script": "source-worker",
      "to": "NewClass"
    }]
  }]
}
```
