---
name: model-context-protocol-knowledge-patch
description: Model Context Protocol (MCP)
version: 2025-11-25
license: MIT
metadata:
  author: Nevaberry
---



# Model Context Protocol Compatibility

Load this skill when implementing or reviewing MCP clients, servers, transports,
authorization, elicitation, sampling, tasks, schemas, or revision negotiation.
Choose a protocol revision before selecting message shapes or capabilities, and
use the negotiated revision rather than assuming that newer features are safe.

## Reference index

| Reference | Topics |
| --- | --- |
| [authorization.md](references/authorization.md) | OAuth profile, discovery, registration, tokens, scopes, and HTTP security |
| [http-and-subscriptions.md](references/http-and-subscriptions.md) | Streamable HTTP, SSE, sessions, cancellation, resumption, and legacy fallback |
| [interaction-patterns.md](references/interaction-patterns.md) | Elicitation, sampling, tools, tasks, progress, completion, audio, and links |
| [protocol-core.md](references/protocol-core.md) | Lifecycle, batching, metadata, names, schemas, validation, and implementation metadata |

## Breaking changes and deprecations

### Do not batch messages after 2025-03-26

JSON-RPC batching existed in the `2025-03-26` revision but was removed in
`2025-06-18`. For `2025-06-18` and later, never send a top-level array; send
each request, notification, or response as a separate JSON-RPC message.
Streamable HTTP POST already requires exactly one message per body.

### Treat lifecycle and protocol-version handling as mandatory

For `2025-06-18`, perform the required lifecycle operation. After
initialization, attach the negotiated `MCP-Protocol-Version` to every HTTP
request. A missing version is interpreted as `2025-03-26` when there is no
other version information; an invalid or unsupported version returns HTTP 400.

### Prefer Streamable HTTP over HTTP+SSE

Use one MCP endpoint for POST and optional GET. Retain the legacy SSE and POST
endpoints only when old-client compatibility is required. When probing an
unknown URL, fall back to legacy HTTP+SSE only if the initial Streamable HTTP
POST fails with 400, 404, or 405; other 4xx responses are not transport probes.

### Follow the current registration preference

For clients without a prior relationship, prefer a Client ID Metadata Document
when the authorization server advertises support. Dynamic Client Registration
is an optional compatibility fallback, followed by user-entered credentials.
The metadata-document URL is the `client_id`, and the served document must
repeat that exact value and declare `client_name` and `redirect_uris`.

### Omit sampling context unless explicitly negotiated

`includeContext: "thisServer"` and `"allServers"` are soft-deprecated in
`2025-11-25`. Omit `includeContext` for its `"none"` default. Send an old value
only when the client advertises `sampling: {context: {}}`.

### Classify tool argument failures as execution errors

As of `2025-11-25`, input-validation failures from `tools/call` are Tool
Execution Errors, not Protocol Errors. Return them where the caller can inspect
the failure and correct the arguments.

## Streamable HTTP quick reference

### POST requests

For each client message, issue a fresh POST with:

```http
Accept: application/json, text/event-stream
Content-Type: application/json
```

Return an empty HTTP 202 for an accepted notification or response-only input.
For request input, return either one JSON response or an SSE stream; clients
must implement both response forms.

### GET streams

A client may GET the same endpoint with `Accept: text/event-stream` for
server-initiated traffic. Return 405 when this stream is unsupported. Do not put
ordinary JSON-RPC responses on it except when replaying a prior request stream.

### Sessions and cancellation

If initialization returns `Mcp-Session-Id`, repeat it on every later HTTP
request. A required missing ID returns 400; an expired or terminated ID returns
404, after which initialize again without an ID. Request cleanup with DELETE,
while accepting that a server may return 405.

A disconnected SSE response does not cancel its request. Send an explicit
`CancelledNotification` to cancel work.

### Pollable SSE

For resumable responses, first emit a unique event ID with empty data. Before a
temporary close, send an SSE `retry` delay. The client honors it and reconnects
with GET plus `Last-Event-ID`, including when the stream began as a POST
response. Replay only the disconnected logical stream.

## Authorization quick reference

### Protect the MCP resource, not an upstream API

HTTP authorization uses OAuth 2.1; stdio obtains credentials from the
environment. Require PKCE for every client. Send bearer tokens on every HTTP
request, never in query strings. Use 401 for missing, invalid, or expired
authorization and 403 for insufficient scope.

An authorized MCP server publishes RFC 9728 protected-resource metadata with
at least one `authorization_servers` entry and points to it from a 401
`WWW-Authenticate` challenge. The client selects an advertised authorization
server and resolves its RFC 8414 metadata.

Include the RFC 8707 `resource` parameter in every authorization and token
request. Use the most specific canonical absolute MCP URI, include a
distinguishing path when needed, and omit fragments. Reject tokens issued for
another resource, and never pass an inbound MCP token through to an upstream
API.

### Select scopes and step up deliberately

Initially use the challenge's `scope`; otherwise request all
`scopes_supported`, or omit `scope` when metadata omits that field. A challenge
scope is authoritative for that request. For step-up, return 403 with bearer
`insufficient_scope`, the required `scope`, and `resource_metadata`; reauthorize
and retry the original operation with a small retry limit.

## Interaction quick reference

### Elicitation

Check the advertised elicitation modes. `{}` means legacy form-only behavior,
and an omitted request `mode` defaults to `"form"`. Form schemas are flat.
Primitive fields and string choices are supported; `2025-11-25` also supports
titled single choices and multi-select string arrays with defaults.

Use URL mode for sensitive or third-party interaction outside the client, not
for authorizing the client to the MCP server. `accept` means only that the user
agreed to open the URL. Wait for `notifications/elicitation/complete`, or handle
`-32042`, complete its required URL elicitations, and retry.

### Tool-enabled sampling

Require `sampling: {tools: {}}` before supplying `tools` or `toolChoice` in
`sampling/createMessage`. Execute assistant `tool_use` content and place the
matching `tool_result` in the immediately following user message. Every use has
exactly one result, and that message contains only tool results; otherwise use
`-32602`.

### Structured tool results

When a tool declares `outputSchema`, make `structuredContent` conform and have
clients validate it. Also serialize the same object into a text `content` item
for older clients. A result may include a `resource_link`; do not assume linked
resources also appear in `resources/list`.

## Experimental tasks quick reference

Tasks in `2025-11-25` require capability negotiation by request category.
Servers advertise `tasks.requests.tools.call`; clients advertise task support
for sampling or elicitation requests. Negotiate `tasks.list` and `tasks.cancel`
separately, and honor each tool's `execution.taskSupport` value.

An accepted task-augmented call returns `result.task` immediately. The receiver
assigns the ID, starts in `working`, and may override the requested TTL. Poll
`tasks/get` at `pollInterval`; status notifications do not replace polling.
Call `tasks/result` for the terminal underlying result or error, and also on
`input_required` so the receiver can deliver nested requests.

Keep terminal states immutable. Expired tasks may disappear, and cancelling a
terminal task returns `-32602`. Apply related-task metadata only as specified in
[interaction-patterns.md](references/interaction-patterns.md).

## Schema and presentation quick reference

Use JSON Schema 2020-12 by default. Account for standalone request-parameter
schemas rather than assuming parameters are embedded in RPC method definitions.
Allow `_meta` on the interface shapes that define it.

Use `name` as the programmatic identifier and optional `title` for display.
Tools, resources, resource templates, and prompts may carry icons. Initialization
`Implementation` data may include a human-readable `description`.

## Implementation checklist

- Pin behavior to the negotiated protocol revision.
- Send one JSON-RPC message per Streamable HTTP POST.
- Carry the negotiated version and any session ID on every required request.
- Validate `Origin`; return HTTP 403 when it is rejected.
- Bind bearer tokens to the canonical MCP resource and never forward them.
- Negotiate elicitation, completion, sampling, and task capabilities before use.
- Preserve structured tool results in both structured and compatibility text forms.
- Treat dropped SSE connections as resumable streams, not implicit cancellation.
- Validate schemas with the correct dialect and error layer.
