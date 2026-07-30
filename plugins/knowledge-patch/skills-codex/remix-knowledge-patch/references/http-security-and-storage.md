# HTTP, security, and storage

## Enable proxy trust deliberately

`remix/node-fetch-server` accepts `trustProxy`. When enabled, reconstructed
`request.url` and handler client-address data honor `Forwarded` or
`X-Forwarded-*`.

Enable it only behind a trusted reverse proxy that controls or sanitizes those
headers. Leave it disabled when clients can supply them directly.

## Preserve duplicate cookies

`Cookie` and `SuperHeaders.cookie` retain same-name cookies in insertion order:

- `get(name)` returns the first match.
- `getAll(name)` returns every match.
- `append(name, value)` adds a match without replacing existing values.

`SuperHeaders.apply(init)` applies a `SuperHeadersInit` to an existing instance
with header-aware behavior.

Header-specific APIs are also available from:

- `remix/headers/cookie`
- `remix/headers/set-cookie`
- `remix/headers/cache-control`
- `remix/headers/range`

Use those semantics rather than normalizing cookies into a one-value map.

## Read multipart part headers as objects

Both multipart-parser entrypoints return `MultipartPart.headers` as a plain
object. Keys are lower-case header names; the value is not a native
`Headers`:

```ts
const contentType = part.headers["content-type"];
```

Do not call `headers.get(...)` on a multipart part.

## Choose request protection by threat model

Browser-origin protection and session-backed CSRF are separate public APIs:

- Configure tokenless cross-origin protection with `cop(options)`.
- Configure session-backed CSRF with `csrf(options)`.
- Read the session-backed token with `getCsrfToken(context)`.

They can be used independently or layered with session middleware. Do not
assume enabling one implicitly configures the other.

## Compose authentication and request middleware

The standalone package set includes:

- Browser login helpers.
- OAuth/OIDC helpers.
- Pluggable authentication.
- Request context backed by `AsyncLocalStorage`.
- Compression, CORS, and CSRF middleware.
- Tokenless cross-origin protection.
- Form-data parsing with streaming uploads.
- Logging, method override, and static-file serving.

Router middleware still follows the explicit continuation contract: each
middleware branch must return a `Response` or return `next()`.

## Keep storage portable

File storage uses JavaScript `File` values rather than runtime-specific file
objects. An S3 storage backend is available.

The session stack provides cookie-based middleware and storage adapters for
Memcache and Redis. Domain-oriented imports include paths such as
`remix/file-storage/s3` and `remix/session-storage/redis`.

Use Web Streams, `Uint8Array`, Web Crypto, and `Blob`/`File` at storage and
upload boundaries when the runtime provides them.

