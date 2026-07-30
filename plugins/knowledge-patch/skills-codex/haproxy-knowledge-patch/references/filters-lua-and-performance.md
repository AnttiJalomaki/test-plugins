# Filters, Lua, and Performance

## Compression and filter execution

### Skip small compression candidates

HTTP request and response compression can skip bodies below a configured byte
threshold as of 3.2.0. For response compression, configure the minimum size as
follows:

```haproxy
filter compression
compression direction response
compression minsize-res 256
```

### Split request and response filters

Starting in 3.4.0, request and response compression use separate
`filter comp-req` and `filter comp-res` filters rather than the shared
compression filter and direction setting. The old `compression-direction`
directive is deprecated.

```haproxy
backend webservers
    filter comp-res
    compression algo gzip
    compression type text/html text/plain application/json
```

### Define filter order explicitly

`filter-sequence` in 3.4.0 controls execution order independently of filter
declaration order. A configured filter omitted from the sequence does not run.
Use this to reorder sensitive combinations such as compression and bandwidth
limiting, or to disable a declared filter without deleting its configuration.

## Lua pattern and map automation

### Mutable pattern references

Lua `core.get_patref` in 3.2.0 returns a mutable `patref` reference to an ACL or
Map file. A reference can:

- add or remove patterns;
- replace Map values;
- add entries in bulk;
- replace a complete file atomically with `prepare()` and `commit()`; and
- register event callbacks.

```lua
local ref = core.get_patref("virt@cached_paths.txt")
if ref ~= nil then
    ref:add(txn.f:path())
end
```

### Boolean sample conversion

Lua fetches continue to convert boolean samples to integers `0` and `1` by
default in 3.2.0. Opt in to real Lua booleans with:

```haproxy
global
    tune.lua.bool-sample-conversion normal
```

### Timed TCP applet receives

`AppletTCP:receive()` accepts an optional timeout in 3.2.0. An interactive TCP
service can therefore resume periodic work rather than block indefinitely for
input.

## CPU placement

Automatic binding in 3.2.0 considers packages, NUMA nodes, CCXs, L3 caches,
cores, and hardware threads. At that point HAProxy still confines itself to one
NUMA node by default, systems above 64 threads need extra configuration to use
every thread, and default limits are 1024 threads and 32 thread groups.

The defaults change in 3.3.0:

- `cpu-policy` defaults to `performance`, so heterogeneous CPUs use only
  performance cores by default;
- automatic placement uses all available cores and NUMA nodes; and
- the previous 64-thread automatic-placement limit is removed.

Validate these defaults on heterogeneous systems and any host where locality
or process isolation matters. Runtime `show dev` reports the resulting
thread-to-CPU bindings.

## Server connection-pool sharing

Global `tune.idle-pool.shared` in 3.4.0 controls cross-thread sharing of idle
server connections:

| Value | Behavior |
| --- | --- |
| `on` | Share within a thread group |
| `full` | Share across all threads |
| `off` | Disable sharing for debugging |

It supersedes and deprecates `tune.takeover-other-tg-connections`.

## Memory and fast-forward controls

Global `tune.notsent-lowat.client` and `tune.notsent-lowat.server` in 3.2.0
reduce kernel-side socket buffers and unacknowledged data to lower memory use.
Measure throughput and latency when lowering these limits.

`tune.disable-fast-forward` is stable in 3.3.0 and can be configured without
`expose-experimental-directives`.
