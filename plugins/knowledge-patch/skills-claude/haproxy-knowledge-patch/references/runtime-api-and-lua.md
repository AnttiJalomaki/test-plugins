# Runtime API and Lua

## Persistent Master CLI worker sessions

When selecting a worker by relative PID, use `@@` instead of `@` to keep the
Master CLI session interactive until exit or command completion (since 3.2.0).
The worker inherits the master's prompt mode. The `prompt` command supports
`n`, `i`, and `p` modes.

Persistent sessions can subscribe to event rings. The `dpapi` ring introduced
with this facility initially carries ACME notifications.

## Runtime backend lifecycle

HAProxy 3.4.0 can create and remove whole backends without a reload. A minimal
creation sequence is:

```text
add backend test-backend from mydefaults mode http
add server test-backend/server1 127.0.0.1:3000 check
enable server test-backend/server1
enable health test-backend/server1
publish backend test-backend
```

Publication is the routing boundary. Disabled or unpublished targets selected
by `use_backend` or `default_backend` are skipped unless `force-be-switch` is
set.

For safe deletion, put each server into maintenance, wait for
`srv-removable`, and delete it. Then unpublish the backend, wait for
`be-removable`, and delete it. Named `defaults` sections remain available as
creation templates unless global `tune.defaults.purge` releases them.

## Runtime certificate operations

For the experimental HTTP-01 ACME workflow introduced in 3.2.0:

- `acme renew @my_files/example` starts issuance.
- `acme status` lists ACME tasks.
- `dump ssl cert @my_files/example` returns the in-memory certificate so it can
  be persisted.

DNS-01 automation through HAProxy Data Plane API 3.3 can save issued
certificates to the filesystem. `haproxy-dump-certs`, introduced in 3.3.0,
writes certificates obtained through stats or master sockets.

Since 3.3.0, `add ssl crt-list` does not require a certificate filesystem path
to match its in-memory name. This permits `crt-store` aliases, but moves the
identity check to the caller: confirm that the supplied path or alias selects
the intended certificate before modifying the crt-list.

## Runtime inspection and trace control

Tracing has Runtime API control and supported sources for HTTP versions, QUIC,
QMux, FastCGI, SPOP, peers, and checks. `ssl` was added in 3.2.0 and `acme` in
3.3.0. Scope a trace narrowly and stop it after collection.

Runtime inspection additions in 3.3.0 include:

- `show dev` for thread-to-CPU bindings;
- `show info` counters for lines added to and removed from Map and ACL files;
- `show stat typed` flags of `P` for persistent shared-memory statistics and
  `V` for volatile statistics.

The command line accepts `-vq` for the version, `-vqs` for the short version,
and `-vqb` for the branch.

## Mutable Lua pattern references

The Lua `patref` API introduced in 3.2.0 obtains a mutable ACL or Map file
reference with `core.get_patref`:

```lua
local ref = core.get_patref("virt@cached_paths.txt")
if ref ~= nil then
    ref:add(txn.f:path())
end
```

A reference can:

- add and remove patterns;
- replace Map values;
- perform bulk additions;
- register event callbacks;
- replace a whole file atomically by staging with `prepare()` and publishing
  with `commit()`.

Check for `nil` before use and prefer prepare/commit when readers must not see
a partially replaced data set.

## Lua boolean samples

Lua fetches still convert boolean samples to integers `0` and `1` by default.
Opt in to true Lua booleans with the 3.2.0 setting:

```haproxy
global
    tune.lua.bool-sample-conversion normal
```

Code that compares against `0` or `1` must be updated when normal conversion
is enabled.

## Timed TCP applet receives

`AppletTCP:receive()` accepts an optional timeout since 3.2.0. A timed receive
lets an interactive TCP service regain control for periodic work instead of
blocking indefinitely for client input. Treat a timeout separately from EOF or
an application payload.
