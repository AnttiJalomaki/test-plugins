# Streaming and interrupts

## Wire-format streaming

The low-level `toLangGraphEventStream` helper has been removed. Low-level
clients should request the wire format through `graph.stream`'s `encoding`
option and return that stream directly:

```typescript
const stream = await graph.stream(input, {
  encoding: "text/event-stream",
  streamMode: ["values", "messages"],
});

return new Response(stream, {
  headers: { "Content-Type": "text/event-stream" },
});
```

## UI transport injection

The React `useStream` hook accepts a custom `transport`, so a UI can retain its
stream handling while changing the network layer:

```typescript
const stream = useStream({
  transport: new FetchStreamTransport({
    apiUrl: "http://localhost:2024",
  }),
});
```

## Snapshot privacy

State schemas do not redact `values` streams. Input and output schemas, and
channels intended as private, can all appear in full state snapshots. Filter
v3 event snapshots with `output_keys` in Python or `outputKeys` in JavaScript
when emitted snapshots must expose only selected channels.

## Typed JavaScript interrupts

The `StateGraph` constructor accepts an `interrupts` map of named definitions.
`interrupt<Input, Resume>()` types the payload sent through
`runtime.interrupt.<name>()` and the value returned on resume.
`graph.isInterrupted(result)` recognizes an interrupted result.

```typescript
import { StateGraph, interrupt } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({ messages: z.array(z.string()) });

const graph = new StateGraph(State, {
  interrupts: {
    approve: interrupt<{ reason: string }, { messages: string[] }>(),
  },
})
  .addNode("review", (_state, runtime) => {
    const response = runtime.interrupt.approve({ reason: "review" });
    return { messages: response.messages };
  })
  .compile();
```

## Python v3 interrupt streams

`graph.stream_events(..., version="v3")` provides typed projections for
message chunks, state snapshots, pending interrupts, interruption status, and
final output.

Drive the stream to completion before reading `stream.output`,
`stream.interrupted`, or `stream.interrupts`. If interrupted, construct a new
v3 stream with `Command(resume=...)` and repeat until it finishes:

```python
stream = graph.stream_events(inputs, config=config, version="v3")
result = stream.output

if stream.interrupted:
    stream = graph.stream_events(
        Command(resume=review(stream.interrupts)),
        config=config,
        version="v3",
    )
```

Token chunks are available in `stream.messages`, full per-step snapshots in
`stream.values`, and nested-subgraph token chunks in
`stream.subgraphs[*].messages`.

## Resume parallel interrupts by ID

Parallel branches may pause on multiple interrupts. Pair every pending
interrupt's `id` with its answer, then pass the whole mapping as the resume
value so each branch receives the intended response:

```typescript
import { Command, INTERRUPT, isInterrupted } from "@langchain/langgraph";

const paused = await graph.invoke(input, config);
const responses: Record<string, string> = {};

if (isInterrupted(paused)) {
  for (const item of paused[INTERRUPT]) {
    if (item.id != null) responses[item.id] = answer(item.value);
  }
}

await graph.invoke(new Command({ resume: responses }), config);
```

## Validate with one interrupt call

Do not re-prompt using a `while` loop that contains `interrupt()`. Each resume
restarts the node and replays earlier iterations, causing repeated work to
grow exponentially.

Store the next prompt in state, call `interrupt()` exactly once per node
invocation, and use a conditional edge to revisit the node after invalid
input:

```typescript
builder
  .addNode("collectAge", (state) => {
    const answer = interrupt(state.pendingQuestion ?? "What is your age?");
    return typeof answer === "number" && answer > 0
      ? { age: answer, pendingQuestion: null }
      : { pendingQuestion: `'${answer}' is not a valid age.` };
  })
  .addConditionalEdges(
    "collectAge",
    (state) => state.age !== null ? END : "collectAge",
  );
```
