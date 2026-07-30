# Deployment and class lifecycle

## Choose a storage backend

SQLite-backed Durable Objects are generally available (2025). They provide a
10 GB database per object and are recommended for every new class through
`new_sqlite_classes`. Only SQLite-backed objects expose SQL and point-in-time
recovery. KV-backed objects remain supported, but existing KV-backed classes
cannot be migrated to SQLite in place.

```toml
[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyDurableObject"]
```

An account with no existing KV-backed Durable Object namespace cannot create
one with `new_classes` (2026). Such a deployment must create a SQLite-backed
namespace. Accounts that already have at least one KV-backed namespace may
still create more; existing namespaces are unaffected.

## Use either declarative exports or migrations

Wrangler's `exports` map is a current-state alternative to the ordered, tagged
`migrations` array. The two forms are mutually exclusive in one Worker.
Existing Workers may keep migrations.

Entries are keyed by class name. A live class defaults to `created`; lifecycle
states also include `deleted`, `renamed`, `transferred`, and the receiving-side
`expecting-transfer`.

```jsonc
{
  "exports": {
    "ChatRoom": {
      "type": "durable-object",
      "state": "renamed",
      "renamed_to": "Room"
    },
    "Room": {
      "type": "durable-object",
      "storage": "sqlite"
    }
  }
}
```

Declarative tombstones permit staged zero-downtime renames and cross-Worker
transfers. Wrangler also reports other Workers whose bindings still reference
a class being renamed or deleted.

## Reconcile code, configuration, and namespaces

A class exported only by Worker code is ignored and does not receive a
namespace. Every desired or already-provisioned namespace needs a matching
`exports` entry.

- A live entry requires `storage`.
- A tombstone forbids `storage`.
- A live entry whose class is absent from code fails deployment.

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

## Keep tombstones through reconciliation

A `deleted` tombstone is rejected while its class remains in code or any Worker
in the account still binds to its namespace. Once the operation lands, keep its
stale tombstone until reconciliation lists it under
`Safe to remove from exports`.

Renamed and transferred tombstones are not safe to remove while
`referencing_scripts` is non-empty.

## Stage a zero-downtime rename

1. Deploy the new class while re-exporting it under the old name. Leave
   `exports` unchanged.
2. Deploy the old-name `renamed` tombstone and the new live entry while keeping
   the alias.
3. Remove the alias after rollout.

```ts
export class NewName extends DurableObject {
  // ...
}
export { NewName as OldName };
```

The rename target must be a different valid identifier, live in the same map,
and not already associated with a namespace.

## Transfer across Workers target-first

For declarative transfer, the target first deploys `expecting-transfer` with
the source name and storage, but without a binding that points back to itself.
The source then deploys `transferred`, atomically committing the handoff.

```jsonc
{
  // Target, deployed first
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
  // Source, deployed second
  "exports": {
    "MyDO": {
      "type": "durable-object",
      "state": "transferred",
      "transferred_to": "target-worker"
    }
  }
}
```

After rollout, change the target entry to live and add its binding. Remove or
redirect the source binding with `script_name`. Both Workers must be in the
same account and dispatch-namespace context.

Removing or replacing the target's pending `expecting-transfer` entry before
the source commits cancels the transfer without moving the namespace.

## Respect environment and deployment boundaries

Named environments inherit the top-level `exports` unless they override it.
Each environment nevertheless owns separate namespaces, and a tombstone only
affects its environment.

Only `wrangler deploy` applies lifecycle changes:

- `wrangler versions upload` rejects `exports`.
- Gradual deployment is unsupported.
- Rollback cannot cross a lifecycle change.

## Convert migrations to exports once

Moving from migrations to `exports` preserves data but is one-way. Replace the
entire migrations array with live entries for all active namespaces:

- Use `sqlite` for a namespace created by `new_sqlite_classes`.
- Use `legacy-kv` for a namespace created by `new_classes`.

No namespace data moves. Storage cannot later be changed in place, and the
Worker cannot return to legacy migrations after deploying `exports`.

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

## Perform a legacy transfer at the destination

In the legacy flow, put `transferred_classes` in the destination Worker's
migration and export the destination class. Do not create the destination
namespace first; the transfer creates it. `from`, `from_script`, and `to` can
rename the class while transferring it.

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
