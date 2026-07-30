# Streaming and Interrupts

Source batches: `langgraph-v1`, `graph-api-overview`,
`human-in-the-loop`.

## Event-stream wire encoding

The low-level JavaScript `toLangGraphEventStream` helper has been removed.
Request the wire representation with the `encoding` option of `graph.stream`
and return that stream directly.

```typescript
const stream = await graph.stream(input, {
  encoding: "text/event-stream",
  streamMode: ["values", "messages"],
});

return new Response(stream, {
  headers: { "Content-Type": "text/event-stream" },
});
```

## Pluggable React transports

React `useStream` accepts a custom `transport`, allowing the network
implementation to change without rewriting UI stream handling.

```typescript
const stream = useStream({
  transport: new FetchStreamTransport({
    apiUrl: "http://localhost:2024",
  }),
});
```

## State visibility in value streams

Input, output, and private state schemas do not redact `values` snapshots. A
node's input schema only limits what it reads, and node schemas may add private
channels to the graph-state union.

Use v3 event streams with `output_keys` in Python or `outputKeys` in
JavaScript when private state must stay out of emitted snapshots.

```python
stream = graph.stream_events(
    {"user_input": "My"},
    version="v3",
    output_keys=["graph_output"],
)
```

## Named, typed JavaScript interrupts

The JavaScript `StateGraph` constructor accepts an `interrupts` map.
`interrupt<Input, Resume>()` types the payload passed to
`runtime.interrupt.<name>()` and the value returned after execution resumes.
Use `graph.isInterrupted(result)` to recognize an interrupted result.

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
final output. Drive the stream to completion before reading `output`,
`interrupted`, or `interrupts`.

If interrupted, start another v3 stream with `Command(resume=...)` and repeat
until the stream completes without an interruption.

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

Token chunks are exposed through `stream.messages`, full step snapshots
through `stream.values`, and nested-subgraph token chunks through
`stream.subgraphs[*].messages`.

## Resuming parallel interrupts

Parallel branches may pause on multiple interrupts simultaneously. Build a
complete mapping from each pending interrupt's `id` to its answer and pass that
mapping as the resume value. This ensures that every branch receives the
intended response.

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

## Validation without replay amplification

Do not put `interrupt()` inside a validation `while` loop. Every resume
restarts the node and replays earlier iterations, so repeated invalid answers
cause the loop's work to grow exponentially.

Keep the next prompt in graph state, call `interrupt()` exactly once in each
node invocation, and use a conditional edge to revisit the node after invalid
input.

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
