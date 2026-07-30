# Runtime, data structures, regular expressions, and I/O

## Process signaling and memory

### Priority messages

Since 28.0, a receiver can create a priority-capable alias:

```erlang
PrioAlias = alias([priority]),
erlang:send(PrioAlias, Message, [priority]).
```

A send that uses both the alias and the `priority` option places the message
ahead of ordinary messages while preserving signal order. Sending through
the alias without the option remains ordinary, and `unalias/1` revokes the
capability.

`exit(PrioAlias, Reason, [priority])` sends a priority exit signal. For
event-generated link and monitor signals, pass `priority` to
`erlang:link/2` or `erlang:monitor/3`.

### Stack-preserving hibernation

`erlang:hibernate/0`, added in 28.0, minimizes the calling process's memory
while it waits for the next message. Unlike `erlang:hibernate/3`, it does not
discard the call stack.

## Shell and terminal behavior

### Lazy standard input

Since 28.0, standard input is read only when `io:get_line/2` or an equivalent
operation requests it. The former `-noinput` workaround for greedy reads is
no longer necessary.

### Raw `noshell`

`noshell` stays cooked by default. Start a raw session to disable line
editing and output echo and to read keystrokes without waiting for Enter:

```erlang
shell:start_interactive({noshell, raw}).
```

### ANSI output

The `io_ansi` module added in 29.0 formats or writes terminal colors and
styles using the local terminfo database. A remote call uses the destination
terminal's capabilities.

```erlang
io_ansi:fwrite([bold, red, "wrong answer: ", "~p~n"], [99]).
```

Use `io_ansi:format/2` to return the encoded output as a binary.

## Regular expressions

### PCRE2 migration

The `re` module moved to PCRE2 in 28.0. Retest patterns and consumers because:

- validation is stricter, so invalid escapes including `\M`, `\i`, `\B`,
  and `\8` can raise `badarg`;
- Unicode property matches can change with the newer property data;
- branch-reset groups can change `re:split/3` output.

The internal value from `re:compile/2` is not portable between nodes or OTP
versions. Since 28.1, use the supported `re` export/import operations when a
compiled expression must be transferred between Erlang node instances.

## Persistent terms

`persistent_term:put_new/2`, added in 28.4, returns quickly when the same key
and value are already stored. If the key exists with a different value, it
raises `badarg`.

```erlang
persistent_term:put_new(config, Config).
```

## Arrays

The `array` module in 29.0 adds:

- `prepend/2`, `append/2`, and `concat/1,2`;
- `slice/3`, `shift/2`, and `from/2,3`;
- index-bounded traversal forms such as `foldl/5`;
- `mapfold` families such as `mapfoldl/3` and `sparse_mapfoldr/5`.

Its internal representation also changed. Array terms serialized with
`term_to_binary/1` on earlier OTP releases are not compatible with OTP 29.
Migrate persisted or exchanged values through a neutral representation.

## Maps

Map order is still undefined. Since 29.0, however, all standard iteration
forms produce the elements of a particular map in the same order:
`maps:keys/1`, `maps:values/1`, `maps:to_list/1`, default iterators, and map
comprehensions agree. This does not promise a stable or sorted order.

Common Test output has used the order from `maps:iterator(Map, ordered)` since
28.1. Update snapshots or output parsers that assumed another rendered key
order.

## Ordered sets and trees

Since 29.0, `gb_sets:from_ordset/1` and
`gb_trees:from_orddict/1` reject unordered input instead of silently
building invalid structures. For example,
`gb_sets:from_ordset([3,2,1])` raises `badarg` with reason `not_ordset`.

## Functional graphs

The `graph` module introduced in 29.0 is a persistent, functional counterpart
to `digraph` and `digraph_utils`. Each modifying operation returns a new
graph, and previous graph values remain usable:

```erlang
G0 = graph:new(),
G1 = graph:add_vertex(G0, a),
G2 = graph:add_vertex(G1, b),
G3 = graph:add_edge(G2, a, b).
```

