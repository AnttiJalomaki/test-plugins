# Node.js Compatibility

Use this reference to select a Node.js compatibility mode, reason about
implemented and stubbed APIs, and account for date-gated interoperability.

## Compatibility modes and environment bindings

Use `nodejs_compat` for the broad Node.js surface. When only
`AsyncLocalStorage` is needed, select the narrower `nodejs_als` flag:

```jsonc
{
  "compatibility_flags": ["nodejs_als"]
}
```

With `nodejs_compat`, compatibility date `2025-04-01` populates `process.env`
from text and JSON bindings, including variables, secrets, and version
metadata. `nodejs_compat_do_not_populate_process_env` opts out.

The undated `disallow_importable_env` flag both blocks environment imports
from `cloudflare:workers` and prevents population of `process.env`.

## Native, partial, and stub coverage

Native runtime coverage includes assertion testing, Buffer, Crypto, Events,
Net, Path, query strings, String Decoder, URL, Utilities, and Zlib.
Debugging uses Chrome DevTools.

DNS, Module, the test runner, and TLS/SSL are only partially supported. A stub
module can satisfy an import without providing its host facility; do not treat
successful import as proof that an operation is implemented.

The 2025 `nodejs_compat` additions include:

- client `node:http` and `node:https`;
- `node:http2` stubs and HTTP server APIs;
- `node:fs` and the Web File System APIs;
- `node:os`, `node:console`, and a `node:vm` stub;
- `node:cluster`, `node:domain`, `node:punycode`, `node:trace_events`, and
  `node:wasi` modules or stubs.

From `2026-01-29`, import-compatible stubs are enabled for
`node:_stream_wrap`, `node:dgram`, `node:inspector`, and `node:sqlite`.

From `2026-03-17`, stubs are added for `node:child_process`, `node:readline`,
`node:repl`, `node:tty`, `node:v8`, and `node:worker_threads`.
`node:perf_hooks` is implemented rather than stubbed.

Each import-only stub rollout has module-specific flags:
`enable_nodejs_<name>_module` enables it before its automatic date and
`disable_nodejs_<name>_module` disables it afterward. Omit the leading
underscore from the flag for `node:_stream_wrap`:

```jsonc
{
  "compatibility_flags": [
    "nodejs_compat",
    "enable_nodejs_sqlite_module",
    "disable_nodejs_stream_wrap_module"
  ]
}
```

## `process`, `require()`, and EOL APIs

From `2025-09-15`, process v2 replaces the small
`nextTick`/`env`/`exit`-oriented shim with a broader implementation.
Unsupported exports are `undefined`. `disable_nodejs_process_v2` retains the
old shim.

From `2026-01-22`, `require()` returns a module's default export when present;
otherwise it returns a mutable copy of the namespace object.
`require_returns_namespace` keeps the previous always-namespace result.

From `2025-09-01`, `nodejs_compat` rolls up removals of Node APIs that have
reached end of life, including version-specific removals such as Node.js 23
APIs. `add_nodejs_compat_eol` is a temporary migration escape hatch.

## Timers and performance

With `nodejs_compat` from `2026-02-10`, global timers return Node-compatible
`Timeout` objects. These expose `refresh()`, `ref()`, `unref()`, and
`hasRef()`.

From `2026-03-17`, globals also include `PerformanceEntry`,
`PerformanceMark`, `PerformanceMeasure`, `PerformanceResourceTiming`,
`PerformanceObserver`, and `PerformanceObserverEntryList`. Enabling
`node:perf_hooks` exposes these classes implicitly.

## Compatibility corrections

With `nodejs_compat` from `2026-05-19`, `Channel.hasSubscribers` and
`TracingChannel.hasSubscribers` in `node:diagnostics_channel` are read-only
boolean properties, not methods.

From `2026-06-16`, unsupported TLS options such as `checkServerIdentity`
passed to `tls.connect()` or `new TLSSocket()` throw
`ERR_OPTION_NOT_IMPLEMENTED` instead of being silently ignored.
