---
name: model-context-protocol-knowledge-patch
description: Model Context Protocol (MCP)
version: 2025-11-25
license: MIT
metadata:
  author: Nevaberry
---



# Model Context Protocol Knowledge Patch

Use this skill when implementing, reviewing, or debugging MCP clients, servers,
transports, authorization, schemas, elicitation, sampling, or experimental
tasks. Start with the breaking-change checks, negotiate every optional feature,
and then open the topic reference that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [authorization.md](references/authorization.md) | OAuth profile, discovery, registration, resource binding, scopes, and step-up authorization |
| [transport-sessions-and-subscriptions.md](references/transport-sessions-and-subscriptions.md) | Streamable HTTP, sessions, SSE resumption, cancellation, security, and legacy HTTP+SSE fallback |
| [interactive-operations.md](references/interactive-operations.md) | Elicitation, sampling with tools, experimental tasks, and task polling |
| [protocol-revisions-and-schemas.md](references/protocol-revisions-and-schemas.md) | Lifecycle, batching, capabilities, tool results, content, metadata, and JSON Schema changes |

## Breaking changes first

### Send one JSON-RPC message at a time

Do not send a top-level JSON-RPC batch when targeting 2025-06-18 or later.
Batching existed in 2025-03-26, but the next revision removed it. Streamable
HTTP also requires each POST body to contain exactly one request, notification,
or response.

### Treat lifecycle operation support as mandatory

The lifecycle operation requirement is a **MUST** from 2025-06-18. Do not treat
it as a best-effort feature when validating an implementation.

### Send the negotiated protocol version

After initialization, include `MCP-Protocol-Version` on every Streamable HTTP
request. A missing version is interpreted as `2025-03-26` when no other version
information exists; an invalid or unsupported value produces HTTP 400.

```http
MCP-Protocol-Version: 2025-11-25
```

### Use the current schema organization

JSON Schema 2020-12 is the default dialect. Request parameter schemas are
standalone rather than embedded in RPC method definitions, and additional
interface shapes accept `_meta`. Update validators and generated bindings
accordingly.

### Classify bad tool input as an execution error

Return input-validation failures as Tool Execution Errors, not Protocol
Errors. This keeps an ordinary bad argument visible to the caller so it can be
corrected.

## Deprecations and compatibility traps

### Prefer Client ID Metadata Documents

For a client and authorization server with no prior relationship, use a Client
ID Metadata Document when the server advertises support. Dynamic Client
Registration and manually entered credentials are compatibility fallbacks.

### Omit legacy sampling context by default

`includeContext: "thisServer"` and `"allServers"` are soft-deprecated. Omit
`includeContext` for the `"none"` default. Send an old value only when the
client advertises `sampling: {context: {}}`.

### Restrict legacy HTTP+SSE detection

When probing an unknown URL, fall back from the initial Streamable HTTP POST to
legacy GET only after HTTP 400, 404, or 405. Other 4xx responses do not select
the old transport. On a successful legacy probe, require the initial
`endpoint` SSE event.

## Authorization quick reference

### Select the profile by transport

- Authorization is optional at the protocol level.
- An HTTP transport that implements authorization should use OAuth 2.1.
- A stdio implementation should obtain credentials from its environment.
- Require PKCE for every client.
- Use authorization-code grants for users or client-credentials grants for
  applications as appropriate.

### Discover the protected resource first

An authorized server publishes RFC 9728 protected-resource metadata with at
least one `authorization_servers` entry. Point the client to it from a 401
`WWW-Authenticate` challenge using `resource_metadata`. If that parameter is
absent, try protected-resource discovery at the MCP-path form, then at the
origin root.

After choosing an advertised authorization server, discover its RFC 8414 or
OIDC metadata. Preserve the defined ordering for issuers that contain paths;
do not construct a single guessed well-known URL.

### Bind every token request to the resource

Include RFC 8707 `resource` in every authorization and token request, even if
the authorization server does not advertise support. Use the most specific
canonical absolute MCP URI, retain a distinguishing path, and omit fragments.
The MCP server must reject tokens issued for another resource.

Send `Authorization: Bearer <access-token>` on every HTTP request. Never place
the token in the query string and never pass an inbound MCP token through to
an upstream API.

### Handle status codes deliberately

- Missing, invalid, or expired authorization returns HTTP 401.
- Insufficient scope returns HTTP 403 with an `insufficient_scope` Bearer
  challenge, the required `scope`, and `resource_metadata`.
- A user-facing client should reauthorize for the challenged scopes and retry
  the original operation with a small retry limit.

## Streamable HTTP quick reference

### POST requests and responses

POST every client message to the single MCP endpoint with both accepted media
types:

```http
Accept: application/json, text/event-stream
Content-Type: application/json
```

For a request, accept either one JSON response or an SSE response stream. For
accepted notification-only or response-only input, expect an empty HTTP 202.

### Optional GET stream

A client may open a separate GET with `Accept: text/event-stream` for
server-initiated traffic. A server without this stream returns 405. Do not put
ordinary JSON-RPC responses on it except while replaying a previous request's
stream.

### Sessions and cleanup

If initialization returns `Mcp-Session-Id`, repeat it on every later HTTP
request. A missing required ID yields 400. An expired or terminated ID yields
404; initialize again without an ID. Request cleanup with DELETE and tolerate
405 when the server does not support deletion.

### Resume without implicitly cancelling

SSE event IDs must be unique within their session, or within their client when
there is no session. Resume with GET plus `Last-Event-ID`. Replay only the
disconnected logical stream. A dropped stream does not cancel its request;
send an explicit cancellation notification.

For pollable SSE, honor the server's `retry` delay. A server may send an event
ID with empty data and close the HTTP connection while the logical stream
remains active.

### Enforce transport security

Validate `Origin` on every incoming connection and return HTTP 403 when it is
invalid. Bind local servers to `127.0.0.1`, not `0.0.0.0`; authenticate
connections; serve authorization endpoints over HTTPS; and accept only
localhost or HTTPS redirect URIs.

## Interactive operations quick reference

### Negotiate elicitation modes

The legacy empty elicitation capability means form-only. Current clients can
advertise `elicitation: {form: {}, url: {}}`; an omitted request mode defaults
to `"form"`.

Form elicitation is for non-sensitive structured input. Keep its schema flat.
Use primitive values and string choices; titled single-select choices use
`oneOf` with `const` and `title`, while titled multi-select choices use a string
array with `items.anyOf`. Honor defaults and pre-populate them.

Use URL mode for sensitive or third-party interaction outside the client, not
to authorize the client to the MCP server. Its behavior is still subject to
change. `accept` means only that the user agreed to open the URL. Completion
arrives later through `notifications/elicitation/complete`, or error `-32042`
can carry required URL elicitations before a retry.

### Validate structured tool results

When a tool declares `outputSchema`, return a matching object in
`structuredContent` and validate it. Also serialize the same value into a text
content item for older clients. A tool result may include a `resource_link`;
do not assume that linked URI also appears in `resources/list`.

### Negotiate sampling tools

Require `sampling: {tools: {}}` before sending `tools` or `toolChoice` in
`sampling/createMessage`. `toolChoice` is `auto`, `required`, or `none`.
Follow every assistant `tool_use` immediately with exactly one matching
`tool_result`; that user message must contain only tool results. Violations are
invalid parameters (`-32602`).

### Gate experimental tasks by operation

Negotiate task support separately for tool calls, sampling, elicitation, list,
and cancel. Respect each tool's `execution.taskSupport`. An accepted task-
augmented request returns `result.task` immediately; poll `tasks/get` at the
advertised interval and use `tasks/result` for the eventual underlying result.
Do not assume optional status notifications replace polling.

## Presentation and capability checks

- Check the `completions` capability before relying on completion requests.
  Pass already resolved variables through `CompletionRequest.context`.
- Use `name` as the protocol identifier and optional `title` as the display
  label.
- Display optional `icons` on tools, resources, resource templates, and
  prompts when supported.
- Use `Implementation.description` as human-readable initialization context.
- Accept audio content in addition to text and images.
- Present descriptive progress from `ProgressNotification.message`.
- Use tool behavior annotations as intent metadata, especially read-only and
  destructive hints.

## Final implementation checklist

- Negotiate the protocol revision and all optional capabilities.
- Reject top-level batches for current revisions.
- Validate JSON Schema 2020-12 inputs and declared structured outputs.
- Include the version, session ID when present, and Bearer token when required
  on every applicable HTTP request.
- Keep authorization resource and token audiences exact.
- Validate `Origin` before processing Streamable HTTP traffic.
- Preserve SSE event IDs, retry delays, and explicit cancellation semantics.
- Separate form elicitation, URL elicitation, sampling, and task capability
  checks.
- Return tool input mistakes as execution errors.
- Open the linked reference before implementing discovery orders, task state
  transitions, or compatibility fallback behavior.
