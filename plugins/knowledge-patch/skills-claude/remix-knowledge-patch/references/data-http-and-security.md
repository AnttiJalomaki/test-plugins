# Data, HTTP, and security

Use this reference for relational data queries, storage adapters, structured
headers, multipart uploads, authentication middleware, origin protection, and
proxy-derived request information.

## Data-table adapters and query API

The typed relational `data-table` toolkit has adapters for:

- MySQL;
- PostgreSQL; and
- SQLite.

Import adapter APIs from domain subpaths such as `remix/data-table/postgres`.

`SqlStatement`, `sql`, and `rawSql` moved out of `remix/data-table/sql`; import
them from `remix/data-table`. `Database` is now a constructible runtime class.

`QueryBuilder` was removed. Use `Query` or `query`. Pass either raw SQL or a
`Query` value to `db.exec(...)`.

Unbound terminal operations produce query objects rather than immediately
executing. This applies to methods including:

- `first()`;
- `count()`;
- `insert()`; and
- `update()`.

Account for that deferred shape when migrating code that previously expected a
terminal call to produce a row, count, or mutation result immediately.

## File and session storage

File storage uses standard JavaScript `File` objects and includes an S3 backend.
Avoid designing a storage abstraction around a runtime-specific file handle
when the framework boundary expects `File`.

The session stack includes cookie-based middleware and storage adapters for:

- Memcache; and
- Redis.

Use domain imports such as `remix/file-storage/s3` and
`remix/session-storage/redis` rather than relying on a root export.

## Duplicate cookie behavior

`Cookie` and `SuperHeaders.cookie` preserve same-name cookies in arrival order:

- `get(name)` returns the first matching value;
- `getAll(name)` returns all matching values; and
- `append(name, value)` adds another cookie without replacing an existing one.

Do not model cookies as a simple map when repeated names carry meaning.

`SuperHeaders.apply(init)` applies a `SuperHeadersInit` to an existing instance
using header-aware behavior. Header-specific APIs are available through
subpaths including:

- `remix/headers/cookie`;
- `remix/headers/set-cookie`;
- `remix/headers/cache-control`; and
- `remix/headers/range`.

## Multipart part headers

`MultipartPart.headers` from both multipart-parser entrypoints is a plain
object. Its keys are lower-case header names. It is not a native `Headers`
instance:

```ts
const contentType = part.headers["content-type"];
```

Migrate `part.headers.get("content-type")` calls to property access, and keep
custom lookups lower-case.

## Request-handling middleware package set

The composable package family includes middleware and helpers for:

- browser login and OAuth/OIDC;
- pluggable authentication;
- request context backed by `AsyncLocalStorage`;
- compression;
- tokenless cross-origin protection;
- CORS;
- CSRF;
- form-data parsing with streaming uploads;
- logging;
- method override; and
- static-file serving.

These capabilities are standalone and composable. Select the narrow package
needed at each boundary rather than assuming one monolithic server runtime.

## Browser-origin protection and CSRF

Configure tokenless browser-origin protection with `cop(options)`. Configure
session-backed CSRF with `csrf(options)` and retrieve a token with
`getCsrfToken(context)`.

The two mechanisms can be used independently or layered with session
middleware. Do not assume enabling one implicitly enables the other:

- use `cop` for browser-origin checks that do not depend on a session token;
- use `csrf` when the request must carry a session-backed token; and
- compose both when the threat model calls for both checks.

## Trusted reverse proxies

`remix/node-fetch-server` has a `trustProxy` option. When enabled, construction
of `request.url` and the client-address data passed to the handler can honor
`Forwarded` or `X-Forwarded-*` headers.

Enable it only when the server is behind a trusted reverse proxy that controls
or sanitizes those headers. Leave it disabled for direct internet traffic or
when untrusted clients can supply forwarding headers.

## Security review checklist

1. Confirm whether authentication, browser-origin protection, and CSRF are
   separate layers in the actual middleware chain.
2. Ensure a session-backed CSRF flow can call `getCsrfToken(context)` where the
   response form or client needs the token.
3. Preserve duplicate cookies and use `getAll()` where repeated values matter.
4. Read multipart headers from a lower-case-keyed object.
5. Keep streaming uploads as streams instead of buffering them through a
   runtime-specific API without need.
6. Enable `trustProxy` only at a verified proxy boundary.
7. Verify data-table terminal calls are executed where a result, rather than a
   `Query`, is required.
