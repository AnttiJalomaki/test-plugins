# Testing Durable Objects

## Configure Workers Vitest 4

Install `vitest@^4.1.0` with `@cloudflare/vitest-pool-workers`. Add
`cloudflareTest()` to the Vite plugin list and point it to the Wrangler
configuration that declares the Durable Object bindings and lifecycle
configuration.

```ts
export default defineConfig({
  plugins: [
    cloudflareTest({
      wrangler: { configPath: "./wrangler.jsonc" },
    }),
  ],
});
```

## Type the provided test environment

Include `@cloudflare/vitest-pool-workers/types` in the test TypeScript
configuration. Augment `cloudflare:workers` so the test runtime's `env` uses
the project's binding types.

```ts
declare module "cloudflare:workers" {
  interface ProvidedEnv extends Env {}
}
```

## Test at the Worker or object boundary

Use `exports.default.fetch()` to exercise the default Worker's HTTP handler and
its routing to Durable Objects.

```ts
const response = await exports.default.fetch(
  "http://example.com?id=http-test",
  { method: "POST" },
);
```

Use a binding from `env` when the test should call a Durable Object stub
directly.

## Account for per-file storage persistence

Repeated access to the same named object across tests in one file sees data
stored by earlier tests. Distinct IDs have independent storage.

`evictAllDurableObjects()` resets running instances without deleting that
persisted data.

## Run scheduled alarms

`runDurableObjectAlarm(stub)` immediately executes a scheduled future alarm
and returns `true`. Calling it when no alarm remains returns `false`.

```ts
const ran = await runDurableObjectAlarm(stub);
expect(ran).toBe(true);
expect(await runDurableObjectAlarm(stub)).toBe(false);
```

## Exercise eviction behavior

`@cloudflare/vitest-pool-workers` 0.16.20 and later exports
`evictDurableObject` and `evictAllDurableObjects` from `cloudflare:test`
(2026).

```ts
import {
  evictAllDurableObjects,
  evictDurableObject,
} from "cloudflare:test";

const stub = env.COUNTER.getByName("my-counter");
await evictDurableObject(stub);
await evictDurableObject(stub, { webSockets: "close" });
await evictAllDurableObjects();
```

Targeted eviction normally simulates eviction with WebSocket hibernation. Pass
`{ webSockets: "close" }` to test the non-hibernating path.

Before tearing down an instance, `evictDurableObject()` waits up to 30 seconds
for in-flight requests to drain. It clears in-memory state but preserves the
object's durable storage.
