# Elicitation, Sampling, Tools, and Tasks

## Tool descriptions and results

### Behavior annotations (`2025-03-26`)

Tool definitions may include annotations describing intended behavior, such as
whether a tool is read-only or destructive. Clients use these hints when
presenting the expected effects.

### Structured and linked results (`2025-06-18-compat`)

A tool can declare `outputSchema` and return the matching JSON object in
`structuredContent`. Servers conform to the declared schema, and clients should
validate it. For compatibility with clients that only consume `content`, also
serialize the same JSON into a text item:

```json
{"content":[{"type":"text","text":"{\"temperature\":22.5}"}],"structuredContent":{"temperature":22.5}}
```

A result can also contain a fetchable or subscribable `resource_link`, together
with its resource annotations:

```json
{"type":"resource_link","uri":"file:///project/src/main.rs","name":"main.rs","mimeType":"text/x-rust"}
```

Do not assume a linked resource is present in `resources/list`.

### Input validation failures (`2025-11-25`)

Return a tool input-validation failure as a Tool Execution Error rather than a
Protocol Error. This keeps the failure at the tool-result layer so the caller
can inspect it and correct the arguments.

## Progress, audio, and completion (`2025-03-26`)

`ProgressNotification` includes a descriptive `message` in addition
to its numeric progress values:

```json
{"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"job-7","progress":42,"message":"Indexing files"}}
```

Content blocks may contain audio alongside text and images.

Servers advertise the `completions` capability before clients rely on argument
autocompletion. Since `2025-06-18`, `CompletionRequest.context` can carry
previously resolved variables so later suggestions account for earlier choices.

## Form elicitation (`2025-06-18-compat`)

A client advertises `capabilities.elicitation` before a server sends a nested
`elicitation/create` request. Use it for non-sensitive structured input. The
original `requestedSchema` is a flat object whose properties are primitive
strings, numbers or integers, booleans, or string enums. Do not use complex
nesting or arrays of objects.

```json
{"jsonrpc":"2.0","id":7,"method":"elicitation/create","params":{"message":"Choose a region","requestedSchema":{"type":"object","properties":{"region":{"type":"string","enum":["eu","us"],"enumNames":["Europe","United States"]}},"required":["region"]}}}
```

The response action is `accept`, `decline`, or `cancel`. Acceptance carries
schema-conforming `content`. Decline is an explicit refusal; cancel is a
dismissal without a choice, and both typically omit `content`.

```json
{"jsonrpc":"2.0","id":7,"result":{"action":"accept","content":{"region":"eu"}}}
```

## Elicitation modes and choices (`2025-11-25-compat`)

Clients explicitly advertise supported modes with
`elicitation: {form: {}, url: {}}`. The legacy empty object means form-only,
and an omitted request `mode` defaults to `"form"`.

Form schemas remain flat. For a titled single-select field, use `oneOf` entries
containing `const` and `title`. For titled multi-select, use a string array with
`items.anyOf`. Defaults are supported and clients should pre-populate them.

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

URL mode is still subject to change. It is for sensitive or third-party
interaction outside the client. Do not use it to authorize the client to its
MCP server. Supply `mode: "url"`, an `elicitationId`, `url`, and
explanatory `message`. An `accept` action only means the user agreed to open the
URL; it does not mean the external flow finished.

The server may later send `notifications/elicitation/complete` with the same
ID. Alternatively, error `-32042` can carry one or more required URL
elicitations; the client completes them and retries the original request.

## Tool-enabled sampling (`2025-11-25-compat`)

A client advertises `sampling: {tools: {}}` before a server includes `tools`
and optional `toolChoice` (`auto`, `required`, or `none`) in
`sampling/createMessage`.

A tool-using result contains assistant `tool_use` content with `id`, `name`, and
`input`. The server executes it and sends another sampling request whose next
user message contains the matching `tool_result`:

```json
{"role":"assistant","content":[{"type":"tool_use","id":"call-1","name":"weather","input":{"city":"Paris"}}]}
{"role":"user","content":[{"type":"tool_result","toolUseId":"call-1","content":[{"type":"text","text":"18 C"}]}]}
```

Every tool use is followed before any other message by exactly one matching
result. A tool-result message contains only tool results. Violations use
`-32602`.

`includeContext: "thisServer"` and `"allServers"` are soft-deprecated. Omit
the field for its `"none"` default. Send an old value only when the client
advertises `sampling: {context: {}}`.

## Experimental task negotiation and creation (`2025-11-25-compat`)

Negotiate tasks by request category. A server can advertise
`tasks.requests.tools.call`; a client can advertise
`tasks.requests.sampling.createMessage` and
`tasks.requests.elicitation.create`. Negotiate `tasks.list` and `tasks.cancel`
as separate capabilities.

A tool sets `execution.taskSupport` to `required`, `optional`, or `forbidden`;
absence means forbidden. A call that violates required or forbidden task use
returns `-32601`.

```json
{
  "method": "tools/call",
  "params": {
    "name": "long_job",
    "arguments": {},
    "task": {"ttl": 60000}
  }
}
```

An accepted augmented request immediately returns `result.task`, not the
underlying operation result. The receiver creates a unique task ID, starts it
in `working`, and may override the requested millisecond TTL.

## Task polling, results, and metadata (`2025-11-25-compat`)

Poll with `tasks/get` and respect `pollInterval`. Optional
`notifications/tasks/status` can reduce perceived latency but cannot replace
polling. `tasks/result` blocks until `completed`, `failed`, or `cancelled`, then
returns exactly the underlying request's result or JSON-RPC error.

When the state is `input_required`, call `tasks/result`; this lets the receiver
deliver the needed requests before the task returns to `working`.

Task-associated messages carry
`_meta["io.modelcontextprotocol/related-task"].taskId`. Omit it from
`tasks/get`, `tasks/list`, and `tasks/cancel` control messages. Also omit it
from a `tasks/result` request, where the `taskId` parameter is authoritative;
include it on the `tasks/result` response because the underlying result has no
task ID.

Terminal task states never transition. Expired tasks may disappear. Cancelling
an already-terminal task returns `-32602`.
