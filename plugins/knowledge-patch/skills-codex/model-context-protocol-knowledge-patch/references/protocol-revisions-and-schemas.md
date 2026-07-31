# Protocol Revisions and Schemas

## Apply lifecycle and framing changes

### Batching was added, then removed

Stable batch `2025-03-26` added JSON-RPC batching, allowing multiple requests
in one top-level array:

```json
[
  {"jsonrpc": "2.0", "id": 1, "method": "ping"},
  {"jsonrpc": "2.0", "id": 2, "method": "tools/list"}
]
```

Stable batch `2025-06-18` removed batching. For that revision and later, a
top-level array of requests or responses is invalid; send each JSON-RPC
message separately. This is a deliberate revision change, not a transport-only
exception.

### Lifecycle support is mandatory

The lifecycle operation requirement changed from **SHOULD** to **MUST** in
2025-06-18. Implementations targeting that revision must implement it.

## Negotiate completions

The `completions` capability lets a server explicitly advertise support for
argument autocompletion. Clients should check the capability before relying on
completion requests.

`CompletionRequest` also accepts `context` containing previously resolved
variables. Supply it when later suggestions depend on values already selected.

## Describe progress and content

`ProgressNotification` has a `message` field in addition to numeric progress
values. Use it for a human-readable status update:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "job-7",
    "progress": 42,
    "message": "Indexing files"
  }
}
```

MCP content may contain audio alongside text and image content.

## Describe tools and validate their results

Tool definitions may contain behavior annotations such as read-only or
destructive intent. Clients can use these annotations to present expected
effects more accurately.

A tool may declare `outputSchema` and return the corresponding JSON object in
`structuredContent`. Servers must conform to the schema and clients should
validate it. For compatibility with older clients, also serialize the same
object into a text item in `content`:

```json
{
  "content": [
    {"type": "text", "text": "{\"temperature\":22.5}"}
  ],
  "structuredContent": {"temperature": 22.5}
}
```

A tool result may also contain a `resource_link` with a fetchable or
subscribable URI and resource annotations:

```json
{
  "type": "resource_link",
  "uri": "file:///project/src/main.rs",
  "name": "main.rs",
  "mimeType": "text/x-rust"
}
```

The target of a `resource_link` is not guaranteed to appear in
`resources/list`.

Since stable batch `2025-11-25`, an input-validation failure from a tool call
should be a Tool Execution Error rather than a Protocol Error. This exposes a
correctable argument failure as the result of the tool operation.

## Use identifiers, titles, icons, and descriptions

Schema types may provide `title` as a human-friendly display label while
keeping `name` as the programmatic identifier. Continue using `name` in
protocol calls and show `title` in interfaces.

Servers may attach icon metadata to tools, resources, resource templates, and
prompts. Clients can display those icons with the corresponding objects.

The `Implementation` interface has an optional `description` for readable
client or server context during initialization.

## Update metadata and generated bindings

The schema adds `_meta` to additional interface types. Validators and
generated bindings must accept it on every interface shape defined to carry
metadata in the negotiated revision.

Request payload schemas are decoupled from RPC method definitions and exposed
as standalone parameter schemas. Schema consumers and generated bindings must
resolve and emit the standalone organization rather than assuming parameters
remain embedded in methods.

## Use the current JSON Schema dialect

JSON Schema 2020-12 is the default dialect for MCP schema definitions. Schema
producers, validators, and generators should use it unless another dialect is
selected explicitly.
