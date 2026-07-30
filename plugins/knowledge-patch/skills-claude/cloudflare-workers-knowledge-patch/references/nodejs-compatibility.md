# Node.js compatibility

## Choose the smallest compatibility surface

Use `nodejs_compat` for the broad Node.js API surface. Use `nodejs_als` when a
Worker needs only `AsyncLocalStorage`.

```jsonc
{
  "compatibility_flags": ["nodejs_als"]
}
```

The runtime's native API coverage includes:

- assertion testing;
- Buffer and Crypto;
- Events and Net;
- Path, query strings, String Decoder, and URL;
- Utilities and Zlib.

Chrome DevTools provides debugging. DNS, Module, the test runner, and TLS/SSL
are only partially supported.

## Binding and process behavior

With `nodejs_compat` and compatibility date `2025-04-01`, text and JSON
bindings, secrets, and version metadata populate `process.env`.
`nodejs_compat_do_not_populate_process_env` opts out.
`disallow_importable_env` also prevents population and blocks environment
imports from `cloudflare:workers`.

From `2025-09-15`, process v2 replaces the small shim centered on `nextTick`,
`env`, and `exit`. It provides a broader implementation and represents
unsupported exports as `undefined`. `disable_nodejs_process_v2` keeps the old
shim.

From `2026-01-22`, `require()` returns a module's default export when it has one.
Otherwise it returns a mutable copy of the namespace object. Use
`require_returns_namespace` to retain the former always-namespace behavior.

## Module rollout

With `nodejs_compat`, compatibility dates in batch `2025` add:

- client `node:http` and `node:https`;
- `node:http2` stubs and HTTP server APIs;
- `node:fs` and Web File System APIs;
- `node:os` and `node:console`;
- a `node:vm` stub;
- `node:cluster`, `node:domain`, `node:punycode`, `node:trace_events`, and
  `node:wasi` as modules or stubs.

From `2026-01-29`, import-compatible stubs are enabled for:

- `node:_stream_wrap`;
- `node:dgram`;
- `node:inspector`;
- `node:sqlite`.

From `2026-03-17`, import-compatible stubs are enabled for:

- `node:child_process`;
- `node:readline`;
- `node:repl`;
- `node:tty`;
- `node:v8`;
- `node:worker_threads`.

The same date adds an implemented `node:perf_hooks`. An import-only stub does
not provide the underlying host facility.

## Per-module stub flags

Enable a stub before its automatic date or disable it afterward with a
module-specific `enable_nodejs_<name>_module` or
`disable_nodejs_<name>_module` flag.

The leading underscore in `node:_stream_wrap` is omitted:

```jsonc
{
  "compatibility_flags": [
    "nodejs_compat",
    "enable_nodejs_sqlite_module",
    "disable_nodejs_stream_wrap_module"
  ]
}
```

## Timers and performance

With `nodejs_compat` from `2026-02-10`, global timer functions return
Node-compatible `Timeout` objects with:

- `refresh()`;
- `ref()`;
- `unref()`;
- `hasRef()`.

From `2026-03-17`, global scope exposes:

- `PerformanceEntry`;
- `PerformanceMark`;
- `PerformanceMeasure`;
- `PerformanceResourceTiming`;
- `PerformanceObserver`;
- `PerformanceObserverEntryList`.

Enabling `node:perf_hooks` enables these globals implicitly.

## End-of-life APIs and compatibility corrections

From `2025-09-01`, `nodejs_compat` applies roll-up removal of Node APIs that have
reached end of life, including version-specific removals such as Node.js 23
APIs. `add_nodejs_compat_eol` is a temporary escape hatch, not a long-term
target.

With `nodejs_compat` from `2026-05-19`,
`Channel.hasSubscribers` and `TracingChannel.hasSubscribers` in
`node:diagnostics_channel` are read-only boolean properties, not methods.

From `2026-06-16`, unsupported options such as `checkServerIdentity` passed to
`tls.connect()` or `new TLSSocket()` throw `ERR_OPTION_NOT_IMPLEMENTED` rather
than being silently ignored.

## TypeScript declarations

Generate declarations that match compatibility settings and bindings:

```sh
wrangler types
wrangler types --check
```

Include `worker-configuration.d.ts` through `compilerOptions.types`, and add
`@types/node` when using `nodejs_compat`.
