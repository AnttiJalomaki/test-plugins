# Lifecycle, Messages, Metadata, and Schemas

## JSON-RPC batching history

The `2025-03-26` revision introduced batching, allowing several protocol
requests in one top-level array:

```json
[
  {"jsonrpc": "2.0", "id": 1, "method": "ping"},
  {"jsonrpc": "2.0", "id": 2, "method": "tools/list"}
]
```

The `2025-06-18` revision removed batching. For that revision and later, a
top-level request or response array is invalid; send every JSON-RPC message
separately. This also matches Streamable HTTP's one-message-per-POST framing.

## Lifecycle requirement (`2025-06-18`)

The lifecycle operation changed from **SHOULD** to **MUST**. Treat it as a
required part of implementations targeting this revision rather than an
optional interoperability enhancement.

## Extensible metadata (`2025-06-18`)

Additional interface types define `_meta`. Validators, schema consumers, and
generated bindings must allow it on each newly covered interface shape and
preserve its defined semantics.

## Programmatic names and display titles (`2025-06-18`)

Schema types may supply `title` as a human-friendly display name while keeping
`name` as the programmatic identifier. Protocol operations continue to address
objects by `name`; user interfaces may prefer `title`.

## Icons and implementation descriptions (`2025-11-25`)

Tools, resources, resource templates, and prompts may expose icons as metadata.
Clients can display those icons when presenting the objects.

The initialization `Implementation` interface has an optional `description`
field for human-readable client or server context.

## Tool validation error layer (`2025-11-25`)

Represent a tool input-validation failure as a Tool Execution Error, not a
Protocol Error. The distinction lets the caller inspect the tool failure and
retry with corrected input.

## JSON Schema dialect (`2025-11-25`)

JSON Schema 2020-12 is the default dialect for MCP schema definitions. Schema
producers, validators, and code generators use it unless a different dialect is
explicitly selected.

## Standalone request-parameter schemas (`2025-11-25`)

Request payload schemas are decoupled from RPC method definitions and exposed
as standalone parameter schemas. Schema consumers and generated bindings must
resolve the new organization rather than assuming that each method definition
contains its parameter shape.

## Invalid Origin response (`2025-11-25`)

A Streamable HTTP server that rejects an invalid `Origin` returns HTTP 403
Forbidden. Validate the header consistently on incoming connections to prevent
DNS-rebinding attacks.
