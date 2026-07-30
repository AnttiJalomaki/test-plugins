# Identity and placement

## Read a named object's name

An object reached through `idFromName()` or `getByName()` can read that same
name from `ctx.id.name` (2026), including inside an alarm handler.

```js
export class ChatRoom extends DurableObject {
  getRoomName() {
    return this.ctx.id.name;
  }
}
```

The value is `undefined` for IDs created by `newUniqueId()`, access through
`idFromString()`, and names longer than 1,024 bytes.

## Scope compute and data to a jurisdiction

Use `namespace.jurisdiction("us")` to constrain an object's compute and stored
data to the United States. Workers outside the US can still access it.

```js
const usObjects = env.MY_DURABLE_OBJECT.jurisdiction("us");
const stub = usObjects.getByName("general");
```

Jurisdiction-scoped namespaces support `eu`, `us`, and `fedramp`. Each scope
has a distinct ID space:

- The same name produces a different ID in each jurisdiction.
- A scoped namespace rejects an ID from another jurisdiction.
- The unscoped namespace can resolve a restricted ID.

Prefer `namespace.jurisdiction()` over per-ID
`newUniqueId({ jurisdiction })`. A `DurableObjectId` can still be logged
outside its jurisdiction for billing and debugging.

```js
const euRooms = env.ROOMS.jurisdiction("eu");
const euId = euRooms.idFromName("lobby");
const stub = env.ROOMS.get(euId);
```

## Preserve jurisdiction through ID round-trips

Inside the object, `ctx.id.jurisdiction` reports its jurisdiction. The value
survives `toString()` followed by `idFromString()`.

It is also available to alarm handlers for alarms scheduled on `2026-03-15`
or later.

```js
export class Room extends DurableObject {
  getJurisdiction() {
    return this.ctx.id.jurisdiction;
  }
}
```

## Hint first placement

Both `get(id, { locationHint })` and
`getByName(name, { locationHint })` accept a best-effort placement hint. The
hint matters only on the object's first access and never relocates an existing
object.

```js
const stub = env.ROOMS.getByName("tokyo", {
  locationHint: "apac-ne",
});
```

`apac-ne` and `apac-se` narrow the broader `apac` region. The accepted `sam`,
`afr`, and `me` hints currently fall back to a nearby region that supports
Durable Objects.
