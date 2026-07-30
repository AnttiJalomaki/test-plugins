# Runtime compatibility

## Environment and request behavior

### Binding values in `process.env`

With `nodejs_compat`, compatibility date `2025-04-01` populates `process.env`
from text and JSON bindings, including variables, secrets, and version metadata.
Use `nodejs_compat_do_not_populate_process_env` to opt out.

The undated `disallow_importable_env` flag both blocks environment imports from
`cloudflare:workers` and prevents this `process.env` population.

### Incoming cancellation

- `enable_request_signal` exposes incoming request cancellation through
  `Request.signal`; its compatibility date is `2025-05-22`.
- `request_signal_passthrough` propagates the incoming signal when forwarding
  with `fetch()`; its compatibility date is `2025-05-05`.

Enable the first when a handler must observe client cancellation and the second
when the downstream fetch should be cancelled with it.

### Authorization across redirects

From `2025-09-01`, `fetch()` removes `Authorization` when following a redirect
to another origin. Set `retain_authorization_on_cross_origin_redirect` only
when retaining credentials is intentional and safe.

### Iterable bodies

From `2026-02-19`, synchronous and asynchronous iterables passed as `Request` or
`Response` bodies are consumed as streams. Arrays no longer become strings such
as `"1,2,3"`.

A synchronous iterable that defines its own `toString` or
`Symbol.toPrimitive` remains on the string-coercion path.

```js
const body = (async function* () {
  yield new TextEncoder().encode("hello");
})();
return new Response(body);
```

## Cache behavior

From `2025-05-19`, a Request's `cf` settings passed to the Cache API override
Cache Rules for user-owned or grey-clouded sites.

From `2025-08-07`, this form is accepted:

```js
fetch(url, { cache: "no-cache" });
```

For subrequests to origins outside the platform, it forces revalidation with a
conditional origin request even when the cached response is fresh, and adds
`Pragma: no-cache` and `Cache-Control: no-cache`.

### Per-request variation

Workers subrequests accept `cf.vary` to interpret an origin `Vary` response
header. It can bypass unconfigured headers and normalize selected values:

```ts
return fetch(request, {
  cf: {
    vary: {
      default: { action: "bypass" },
      headers: {
        accept: {
          action: "normalize",
          media_types: ["text/html", "application/json"],
        },
      },
    },
  },
});
```

## Web-platform conformance

Compatibility gates from batch `2025` include:

| Date or flag | Behavior |
|---|---|
| `2025-05-01` | standards-compliant `URLPattern`; `urlpattern_original` rolls back |
| `2025-06-16` | unknown import attributes throw |
| `2025-08-01` | an `EventTarget` listener's `this` is its target |
| `pedantic_wpt` | stricter `Event` and `EventTarget` behavior |
| `enable_navigator_language` | exposes `navigator.language`, always `en` |

From `2025-05-05`, `enable_weak_ref` exposes `FinalizationRegistry` and
`WeakRef`. Finalization is nondeterministic and may never occur. A finalizer can
run after the handler finishes, has no associated async context, and cannot do
I/O or emit tail events.

Additional JavaScript APIs include:

- explicit resource management and `Float16Array`, added in May;
- global `MessageChannel` and `MessagePort`, added in August and explicitly
  gateable with `expose_global_message_channel`;
- `Uint8Array` base64 and hex operations from the V8 14.0 rollout.

From `2025-06-01`, `eval()` and `new Function()` are permitted during startup.
Use `disallow_eval_during_startup` to restore startup errors.

## Execution contexts and optional properties

Every event-handler invocation receives a fresh `ctx`, retroactively across
compatibility dates. The
`nonclass_entrypoint_reuses_ctx_across_invocations` flag restores reuse.

From `2025-06-16`, AsyncLocalStorage snapshots and bound functions belong to the
request that created them and throw when called from another request.

From `2025-12-03`, an optimized runtime struct representation explicitly
installs optional fields with value `undefined`. Test values:

```js
if (obj.key !== undefined) {
  // field has a usable value
}
```

Do not infer a usable value from `"key" in obj` or
`Object.hasOwn(obj, "key")`.

## Text, streams, and errors

Text decoding gates from batch `2026`:

- `2026-01-13`: stream `readAllText()` strips a leading UTF-8 BOM;
  `do_not_strip_bom_in_read_all_text` restores the old result.
- `2026-02-24`: lone UTF-16LE surrogates are replaced with U+FFFD.
- `2026-03-03`: CJK and Big5 decoding becomes standards-oriented.

From `2026-03-24`, `TextEncoderStream` and `TextDecoderStream` use a readable
high-water mark of 0. Writes wait for a reader to pull rather than clearing
backpressure at startup. The same date selects standards-compliant
`WritableStream` writer locking and release.

From `2026-04-21`, `structuredClone()` and V8 serialization retain more error
types and an error object's own properties. Deserialization does not preserve
the original stack by default. `legacy_error_serialization` restores the older
basic-error representation.

From `2026-03-03`, `unhandledrejection` waits until the current microtask
checkpoint completes. A handler installed later in the same multi-microtask
promise chain can prevent the event.

## Routing, observability, and other runtime capabilities

From `2025-04-01`, a navigation request prefers Static Assets fallback handling
even without an exact asset match. An SPA `/index.html` or custom `/404.html`
therefore runs before the Worker unless `assets.run_worker_first = true`
applies.

From `2025-11-05`, `"observability": { "enabled": true }` also enables automatic
Workers tracing. With an earlier date, enable traces explicitly:

```json
{
  "observability": {
    "traces": { "enabled": true }
  }
}
```

Preview warnings that were once visible only in DevTools are also sent to an
attached tail Worker.

From `2025-11-17`, `ctx.exports` supplies automatically configured loopback
bindings for top-level exports. A Worker can call one of its own
`WorkerEntrypoint` exports without declaring an explicit service binding.

From `2025-08-01`, `set_forwardable_email_full_headers` lets email Workers
receive all values for headers such as `To` and `Cc`, rather than one truncated
value.

Dynamic Worker creation accepts a null name. The runtime update of `2026-04-17`
also permits custom limits when creating Dynamic Workers.
