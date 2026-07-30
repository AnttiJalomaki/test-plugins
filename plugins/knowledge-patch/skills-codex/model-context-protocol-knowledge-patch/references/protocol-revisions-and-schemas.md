# Protocol revisions and schemas

## Select an era before dispatch

The protocol has a legacy handshake era and a modern discovery era. A client
or server must use one codec and method registry for its negotiated revision;
do not select method semantics independently per message.

### Legacy lifecycle

Legacy peers use `initialize` and `notifications/initialized`. The lifecycle
operation requirement became **MUST** in `2025-06-18`; implementations targeting
that revision cannot treat it as optional.

After initialization, every Streamable HTTP request sends the negotiated
`MCP-Protocol-Version`. In the absence of other version information, a missing
header on the older transport means `2025-03-26`. Invalid or unsupported
values produce HTTP 400.

### Modern discovery and stateless metadata

The `2026-07-28-rc` removes the initialization handshake and protocol-level
sessions. Servers implement:

```json
{"jsonrpc":"2.0","id":1,"method":"server/discover"}
```

`server/discover` returns supported protocol versions, capabilities, and
identity. A client can use it before any other call to select a revision or as
a backward-compatibility probe over stdio.

Every modern request carries these `_meta` entries:

- Required `io.modelcontextprotocol/protocolVersion`.
- Required `io.modelcontextprotocol/clientCapabilities`.
- Recommended `io.modelcontextprotocol/clientInfo`.

Every server response should carry `io.modelcontextprotocol/serverInfo`.
Version mismatch raises `UnsupportedProtocolVersionError`.

List endpoints do not vary by connection. A server needing state across calls
must issue an explicit application handle and receive it as an ordinary tool
argument.

Self-reported server identity is for display and diagnostics, never for
security or behavior decisions.

## JSON-RPC framing and batching

The `2025-03-26` revision added top-level JSON-RPC arrays for batching.
The `2025-06-18` revision removed them. For that and later revisions, a
top-level array is invalid and each request, response, or notification travels
as a separate JSON-RPC message.

Streamable HTTP specifically accepts one JSON-RPC request, notification, or
response per POST body. Do not combine them even when interoperating with an
earlier peer that understood batch syntax elsewhere.

## Metadata surfaces

The `2025-06-18` schemas add `_meta` to additional interface types. Validators
and generated bindings must permit metadata on all newly covered shapes.
Extension data belongs in `_meta`, especially in SDKs that discard unknown
protocol-model fields.

Modern OpenTelemetry propagation uses conventional keys in `_meta`:

```json
{
  "_meta": {
    "traceparent": "00-...",
    "tracestate": "vendor=value",
    "baggage": "tenant=example"
  }
}
```

Request-scoped logging uses
`_meta["io.modelcontextprotocol/logLevel"]`. A modern server must not emit
`notifications/message` for a request that omitted this field.

## Result envelopes

Every modern result has a wire-level `resultType`:

- `"complete"` for an ordinary final result.
- `"input_required"` for a result carrying `inputRequests`.

A client reading an older protocol result interprets missing `resultType` as
complete. A modern response missing it is invalid.

SDKs may consume `resultType` before exposing a public result object.
Likewise, reserved retry fields can be lifted from parameters into handler
context, including:

- The request envelope.
- `inputResponses`.
- Parsed or verified `requestState`.

Do not write application handlers that depend on these reserved wire fields
remaining in the ordinary parameter object.

## Removed operations and method availability

Modern MCP removes:

- `ping`
- `logging/setLevel`
- `notifications/roots/list_changed`

It also replaces push-style roots, sampling, and elicitation with multi-round
results; moves Tasks to an extension; and moves list-change and resource
subscription traffic to `subscriptions/listen`.

Sending a spec method absent from the negotiated era should fail locally when
the SDK knows the registry. An inbound deleted method returns JSON-RPC
`-32601`, even if application code attempted to register a handler with the
same method name.

## Deprecated protocol features

The modern RC formally deprecates Roots, Sampling, Logging, HTTP+SSE, and
Sampling `includeContext` values `"thisServer"` and `"allServers"`.

Use:

- Tool parameters, resource URIs, or server configuration instead of Roots.
- Direct provider APIs instead of Sampling.
- Standard error or OpenTelemetry instead of Logging.
- Streamable HTTP instead of HTTP+SSE.
- Omitted `includeContext` or `"none"` instead of the contextual values.

Deprecated features can remain during their compatibility window, but new
implementations should not adopt them.

## JSON Schema behavior

JSON Schema 2020-12 is the default MCP dialect since `2025-11-25`. Producers,
validators, and generators should use it unless a different dialect is
selected explicitly.

The modern protocol allows any JSON Schema 2020-12 keyword in tool
`inputSchema` and `outputSchema`. Implementations must:

- Resolve `$ref`.
- Support composition keywords.
- Enforce resource limits while resolving and validating.
- Accept any JSON value in `structuredContent`, not only objects.

Generated `schema.json` definitions for minimum, maximum, and default values
accept numbers, rather than integers only.

Request parameter schemas are standalone definitions decoupled from RPC method
definitions. Generated bindings and schema consumers must resolve this
organization instead of assuming every parameter schema is nested under its
method.

## Cacheable modern results

These modern results implement `CacheableResult`:

- `tools/list`
- `prompts/list`
- `resources/list`
- `resources/read`
- `resources/templates/list`

They carry:

```json
{"ttlMs":30000,"cacheScope":"private"}
```

`ttlMs` is a freshness hint and `cacheScope` is either `"public"` or
`"private"`. Keep `tools/list` output deterministic. Servers should use zero
TTL and private scope as conservative defaults unless they can state a safer
policy. Legacy responses do not receive these fields.

Clients must partition private entries correctly, honor expiry, and avoid
inventing caching when a pre-modern server omits hints.

## Error-code allocation

The modern draft uses:

- `-32602` for resource-not-found, replacing `-32002`.
- `-32000` through `-32019` for implementation-defined server errors.
- `-32020` through `-32099` for MCP-reserved errors.
- `-32020` for `HeaderMismatch`.
- `-32021` for `MissingRequiredClientCapability`.
- `-32022` for `UnsupportedProtocolVersion`.

`HeaderMismatchError` is part of the schema. HTTP handlers may map
missing-capability error `-32021` to HTTP 400 when dispatch has not committed a
stream; other handler protocol errors normally remain in-band in an HTTP 200
response. Once an SSE response is committed, its error remains in that stream.

A proxy must translate downstream capability requirements into requirements
for its own hop rather than relaying a downstream
`requiredCapabilities` value unchanged.

## Compatibility checklist

Before emitting or validating a message:

1. Resolve the connection's protocol era and exact revision.
2. Select that era's method registry and schema set.
3. Apply its lifecycle or discovery rule.
4. Enforce its transport framing.
5. Strip or lift reserved envelope fields before application dispatch.
6. Validate results against the negotiated schema, not merely the newest
   installed protocol types.
7. Keep deprecated wire support isolated from new application APIs.
