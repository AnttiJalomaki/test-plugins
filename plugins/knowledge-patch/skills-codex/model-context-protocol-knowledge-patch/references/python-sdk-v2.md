# Python SDK v2

## Dependency boundary

The v2 line is prerelease. Install `mcp>=2,<3` with an explicit prerelease
version; unqualified `pip install mcp` still selects stable v1.x during the
release-candidate period. Published libraries that need v1 behavior should cap
the dependency, for example:

```text
mcp>=1.27,<2
```

Let `mcp` exact-pin its matching standalone `mcp-types` distribution. Resolve
the new floors, including `pydantic>=2.12` and `sse-starlette>=3`.
`mcp dev` and `mcp install` pin their temporary `uv` environments to the
installed SDK version instead of resolving an unrelated stable version.

## `httpx2` migration

V2 replaces `httpx` and `httpx-sse` with `httpx2>=2.5`. Convert SDK-facing
clients, auth objects, mocks, exception handlers, and `isinstance` checks.
Values from the two packages are not interchangeable.

```python
import httpx2

http_client = httpx2.AsyncClient(follow_redirects=True)
```

`httpx2` uses the operating-system CA store. Minimal containers may need
`SSL_CERT_FILE`, `SSL_CERT_DIR`, or an explicit `ssl.SSLContext`.

## Protocol models and serialization

Wire models live in `mcp-types`; `mcp.types` is a permanent same-object alias
for SDK users.

Renamed or removed types include:

- `Content` becomes `ContentBlock`.
- `ResourceReference` becomes `ResourceTemplateReference`.
- `Cursor` becomes plain `str`.
- Version constants move from `mcp.shared.version` to `mcp.types.version`.

Python attributes are snake_case while JSON aliases remain camelCase.
Constructors accept either spelling. When producing wire data yourself, retain
the aliases:

```python
schema = tool.input_schema
payload = tool.model_dump(by_alias=True, mode="json")
```

Unknown fields are discarded rather than round-tripped. Put extensions in
`_meta`.

Protocol models are validated against the negotiated revision when a server
returns them or a client receives them. Construction alone is not proof of
wire validity: `Tool(input_schema={})` constructs but fails during serialization
because the schema requires `"type": "object"`.

### URI fields

Resource URI fields and client methods use `str`. They accept relative values,
preserve exact spelling, and do not expose `AnyUrl` components or
normalization. Convert old values with `str()` and parse returned strings
explicitly when components are needed.

### Message unions

`ClientRequest`, `ServerNotification`, `JSONRPCMessage`, and related unions are
no longer `RootModel` wrappers. Remove `.root`, construct a concrete union
member directly, and validate raw data with an exported adapter:

```python
from mcp.types import jsonrpc_message_adapter

message = jsonrpc_message_adapter.validate_python(data)
```

### Metadata dictionaries

`RequestParams.Meta` becomes the separate `RequestParamsMeta` `TypedDict`.
Use dictionary access, such as `ctx.meta.get("progress_token")`. Notification
metadata is a plain dictionary.

### Version constants

`LATEST_PROTOCOL_VERSION` means the newest revision in any era, currently the
modern revision, and cannot be offered in a legacy initialize handshake.
Use `LATEST_HANDSHAKE_VERSION` or `HANDSHAKE_PROTOCOL_VERSIONS` there.
`SUPPORTED_PROTOCOL_VERSIONS` is deprecated and is now a tuple spanning both
handshake and modern revisions.

## Errors

`McpError` becomes `MCPError`, exported from `mcp`. Construct it directly from
code, message, and optional data; do not wrap them in `ErrorData`.

```python
from mcp import MCPError
from mcp.types import INVALID_REQUEST

raise MCPError(INVALID_REQUEST, "bad input")
```

The exception exposes `.code`, `.message`, and `.data`.

Timed-out client calls raise `MCPError` with `REQUEST_TIMEOUT` (`-32001`), not
an HTTP 408. Cancelling or timing out an awaited request sends cancellation to
the peer.

For a high-level `@mcp.tool()`, raising `MCPError` produces a top-level
JSON-RPC error with its code and data. Return
`CallToolResult(is_error=True, ...)` or raise another exception when the failure
should be a model-visible tool execution result.

An unexpected low-level exception is sanitized to `-32603` and
`Internal server error`. Raise `MCPError` for a deliberate wire failure.

## High-level server migration

`FastMCP` and `mcp.server.fastmcp.*` become `MCPServer` and
`mcp.server.mcpserver.*`. Ambient `get_context()` is removed. Declare an
injected `Context` and use `ctx.mcp_server` instead of `ctx.fastmcp`.

```python
from mcp.server.mcpserver import Context, MCPServer

mcp = MCPServer("demo")

@mcp.tool()
async def status(ctx: Context) -> str:
    return str(ctx.request_id)
```

### Construction and transport configuration

Keep only the server name positional because `title` and `description` now
precede `instructions`. An unnamed server reports `mcp-server`; omitted version
reports `""`, not the SDK version.

Move `host`, `port`, `json_response`, and `stateless_http` from the constructor
to `run()` or the app factory. `mount_path` is removed.

```python
mcp = MCPServer("demo", instructions="Use the weather tools.")
mcp.run(
    transport="streamable-http",
    host="0.0.0.0",
    port=9000,
    stateless_http=True,
)
```

### Streamable HTTP lifecycle and limits

Request bodies default to a 4 MiB limit, configurable with
`max_request_body_size`. Loopback app factories enable DNS-rebinding protection
through `transport_security`.

The server lifespan enters once when the session manager starts and is shared
across sessions and stateless requests. Move per-connection acquisition into a
handler.

### Scheduling and progress

Synchronous high-level tools, resources, prompts, and resolvers run
concurrently in AnyIO worker threads. Use `async def` for event-loop-affine or
thread-affine work.

The removed `mcp.shared.progress` context manager becomes:

```python
await ctx.report_progress(current, total, message)
```

`current` is an absolute value, not a delta.

### Direct calls

`MCPServer.call_tool()` returns `CallToolResult | InputRequiredResult`.
Direct prompt and resource calls can also return `InputRequiredResult` and
accept an optional explicit context.

### Resources and templates

Resource subclasses reject unknown keyword arguments.
`FileResource.is_binary` becomes `encoding`:

- Text MIME types default to `"utf-8-sig"`.
- Binary MIME types default to `None`.
- `encoding=None` explicitly forces a blob.

RFC 6570 template matching is exact. It rejects decoded traversal, null-byte,
and absolute-path parameter values. Exempt a deliberately unrestricted
parameter explicitly:

```python
ResourceSecurity(exempt_params={"target"})
```

## Resolver-injected parameters

Annotate a tool parameter with `Resolve(fn)` to supply it without exposing the
parameter to the caller. A resolver may return `Elicit(...)`.

On a legacy connection the SDK uses a live elicitation request. On a modern
connection it uses a multi-round result. The same tool body can therefore
serve both eras. `Resolve(...)` similarly supports `Sample` and `ListRoots`
when migrating back-channel behavior.

## Low-level server redesign

Replace low-level `Server` decorators with constructor `on_*` callbacks.
Callbacks receive `(ctx, params)` and return the full protocol result.
Automatic argument validation and automatic wrapping of dictionaries, lists,
strings, bytes, and exceptions are gone.

```python
from mcp.server import Server
from mcp.types import CallToolResult, TextContent

async def call_tool(ctx, params) -> CallToolResult:
    return CallToolResult(
        content=[TextContent(type="text", text=params.name)]
    )

server = Server("demo", on_call_tool=call_tool)
```

The low-level server does not validate tool arguments against advertised
`inputSchema`; `params.arguments` may be `None`. Validate and default arguments
in the handler.

A generic low-level tool exception becomes a top-level JSON-RPC error with
code `0`. Catch it and return `CallToolResult(is_error=True, ...)` when the
failure should be readable as a tool result.

Register a custom or post-construction method with:

```python
add_request_handler(method, params_type, handler)
```

Unlike built-in tool dispatch, this validates parameters against the supplied
model. A provisional `middleware` list can wrap every inbound message; do not
override private `_handle_*` methods.

## Explicit request contexts

Remove ambient `server.request_context` and
`mcp.shared.context.RequestContext`. Use explicit `ServerRequestContext` or
`ClientRequestContext`.

Each inbound message gets a fresh `ServerSession` proxy. Never use
`ctx.session` identity as a connection-state key. Notifications sent after
connection closure are silently dropped rather than raising.

## Client connection and negotiation

The high-level `Client` accepts:

- An in-process server.
- A Streamable HTTP URL.
- A transport context manager such as `stdio_client(...)`.

Entering it with `async with` connects and negotiates. `client.session` exposes
the underlying `ClientSession`. Registered sampling and elicitation callbacks
also fulfill modern requests embedded in results.

`Client` defaults to `mode="auto"`, probes `server/discover`, and selects the
modern era before falling back to initialize. Use `mode="legacy"` when the old
handshake and a push back-channel are required.

`Client(server)` replaces `create_connected_server_and_client_session()` and
defaults to direct modern dispatch rather than the v1 in-memory JSON-RPC
transport.

On a modern connection, push-style `ctx.elicit()`, sampling, roots, and other
server requests raise `NoBackChannelError`, including over stdio or in-process
dispatch. Temporarily pin legacy behavior or migrate to `Resolve(...)`.

## Client metadata, pagination, and timeouts

Negotiated data is exposed as properties:

- `session.server_capabilities`
- `server_info`
- `instructions`
- `protocol_version`

`get_server_capabilities()` is removed.

Low-level list methods take
`params=PaginatedRequestParams(cursor=...)`. Timeout values formerly expressed
as `timedelta` are float seconds.

```python
from mcp.types import PaginatedRequestParams

page = await session.list_tools(
    params=PaginatedRequestParams(cursor=cursor)
)
result = await session.call_tool(
    "slow",
    {},
    read_timeout_seconds=30.0,
)
```

Client callbacks and notifications run concurrently under
`JSONRPCDispatcher`. Coordinate explicitly when callback order matters.

## Streamable HTTP client

The deprecated `streamablehttp_client` is removed. The underscored
`streamable_http_client(url, *, http_client=..., terminate_on_close=...)`
yields only `(read_stream, write_stream)`.

Configure headers, authorization, and timeouts on an `httpx2.AsyncClient`.
`StreamableHTTPTransport` likewise takes only its URL.

```python
import httpx2
from mcp.client.streamable_http import streamable_http_client

timeout = httpx2.Timeout(30, read=300)
async with httpx2.AsyncClient(
    timeout=timeout,
    auth=auth,
    follow_redirects=True,
) as http:
    async with streamable_http_client(
        url,
        http_client=http,
    ) as (read, write):
        ...
```

The `get_session_id` callback is removed. If necessary, capture
`mcp-session-id` through an `httpx2` response hook.

A non-2xx POST fails only that request as `MCPError`, preserves an in-band
JSON-RPC error body when available, and leaves the connection usable. Status
failures no longer escape the transport as `httpx.HTTPStatusError` inside an
`ExceptionGroup`; connection-level `httpx2` failures still escape.

## stdio process behavior

`stdio_server()` moves protocol traffic onto private descriptors, redirects
file descriptor 0 to the null device, and redirects file descriptor 1 to
standard error. Handlers and child processes therefore cannot consume or
corrupt the protocol stream.

On POSIX, `stdio_client()` no longer kills grandchildren left behind after a
server exits cleanly. The server owns their cleanup.

## OAuth migration

`OAuthClientProvider.callback_handler` returns
`AuthorizationCodeResult(code, state, iss)`, permitting issuer validation.
Client-credentials providers rename `scopes=` to `scope=`.

Remove `RFC7523OAuthClientProvider` and `JWTParameters`. Choose the grant's
actual provider:

- `ClientCredentialsOAuthProvider`
- `PrivateKeyJWTOAuthProvider`
- `IdentityAssertionOAuthProvider`

When the server supports refresh tokens and they are enabled, the client adds
`offline_access` and `prompt=consent`. To keep no-refresh behavior, set
`grant_types=["authorization_code"]`.

Pathless OAuth metadata and redirect URLs no longer gain a trailing slash.
Re-register a persisted client whose exact redirect URI changed. Compare
protected-resource and issuer URLs exactly.

## Automatic tracing

Every outbound request has a `_meta` envelope. When a global OpenTelemetry
provider is configured, the SDK records spans and injects trace context without
additional SDK setup.

## Deprecation warnings

Deprecated calls emit `MCPDeprecationWarning`, a `UserWarning`, on every use.
Warning-as-error test suites should keep this category non-fatal only while a
legacy API is intentionally required.

## Removed transports and Tasks APIs

V2 removes WebSocket transports and the `mcp[ws]` extra.
It also removes `mcp.*.experimental` Tasks APIs. Tasks moved from core to an
official extension that this SDK does not yet implement.

## Subscriptions

`MCPServer` serves `subscriptions/listen` automatically. Publish changes with
helpers such as:

```python
await ctx.notify_resource_updated(uri)
```

Provide a shared `SubscriptionBus` for delivery across replicas. Clients
consume typed events with:

```python
async with client.listen(...) as sub:
    ...
```

`sub.honored` reports which requested filters the server accepted.

## Modern HTTP routing and caching

Modern requests carry `Mcp-Method`; the three tool-like calls also carry
`Mcp-Name`. A tool property marked `x-mcp-header` is mirrored into an
`Mcp-Param-*` header and checked against the request body.

Servers configure list/read freshness by method through `cache_hints=`.
The high-level client honors `ttlMs` and `cacheScope` with its built-in response
cache. Missing hints, including from pre-modern servers, mean uncached
behavior.
