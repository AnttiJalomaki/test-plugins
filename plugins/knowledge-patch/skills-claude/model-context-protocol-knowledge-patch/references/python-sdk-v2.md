# Python SDK v2 migration and modern clients

Use this reference when moving a Python SDK application to v2, updating
protocol models and handlers, configuring transports, or adopting modern
multi-round and subscription behavior.

## Pin the prerelease dependency set

The v2 line is prerelease. Declare `mcp>=2,<3` only with prerelease resolution
enabled, and prefer an exact published prerelease pin for reproducible
application installs during the release-candidate period. An unqualified
`pip install mcp` still selects stable v1.x.

Let `mcp` exact-pin its matching standalone `mcp-types` package. Resolve raised
floors including `pydantic>=2.12` and `sse-starlette>=3`.

`mcp dev` and `mcp install` pin their temporary `uv` environment to the
installed SDK version instead of resolving an unrelated stable release.

Published libraries that require v1 semantics should cap below v2, for example:

```text
mcp>=1.27,<2
```

## Replace `httpx` with `httpx2`

The SDK removes `httpx` and `httpx-sse` in favor of `httpx2>=2.5`.
SDK-facing clients, authentication objects, mocks, exception handlers, and
`isinstance` checks must all use `httpx2` types; the packages are not
interchangeable.

```python
import httpx2

http_client = httpx2.AsyncClient(follow_redirects=True)
```

`httpx2` uses the operating-system CA store. Minimal images may need
`SSL_CERT_FILE`, `SSL_CERT_DIR`, or an explicit `ssl.SSLContext`.

## Update protocol model imports and names

Wire models live in the exact-versioned `mcp-types` distribution.
`mcp.types` remains a permanent same-object alias for SDK users.
Version constants move from `mcp.shared.version` to `mcp.types.version`.

Rename removed types:

| V1 | V2 |
| --- | --- |
| `Content` | `ContentBlock` |
| `ResourceReference` | `ResourceTemplateReference` |
| `Cursor` | `str` |

Python attributes are snake_case while JSON aliases remain camelCase.
Constructors accept either spelling, but self-serialized wire data must retain
aliases:

```python
schema = tool.input_schema
payload = tool.model_dump(by_alias=True, mode="json")
```

Unknown fields on protocol models are silently discarded rather than
round-tripped. Put extensions in `_meta`.

Server results and inbound client traffic are validated against the negotiated
revision. A value such as `Tool(input_schema={})` may construct but fail during
serialization because the schema must contain `"type": "object"`.

## Replace wrapped message unions

`ClientRequest`, `ServerNotification`, `JSONRPCMessage`, and other message
unions are no longer `RootModel` wrappers. Remove `.root`, construct concrete
members directly, and validate raw input with exported adapters:

```python
from mcp.types import jsonrpc_message_adapter

message = jsonrpc_message_adapter.validate_python(data)
```

`RequestParams.Meta` becomes the `RequestParamsMeta` `TypedDict`. Use
dictionary access such as `ctx.meta.get("progress_token")`. Notification
metadata is a plain dictionary.

## Treat resource URIs as strings

Resource URI fields and client methods use plain `str`. They accept relative
values, preserve exact spelling, and no longer provide `AnyUrl` parsing or
normalization.

Convert an existing `AnyUrl` with `str()` and explicitly parse a returned string
when components are needed.

## Choose the correct version constants

`LATEST_PROTOCOL_VERSION` means the newest revision in any era
(`2026-07-28`), which cannot be offered in the legacy initialize handshake.
Use `LATEST_HANDSHAKE_VERSION` or `HANDSHAKE_PROTOCOL_VERSIONS` for that
exchange.

`SUPPORTED_PROTOCOL_VERSIONS` is deprecated and now spans both handshake and
modern revisions.

## Raise `MCPError` correctly

`McpError` becomes `MCPError`, exported from `mcp`. It takes code, message, and
optional data directly instead of an `ErrorData` wrapper, and exposes `.code`,
`.message`, and `.data`:

```python
from mcp import MCPError
from mcp.types import INVALID_REQUEST

raise MCPError(INVALID_REQUEST, "bad input")
```

## Rename the high-level server and inject context

`FastMCP` and `mcp.server.fastmcp.*` become `MCPServer` and
`mcp.server.mcpserver.*`.

`get_context()` is removed. Request an injected `Context` in the handler and
use `ctx.mcp_server`, not `ctx.fastmcp`:

```python
from mcp.server.mcpserver import Context, MCPServer

mcp = MCPServer("demo")

@mcp.tool()
async def status(ctx: Context) -> str:
    return str(ctx.request_id)
```

## Construct and run servers explicitly

Keep only the server name positional because `title` and `description` now
precede `instructions`. An unnamed server reports `mcp-server`; an omitted
version reports `""`, not the SDK version.

Move transport settings such as `host`, `port`, `json_response`, and
`stateless_http` from the constructor to `run()` or the app factory.
`mount_path` is removed:

```python
mcp = MCPServer("demo", instructions="Use the weather tools.")
mcp.run(
    transport="streamable-http",
    host="0.0.0.0",
    port=9000,
    stateless_http=True,
)
```

Streamable HTTP bodies default to a 4 MiB limit, configurable through
`max_request_body_size`. Loopback app factories automatically enable
DNS-rebinding protection through `transport_security`.

The server lifespan enters once when the session manager starts and is shared
across sessions and stateless requests. Move per-connection acquisition into a
handler or another explicitly scoped resource.

## Account for concurrent handlers and progress

Synchronous high-level tools, resources, prompts, and resolvers run
concurrently on AnyIO worker threads. Use `async def` for event-loop-affine or
thread-affine work.

The `mcp.shared.progress` context manager is removed. Call
`ctx.report_progress(current, total, message)`; `current` is an absolute value,
not an accumulated delta.

## Handle high-level direct results

`MCPServer.call_tool()` returns `CallToolResult | InputRequiredResult`.
Direct prompt and resource calls can also return `InputRequiredResult` and
accept an optional explicit context.

Raising `MCPError` from `@mcp.tool()` produces a top-level JSON-RPC error with
its code and data intact. For a caller-readable tool execution failure, return
`CallToolResult(is_error=True, ...)` or raise another exception.

## Harden resources and templates

Resource subclasses reject unknown keyword arguments.
`FileResource.is_binary` becomes `encoding`:

- textual MIME types default to `"utf-8-sig"`;
- binary MIME types default to `None`;
- `encoding=None` explicitly forces a blob.

RFC 6570 matching is exact. It rejects decoded traversal, null-byte, and
absolute-path parameter values unless a parameter is explicitly exempted:

```python
ResourceSecurity(exempt_params={"target"})
```

Use exemptions narrowly and validate the exempted value in application code.

## Rewrite low-level handlers

Low-level `Server` decorators become constructor `on_*` callbacks. Each
receives `(ctx, params)` and returns the complete protocol result:

```python
from mcp.server import Server
from mcp.types import CallToolResult, TextContent

async def call_tool(ctx, params) -> CallToolResult:
    return CallToolResult(
        content=[TextContent(type="text", text=params.name)]
    )

server = Server("demo", on_call_tool=call_tool)
```

Automatic tool-argument validation and automatic wrapping of dictionaries,
lists, strings, bytes, and exceptions are gone. `params.arguments` may be
`None`; validate it against the advertised input schema and apply defaults in
the handler.

Do not rely on generic exception conversion. A generic low-level tool callback
failure can surface as a top-level error with code `0`, while an unexpected
dispatcher failure is sanitized to `-32603` with `Internal server error`.
Raise `MCPError` for a deliberate wire error. Catch tool execution failures and
return `CallToolResult(is_error=True, ...)` when the caller should inspect them.

## Add custom methods and middleware

Register a custom or post-construction method with:

```python
add_request_handler(method, params_type, handler)
```

Unlike built-in tool dispatch, this validates inbound parameters against the
supplied model.

A provisional `middleware` list can wrap every inbound message. Use it instead
of overriding private `_handle_*` methods.

## Use explicit request contexts

Ambient `server.request_context` and
`mcp.shared.context.RequestContext` are removed. Use explicit
`ServerRequestContext` or `ClientRequestContext`.

Each inbound message receives a fresh `ServerSession` proxy. Never key
connection state by `ctx.session` object identity. Notifications after a
connection closes are silently dropped rather than raising.

## Connect with the high-level client

`Client` accepts:

- an in-process server;
- a Streamable HTTP URL;
- a transport context manager such as `stdio_client(...)`.

Entering it with `async with` connects and negotiates. `client.session` exposes
the underlying `ClientSession`.

The default `mode="auto"` probes `server/discover`, selects the modern era when
available, and falls back to initialization. Use `mode="legacy"` when the
legacy handshake and back-channel semantics are required.

`Client(server)` replaces
`create_connected_server_and_client_session()` and defaults to direct modern
dispatch, not v1's in-memory JSON-RPC transport.

Registered sampling and elicitation callbacks can also fulfill requests
embedded in modern results.

## Migrate modern back-channel flows

On a modern connection, push-style `ctx.elicit()`, sampling, roots, and other
server requests raise `NoBackChannelError`, including over stdio and in-process
dispatch.

Pin legacy temporarily, or use `Resolve(...)` with `Elicit`, `Sample`, or
`ListRoots`. In the modern era those values become an
`InputRequiredResult` round trip.

Annotating a tool parameter with `Resolve(fn)` supplies the resolver's value
without exposing that parameter in the tool schema. A resolver can return
`Elicit(...)`; the SDK issues live elicitation for a legacy connection and
uses a multi-round result for a modern connection, so one tool body can serve
both.

## Update client metadata, pagination, and timeouts

Read negotiated values from properties including:

- `session.server_capabilities`;
- `server_info`;
- `instructions`;
- `protocol_version`.

`get_server_capabilities()` is removed.

Low-level list methods take
`params=PaginatedRequestParams(cursor=...)`. Former `timedelta` timeouts become
plain floats in seconds:

```python
from mcp.types import PaginatedRequestParams

page = await session.list_tools(
    params=PaginatedRequestParams(cursor=cursor)
)
result = await session.call_tool(
    "slow", {}, read_timeout_seconds=30.0
)
```

A timed-out call raises `MCPError` with `REQUEST_TIMEOUT` (`-32001`), not HTTP
408. Cancelling or timing out an awaited request sends cancellation to the
peer.

Client callbacks and notifications run concurrently under
`JSONRPCDispatcher`; coordinate callbacks that require ordering.

## Migrate the Streamable HTTP client

`streamablehttp_client` is removed. Use
`streamable_http_client(url, *, http_client=..., terminate_on_close=...)`,
which yields only `(read_stream, write_stream)`.

Configure headers, authentication, and timeouts on an
`httpx2.AsyncClient`. `StreamableHTTPTransport` likewise takes only its URL.
The `get_session_id` callback is removed; capture `mcp-session-id` with an
`httpx2` response hook only when needed.

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
        url, http_client=http
    ) as (read, write):
        ...
```

A non-2xx POST fails only its request as `MCPError`, preserves a JSON-RPC error
body when present, and leaves the connection usable. Status failures no longer
escape the transport context as `httpx.HTTPStatusError` inside an
`ExceptionGroup`. Connect-level `httpx2` failures still escape.

## Account for stdio process changes

`stdio_server()` moves protocol traffic to private descriptors, redirects file
descriptor 0 to the null device, and redirects file descriptor 1 to stderr.
Handlers and child processes therefore cannot consume or corrupt the wire.

On POSIX, `stdio_client()` no longer kills children left behind when a server
exits gracefully. The server owns their cleanup.

## Use the new OAuth providers

The callback returns `AuthorizationCodeResult(code, state, iss)`;
client-credentials providers use singular `scope=`; and the deprecated
RFC 7523 provider and `JWTParameters` are removed. Refresh-token defaults and
exact-URL behavior also changed.

See [authorization.md](authorization.md) for provider selection, issuer
validation, refresh defaults, and redirect re-registration.

## Enable tracing deliberately

Every outbound request contains a `_meta` envelope. When a global
OpenTelemetry provider is configured, the SDK starts SDK spans and injects
trace context without additional setup.

Confirm exporters, sampling, and sensitive-attribute policy before enabling a
global provider in production.

## Handle visible deprecation warnings

Deprecated calls emit `MCPDeprecationWarning`, a `UserWarning`, on every use.
Test suites that turn warnings into errors should keep that category explicitly
non-fatal while a necessary legacy call remains, then remove the exception with
the call.

## Use subscriptions and caching

`MCPServer` serves `subscriptions/listen` automatically. Publish with helpers
such as:

```python
await ctx.notify_resource_updated(uri)
```

Provide a shared `SubscriptionBus` for delivery across replicas. Clients
consume typed events with `async with client.listen(...)`; `sub.honored`
reports which requested filters the server accepted.

Modern HTTP sends `Mcp-Method`; the three named tool-like calls also send
`Mcp-Name`. A tool schema property marked `x-mcp-header` mirrors its argument
to `Mcp-Param-*`, and the server cross-checks it against the body.

Servers set list/read freshness per method with `cache_hints=`. The high-level
client honors `ttlMs` and `cacheScope` through its built-in response cache.
Servers without hints, including pre-modern peers, remain uncached.

## Remove unsupported APIs

V2 removes both WebSocket transports and the `mcp[ws]` extra.

It also removes `mcp.*.experimental` Tasks APIs. Tasks moved from core protocol
behavior to an official extension, and this SDK does not yet implement that
extension.
