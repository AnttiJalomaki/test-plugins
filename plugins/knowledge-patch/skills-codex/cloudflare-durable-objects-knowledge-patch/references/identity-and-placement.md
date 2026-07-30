# Identity and placement

## Recover a named object's name

An object reached through `idFromName()` or `getByName()` can read that same
name from `ctx.id.name`, including in an alarm handler.

```ts
export class ChatRoom extends DurableObject {
  getRoomName() {
    return this.ctx.id.name;
  }
}
```

The property is `undefined` for IDs produced by `newUniqueId()`, when access
uses `idFromString()`, and for names longer than 1,024 bytes.

## Scope namespaces by jurisdiction

Use `namespace.jurisdiction("eu" | "us" | "fedramp")` to constrain compute and
stored data. Workers outside the selected jurisdiction can still access the
object. The US scope is available as `namespace.jurisdiction("us")`.

```ts
const usObjects = env.ROOMS.jurisdiction("us");
const stub = usObjects.getByName("general");
```

Each jurisdiction has a distinct ID space. The same name produces a different
ID in each scope, and a scoped namespace rejects an ID from another
jurisdiction. An unscoped namespace can resolve a restricted ID.

Prefer scoping the namespace over per-ID
`newUniqueId({ jurisdiction })`. A `DurableObjectId` may still be logged
outside its jurisdiction for billing and debugging.

## Preserve jurisdiction through ID round-trips

Inside an object, `ctx.id.jurisdiction` reports the selected jurisdiction and
survives `toString()` followed by `idFromString()`. Alarm handlers also receive
it for alarms scheduled on `2026-03-15` or later.

## Use location hints only for first placement

Both `get(id, { locationHint })` and
`getByName(name, { locationHint })` accept a best-effort hint for the object's
first access. A hint does not relocate an existing object.

```ts
const stub = env.ROOMS.getByName("tokyo", {
  locationHint: "apac-ne",
});
```

`apac-ne` and `apac-se` narrow the broader `apac` region. `sam`, `afr`, and
`me` are accepted but currently fall back to a nearby region that supports
Durable Objects.
