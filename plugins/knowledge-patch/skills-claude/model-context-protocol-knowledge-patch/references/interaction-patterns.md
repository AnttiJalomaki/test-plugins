# Elicitation, sampling, tools, and multi-round work

Use this reference for structured tool results, user input, sampling tool loops,
legacy tasks, modern task extensions, and input-required retries.

Relevant protocol attributions: `2025-03-26`, `2025-06-18-compat`,
`2025-11-25-compat`, and `2026-07-28-rc`.

## Describe tool behavior and results

Tool definitions can carry behavior annotations such as read-only or
destructive intent. Clients can use those annotations to present expected
effects accurately, but must not substitute them for authorization or policy.

A tool may declare `outputSchema` and return the conforming JSON value in
`structuredContent`. Servers must conform to the schema and clients should
validate it. For compatibility with older clients, also serialize the same
value into a text item in `content`:

```json
{
  "content":[{"type":"text","text":"{\"temperature\":22.5}"}],
  "structuredContent":{"temperature":22.5}
}
```

A tool result can also contain a `resource_link` with a fetchable or
subscribable URI and resource annotations:

```json
{
  "type":"resource_link",
  "uri":"file:///project/src/main.rs",
  "name":"main.rs",
  "mimeType":"text/x-rust"
}
```

The linked resource is not guaranteed to appear in `resources/list`.

## Negotiate form elicitation

A legacy client advertises `capabilities.elicitation` before a server sends
`elicitation/create`. The 2025-06-18 form asks for non-sensitive structured
input with a flat object schema.

Allowed fields are primitive strings, numbers or integers, booleans, and string
enums. Complex nesting and arrays of objects are not supported.

```json
{
  "method":"elicitation/create",
  "params":{
    "message":"Choose a region",
    "requestedSchema":{
      "type":"object",
      "properties":{"region":{"type":"string","enum":["eu","us"]}},
      "required":["region"]
    }
  }
}
```

The response action is:

- `accept`, with schema-conforming `content`;
- `decline`, an explicit refusal that normally omits content;
- `cancel`, a dismissal without a choice that normally omits content.

The 2025-11-25 capability explicitly negotiates
`elicitation: {form: {}, url: {}}`. A legacy empty object means form only, and
an omitted request `mode` defaults to `"form"`.

For titled single-select fields, use `oneOf` entries containing `const` and
`title`. For titled multi-select, use a string array whose `items.anyOf`
contains those entries. Defaults are supported and clients should pre-populate
them:

```json
{
  "type":"array",
  "items":{
    "anyOf":[
      {"const":"eu","title":"Europe"},
      {"const":"us","title":"United States"}
    ]
  },
  "default":["eu"]
}
```

## Use legacy URL elicitation only for out-of-band work

The 2025-11-25 URL mode is for sensitive or third-party interaction outside the
client. It is not a way to authorize the client to the MCP server.

An `accept` result means only that the user agreed to open the URL. It does not
mean the external flow completed. The legacy server can later send
`notifications/elicitation/complete` with the same `elicitationId`, or return
error `-32042` containing required URL elicitations that the client completes
before retrying the original request.

In the modern multi-round form, URL elicitation has neither `elicitationId` nor
`notifications/elicitation/complete`. Retry the original request to learn
whether work completed. Put a server-defined correlation value in
`requestState` when it must survive retries.

## Run legacy tool-enabled sampling loops

A client advertises `sampling: {tools: {}}` before a server includes `tools`
and optional `toolChoice` (`"auto"`, `"required"`, or `"none"`) in
`sampling/createMessage`.

A tool-using assistant result contains `tool_use` with `id`, `name`, and
`input`. The server executes it, then sends another sampling request whose next
user message contains the matching `tool_result`:

```json
{"role":"assistant","content":[{"type":"tool_use","id":"call-1","name":"weather","input":{"city":"Paris"}}]}
{"role":"user","content":[{"type":"tool_result","toolUseId":"call-1","content":[{"type":"text","text":"18 C"}]}]}
```

Every tool use must be followed, before any other message, by exactly one
matching result. A tool-result message contains only tool results. Violations
use `-32602`.

Sampling context is soft-deprecated. Omit `includeContext` to select its
`"none"` default. Send `"thisServer"` or `"allServers"` only when the client
advertises `sampling: {context: {}}`.

## Interoperate with experimental 2025 tasks

Tasks in 2025-11-25 are experimental and negotiated by request category:

- a server can advertise `tasks.requests.tools.call`;
- a client can advertise `tasks.requests.sampling.createMessage`;
- a client can advertise `tasks.requests.elicitation.create`;
- `tasks.list` and `tasks.cancel` have separate capabilities.

A tool sets `execution.taskSupport` to `"required"`, `"optional"`, or
`"forbidden"`; absent means forbidden. Violating required or forbidden task use
should return `-32601`.

An augmented call supplies a requested millisecond TTL:

```json
{
  "method":"tools/call",
  "params":{"name":"long_job","arguments":{},"task":{"ttl":60000}}
}
```

On acceptance, immediately return `result.task`, not the underlying operation
result. The receiver creates the unique task ID, starts in `working`, and may
override the requested TTL.

Poll with `tasks/get` and respect `pollInterval`; optional
`notifications/tasks/status` never replaces polling. `tasks/result` blocks
until `completed`, `failed`, or `cancelled`, then returns exactly the underlying
result or JSON-RPC error. In `input_required`, call `tasks/result` so the
receiver can deliver required requests before returning to `working`.

Task-associated messages carry
`_meta["io.modelcontextprotocol/related-task"].taskId`, except:

- `tasks/get`, `tasks/list`, and `tasks/cancel` control messages;
- a `tasks/result` request, whose parameter is authoritative.

A `tasks/result` response must carry the metadata because the underlying result
has no task ID. Terminal states never transition, expired tasks may disappear,
and cancelling a terminal task returns `-32602`.

## Implement the modern task extension separately

Modern tasks move from core capabilities into the official
`io.modelcontextprotocol/tasks` extension. Do not translate the 2025 shape
field-for-field.

The redesign:

- removes `tasks/result` and `tasks/list`;
- uses `tasks/get` for polling;
- uses `tasks/update` for client-to-server input;
- lets a server return task handles without per-request opt-in.

An SDK can remove its experimental task managers and still permit explicit
custom-method interop with a 2025 peer.

## Implement modern multi-round requests

A modern server does not directly send `roots/list`, `sampling/createMessage`,
or `elicitation/create`. It returns an `InputRequiredResult` with
`resultType: "input_required"` and a map of named `inputRequests`.

The client fulfills those requests, then retries the original operation with
that round's `inputResponses`. A normal result carries
`resultType: "complete"`.

Each retry includes only its current input responses. Preserve older answers
and the flow phase in `requestState`. Verify that state, bind it to principal,
operation, and expiry, and do not put secrets in a merely signed state value.

Automatic clients may fulfill embedded elicitation, sampling, and roots
requests through registered callbacks. Manual clients need an explicit
input-required driver and a bounded round limit to avoid infinite retry loops.
