# Runtime and Web-Platform Behavior

Use this reference for date-gated request behavior, caching, streams,
JavaScript and web APIs, execution contexts, serialization, and runtime data.

## Requests, cancellation, and redirects

`enable_request_signal` exposes cancellation of an incoming request through
`Request.signal` (date `2025-05-22`). `request_signal_passthrough` forwards
that signal when the request is passed to `fetch()` (date `2025-05-05`).
Enable both when downstream work should stop after the client disconnects.

From `2025-09-01`, `fetch()` strips `Authorization` when following a redirect
to another origin. Use `retain_authorization_on_cross_origin_redirect` only
when credentials are intentionally safe at the redirect target.

From `2026-02-19`, synchronous and asynchronous iterables supplied as
`Request` or `Response` bodies are consumed as streaming content. Arrays no
longer become strings such as `"1,2,3"`. A synchronous iterable that defines
its own `toString` or `Symbol.toPrimitive` stays on the string-coercion path.

```js
const body = (async function* () {
  yield new TextEncoder().encode("hello");
})();
return new Response(body);
```

## Cache behavior

From `2025-05-19`, a Request's `cf` settings passed to the Cache API override
Cache Rules for user-owned or grey-clouded sites.

From `2025-08-07`, `fetch(url, { cache: "no-cache" })` is accepted. For a
subrequest to an origin outside the platform, it forces a conditional origin
request even when the cached response is fresh and adds
`Pragma: no-cache` and `Cache-Control: no-cache`.

Workers subrequests also accept `cf.vary` to control caching of a single origin
response with a `Vary` header. The configuration can bypass unconfigured
headers and normalize selected values:

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

## Text, encoding, and stream behavior

From `2026-01-13`, stream `readAllText()` strips a leading UTF-8 BOM.
`do_not_strip_bom_in_read_all_text` restores the earlier result.

The `2026-02-24` and `2026-03-03` gates replace lone UTF-16LE surrogates with
U+FFFD and select standards-oriented CJK and Big5 decoding.

From `2026-03-24`, `TextEncoderStream` and `TextDecoderStream` have a readable
high-water mark of 0. Writes wait for a reader to pull rather than clearing
backpressure during startup. The same date enables standards-compliant
`WritableStream` writer lock and release behavior.

## Web-platform conformance

- `2025-05-01` selects the standards-compliant `URLPattern`;
  `urlpattern_original` restores the previous implementation.
- `2025-06-16` makes unknown import attributes throw.
- `2025-08-01` binds an `EventTarget` listener's `this` to its target.
- `pedantic_wpt` opts into stricter `Event` and `EventTarget` behavior.
- `enable_navigator_language` exposes `navigator.language`, always as `en`.

From `2026-03-03`, `unhandledrejection` waits until the current microtask
checkpoint finishes. A rejection handler attached later in the same
multi-microtask promise chain can therefore prevent the event.

## JavaScript APIs

From `2025-05-05`, `enable_weak_ref` exposes `WeakRef` and
`FinalizationRegistry`. Finalizers are nondeterministic and may never run.
They can run after the handler completes, have no associated async context,
and cannot perform I/O or emit tail events.

Workers added explicit resource management and `Float16Array` in May, global
`MessageChannel` and `MessagePort` in August, and `Uint8Array` base64 and hex
operations with the V8 14.0 rollout. `expose_global_message_channel`
explicitly gates the global messaging constructors.

From `2025-06-01`, `eval()` and `new Function()` are permitted during Worker
startup. `disallow_eval_during_startup` restores startup errors.

From `2026-04-21`, `structuredClone()` and V8 serialization preserve more
error types and an error object's own properties. Deserialization does not
preserve the original stack by default. `legacy_error_serialization` selects
the older basic-error behavior.

## Execution contexts and exports

Every event-handler invocation receives a fresh `ctx`, including at older
compatibility dates. `nonclass_entrypoint_reuses_ctx_across_invocations`
restores the old reuse behavior.

From `2025-06-16`, AsyncLocalStorage snapshots and bound functions belong to
the request that created them and throw when called from another request.

From `2025-11-17`, `ctx.exports` supplies automatically configured loopback
bindings for top-level exports. A Worker can call its own
`WorkerEntrypoint` exports without declaring explicit service bindings.

## Runtime data and special handlers

From `2025-12-03`, optimized runtime structs install optional properties with
the value `undefined` instead of omitting them. Test
`obj.key !== undefined`; do not infer a value from `"key" in obj` or
`Object.hasOwn(obj, "key")`.

From `2025-08-01`, `set_forwardable_email_full_headers` makes email Workers
receive every value of headers such as `To` and `Cc`, rather than one
truncated value.

Dynamic Worker creation accepts a null name. The runtime update of
`2026-04-17` also permits custom limits when creating Dynamic Workers.
