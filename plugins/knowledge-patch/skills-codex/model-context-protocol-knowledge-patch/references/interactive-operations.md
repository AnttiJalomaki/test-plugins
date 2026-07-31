# Interactive Operations

## Negotiate elicitation

Compatibility batch `2025-06-18-compat` adds elicitation. A client advertises
`capabilities.elicitation`; a server can then issue an `elicitation/create`
request inside another operation to ask for non-sensitive structured input.

The initial `requestedSchema` is a flat object whose properties are primitive
strings, numbers or integers, booleans, or string enums. Do not use complex
nesting or arrays of objects.

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "elicitation/create",
  "params": {
    "message": "Choose a region",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "region": {"type": "string", "enum": ["eu", "us"]}
      },
      "required": ["region"]
    }
  }
}
```

The response action is one of:

- `accept`, with schema-conforming `content`;
- `decline`, an explicit refusal, normally without `content`;
- `cancel`, dismissal without a choice, normally without `content`.

```json
{"jsonrpc":"2.0","id":7,"result":{"action":"accept","content":{"region":"eu"}}}
```

## Negotiate form and URL modes

Compatibility batch `2025-11-25-compat` adds explicit mode negotiation:

```json
{"elicitation": {"form": {}, "url": {}}}
```

The legacy empty elicitation capability means form-only. When
`elicitation/create` omits `mode`, it defaults to `"form"`.

### Form mode choices

Form schemas remain flat. They now support titled and multi-select choices:

- Use `oneOf` entries containing `const` and `title` for a titled
  single-select choice.
- Use a string array whose `items.anyOf` entries contain `const` and `title`
  for a titled multi-select choice.
- Use schema defaults and have clients pre-populate them.

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

### URL mode

URL mode is intended for sensitive or third-party interactions outside the
client. Do not use it to authorize the client to the MCP server. Its behavior
is still subject to change.

```json
{
  "method": "elicitation/create",
  "params": {
    "mode": "url",
    "elicitationId": "setup-7",
    "url": "https://mcp.example.com/connect",
    "message": "Connect the external service."
  }
}
```

An `accept` response means only that the user agreed to open the URL; it does
not mean the out-of-band work finished. The server can later send
`notifications/elicitation/complete` with the same `elicitationId`.
Alternatively, JSON-RPC error `-32042` can carry one or more required URL
elicitations for the client to complete before it retries the original
request.

## Use tools during sampling

A client must advertise `sampling: {tools: {}}` before a server includes
`tools` and optional `toolChoice` in `sampling/createMessage`. `toolChoice` is
one of `auto`, `required`, or `none`.

A tool-using sampling result contains assistant `tool_use` content with `id`,
`name`, and `input`. The server executes it and sends another sampling request
whose next user message contains the matching `tool_result`:

```json
{"role":"assistant","content":[{"type":"tool_use","id":"call-1","name":"weather","input":{"city":"Paris"}}]}
{"role":"user","content":[{"type":"tool_result","toolUseId":"call-1","content":[{"type":"text","text":"18 C"}]}]}
```

Every tool use must be followed, before any other message, by exactly one
matching result. A tool-result message must contain only tool results.
Violations produce invalid parameters error `-32602`.

## Avoid deprecated sampling context

`includeContext: "thisServer"` and `"allServers"` are soft-deprecated. Omit
the field to use its `"none"` default. A server should send either old value
only if the client explicitly advertises `sampling: {context: {}}`.

## Negotiate experimental tasks

Tasks are experimental in the 2025-11-25 revision. Negotiate them by request
category rather than as a single blanket capability:

- A server can advertise `tasks.requests.tools.call`.
- A client can advertise `tasks.requests.sampling.createMessage` and
  `tasks.requests.elicitation.create`.
- `tasks.list` and `tasks.cancel` have separate capabilities.

A tool declares `execution.taskSupport` as `required`, `optional`, or
`forbidden`; absence means `forbidden`. Violating required or forbidden task
use should return method not found (`-32601`).

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

An accepted task-augmented request immediately returns `result.task`, not the
underlying operation result. The receiver creates a unique task ID, starts the
task in `working`, and may override the requested TTL in milliseconds.

## Poll and complete tasks

Poll with `tasks/get`, respecting `pollInterval`. Optional
`notifications/tasks/status` can reduce latency but never replaces polling.
Call `tasks/result` to block until the task reaches `completed`, `failed`, or
`cancelled`; it then returns exactly the underlying operation's result or
JSON-RPC error.

```json
{"method":"tasks/get","params":{"taskId":"task-7"}}
{"method":"tasks/result","params":{"taskId":"task-7"}}
```

When the state is `input_required`, call `tasks/result` so the receiver can
deliver the required server-to-client requests. The receiver returns the task
to `working` after receiving input.

Task-associated messages carry this relation:

```json
{"io.modelcontextprotocol/related-task":{"taskId":"task-7"}}
```

Place it under `_meta`. Omit it from `tasks/get`, `tasks/list`, and
`tasks/cancel` control messages. Also omit it from a `tasks/result` request,
whose `taskId` parameter is authoritative. A `tasks/result` response must carry
it because the underlying result otherwise has no task ID.

Terminal task states never transition. Expired tasks may disappear. Cancelling
an already terminal task returns invalid parameters (`-32602`).
