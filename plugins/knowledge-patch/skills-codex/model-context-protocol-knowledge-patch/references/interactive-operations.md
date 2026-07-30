# Interactive operations

## Tool results and presentation metadata

### Tool behavior annotations

Since `2025-03-26`, tool definitions can include behavior annotations such as
read-only or destructive intent. Clients can use them to present expected
effects, but should not mistake descriptive annotations for an authorization
boundary.

Tools may declare an `outputSchema` and return matching JSON in
`structuredContent` (since `2025-06-18-compat`). Servers must conform to the
declared schema and clients should validate it. To support older clients,
serialize the same value into a text item in `content`.

```json
{
  "content": [{"type": "text", "text": "{\"temperature\":22.5}"}],
  "structuredContent": {"temperature": 22.5}
}
```

A tool result may contain `resource_link` with a fetchable or subscribable URI
and resource annotations. The linked URI is not guaranteed to appear in
`resources/list`.

```json
{
  "type": "resource_link",
  "uri": "file:///project/src/main.rs",
  "name": "main.rs",
  "mimeType": "text/x-rust"
}
```

Since `2025-11-25`, tools, resources, resource templates, and prompts can expose
icons. `Implementation` also has an optional human-readable `description`.
Schema types may use `title` for display while keeping `name` as the
programmatic identifier.

Tool input validation failures should be Tool Execution Errors, not Protocol
Errors. This makes the failure available to the caller so it can correct the
arguments.

### Progress, audio, and completion

`ProgressNotification.message` adds a descriptive status to numeric progress.

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

Content can include audio alongside text and images. The `completions`
capability lets a server advertise argument-autocompletion support; clients
should check it before calling completion operations. A
`CompletionRequest.context` can carry previously resolved variables so later
suggestions account for earlier choices.

## Form elicitation

In the 2025 protocol, a client advertises elicitation before a server sends an
`elicitation/create` request inside another operation. In
`2025-06-18-compat`, `requestedSchema` is a flat object whose properties are
primitive strings, numbers/integers, booleans, or string enums. Do not put
sensitive information in form elicitation.

```json
{
  "method": "elicitation/create",
  "params": {
    "message": "Choose a region",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "region": {
          "type": "string",
          "enum": ["eu", "us"],
          "enumNames": ["Europe", "United States"]
        }
      },
      "required": ["region"]
    }
  }
}
```

The response action is:

- `accept`, with schema-conforming `content`.
- `decline`, an explicit refusal that usually omits content.
- `cancel`, dismissal without a choice, also usually without content.

```json
{"result":{"action":"accept","content":{"region":"eu"}}}
```

Mode negotiation in `2025-11-25-compat` uses
`elicitation: {form: {}, url: {}}`. The legacy empty capability object means
form-only, and an omitted request `mode` defaults to `"form"`.

Form schemas remain flat, but gained richer choice fields:

- Use `oneOf` entries with `const` and `title` for titled single-select fields.
- Use an array whose `items.anyOf` contains titled choices for multi-select.
- Use defaults where appropriate; clients should pre-populate them.

```json
{
  "type": "array",
  "items": {
    "anyOf": [
      {"const": "eu", "title": "Europe"},
      {"const": "us", "title": "United States"}
    ]
  }
}
```

## URL elicitation

The 2025 URL mode is for a sensitive or third-party interaction outside the
client. It must not be used to authorize the client itself to the MCP server.

In `2025-11-25-compat`, a URL request had `elicitationId`, `url`, and `message`.
An `accept` response meant only that the user agreed to open the URL, not that
the external interaction finished. Completion could arrive through
`notifications/elicitation/complete`; error `-32042` could instead carry one or
more required URL elicitations for completion before retry.

The modern RC removes `elicitationId` and the completion notification. The
client discovers completion by retrying the original operation. A server that
needs correlation puts its own identifier in `requestState`.

## Tool-enabled sampling

In `2025-11-25-compat`, clients advertise `sampling: {tools: {}}` before a
server sends `tools` and optional `toolChoice` (`auto`, `required`, or `none`)
in `sampling/createMessage`.

A tool-using sampling result contains assistant `tool_use` content. The server
executes it and sends another sampling request whose next user message has the
matching `tool_result`.

```json
{"role":"assistant","content":[{"type":"tool_use","id":"call-1","name":"weather","input":{"city":"Paris"}}]}
{"role":"user","content":[{"type":"tool_result","toolUseId":"call-1","content":[{"type":"text","text":"18 C"}]}]}
```

Every tool use must be followed, before any other message, by exactly one
matching result. A tool-result message contains only tool results. Violations
use `-32602`.

`includeContext: "thisServer"` and `"allServers"` are soft-deprecated. Omit the
field for its `"none"` default. A server should send either old value only
when the client explicitly advertises `sampling: {context: {}}`.

## Experimental 2025 tasks

Tasks in `2025-11-25-compat` were experimental and negotiated by request
category:

- Servers could advertise `tasks.requests.tools.call`.
- Clients could advertise `tasks.requests.sampling.createMessage` and
  `tasks.requests.elicitation.create`.
- `tasks.list` and `tasks.cancel` had separate capabilities.
- Tools used `execution.taskSupport` with `required`, `optional`, or
  `forbidden`; absence meant forbidden. Violations used `-32601`.

An augmented request included `task: {ttl: ...}`. Acceptance returned
`result.task` immediately rather than the underlying operation result. The
receiver created a unique ID, began in `working`, and could override the
requested millisecond TTL.

Poll with `tasks/get` while honoring `pollInterval`.
`notifications/tasks/status` is optional and does not replace polling.
`tasks/result` blocks until `completed`, `failed`, or `cancelled`, then returns
exactly the underlying result or JSON-RPC error. In state `input_required`, the
requestor calls `tasks/result` so the receiver can deliver required requests
before returning to `working`.

Task-associated messages carry:

```text
_meta["io.modelcontextprotocol/related-task"].taskId
```

Control requests `tasks/get`, `tasks/list`, and `tasks/cancel` omit the related
task marker, as does a `tasks/result` request because its parameter is
authoritative. A `tasks/result` response carries it because its underlying
result has no task ID. Terminal states do not transition; expired tasks may
disappear; cancelling a terminal task returns `-32602`.

## Modern tasks extension

In the `2026-07-28-rc`, tasks moved from core to the official
`io.modelcontextprotocol/tasks` extension advertised in client and server
`extensions` capabilities.

The redesign:

- Removes `tasks/result` and `tasks/list`.
- Uses `tasks/get` for polling.
- Uses `tasks/update` for client-to-server input.
- Allows a server to return a task handle without per-request opt-in.

SDKs may retain deprecated task wire types for 2025 interoperability without
implementing the modern extension. Do not assume a v2 SDK supports Tasks merely
because the protocol defines the extension.

## Modern multi-round trips

Modern servers return an `InputRequiredResult` instead of directly issuing
roots, sampling, or elicitation requests:

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "choice": {
      "method": "elicitation/create",
      "params": {"message": "Choose a region"}
    }
  }
}
```

The client fulfills those requests and retries the original operation with
`inputResponses`. Every result carries `resultType`; ordinary results use
`"complete"`. When reading an older peer, a missing value is interpreted as
complete.

Each retry carries only that round's responses. Carry earlier answers, phase,
principal binding, operation binding, and expiry in signed `requestState`.
Signing provides integrity, not confidentiality; do not put secrets in an
unencrypted state token.

Modern SDKs can automatically drive several rounds, but applications should
retain a bounded round count. Stateless legacy HTTP cannot fully emulate the
back-channel and should produce a capability refusal rather than hanging.
