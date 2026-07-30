# Upgrades and Operations

## Compatibility rules

HAProxy's deprecation sequence is LTS-aware. A working supported setup on a
non-LTS branch is intended to continue working on the next LTS, and that LTS
does not add warnings for supported configuration that was warning-free on the
preceding non-LTS. A feature normally warns first in a non-LTS and becomes an
error when removed in the following non-LTS, at least a year later.

Deprecated directives warn since 3.1.0 unless the global
`expose-deprecated-directives` option is enabled. Use that switch for a bounded
migration window, not to hide upgrade work.

### Name uniqueness

Duplicate names began warning in 3.1.0 and became errors in 3.3.0. This covers
names across the proxy-section families `frontend`, `listen`, `backend`,
`defaults`, and `log-forward`, plus duplicate server names. HAProxy 3.1.0 had
no other breaking configuration changes.

### Removed and deprecated facilities

- `program` sections and legacy C mailers were deprecated in 3.1.0 and removed
  in 3.3.0. Lua mailers are the supported replacement.
- The OpenTracing filter was scheduled for deprecation in 3.1.0. It is
  officially deprecated as of 3.4.0, with removal planned for 3.5; use the
  OpenTelemetry add-on.
- Backend `dispatch` and `option transparent` warn as deprecated since 3.3.0.
- Global names beginning with `tune.quic.frontend` are deprecated; use
  `tune.quic.fe`.
- The global `master-worker` directive is deprecated; use command-line `-W` or
  `-Ws`.
- `no-quic` was replaced by `tune.quic.listen on|off` in 3.3.0.
- `compression-direction` is deprecated after the 3.4.0 split into request and
  response compression filters.
- `tune.takeover-other-tg-connections` is superseded and deprecated by
  `tune.idle-pool.shared` in 3.4.0.

### Dispatch migrations before 3.5

Replace a fixed `dispatch` target with a regular server named `dispatch` at the
same address. If legacy servers must remain in the backend, set their weights
to zero to preserve dispatch behavior:

```haproxy
backend legacy_dispatch
    server dispatch 192.0.2.10:8080
```

Replace `transparent` or `option transparent` with a zero-address server. This
retains routing to the original TPROXY destination:

```haproxy
backend original_destination
    server tproxy 0.0.0.0
```

## Defaults and stricter validation

### Balancing and abort behavior

Since 3.3.0, a backend with no `balance` directive uses `random`, not
`roundrobin`. The policy samples two servers and chooses the less-loaded one.
Set `balance roundrobin` explicitly to preserve the former behavior. Since
3.4.0, a tie in concurrent connections is broken using recent HTTP request
rates, affecting distribution in large lightly loaded pools.

HTTP backends enable `option abortonclose` by default since 3.3.0, so HAProxy
can stop work before an abandoned request reaches a server. The option is also
valid in a frontend. Disable it explicitly only where server-side completion
is required despite client departure.

### Arguments, ACLs, and headers

Empty configuration arguments, including empty environment variables inside
double quotes, warn since 3.2.0 and were scheduled to become errors in the next
version. Use `${NAME[*]}` for an intentionally empty expansion.

Since 3.3.0, an ACL cannot specify multiple match types after `-m`; the
configuration fails instead of silently choosing the last type. Ambiguous
forms such as `path_beg -m reg` also warn.

`http-send-name-header` can no longer name `connection`, `content-length`,
`host`, or `transfer-encoding`, because replacing them would make the outgoing
HTTP request invalid.

## Master-worker and reload behavior

Since 3.1.0, the master only starts workers and workers parse configuration.
This removes master-side parse-and-undo work and prevents the associated reload
inconsistencies and file-descriptor leaks.

For crash investigation, `master-worker no-exit-on-failure` keeps other workers
running when one worker segfaults. The default `mworker-max-reloads` is 50
since 3.3.0.

Dynamic backends keep named `defaults` sections resident for runtime creation.
If the deployment never creates backends dynamically, set global
`tune.defaults.purge` to release that memory.

## CPU and threading changes

Automatic CPU binding in 3.2.0 became topology-aware across packages, NUMA
nodes, CCXs, L3 caches, cores, and hardware threads. That release still
restricted HAProxy to one NUMA node by default, required extra configuration
to use more than 64 threads, and raised limits to 1024 threads and 32 thread
groups.

The 3.3.0 defaults supersede those constraints: `cpu-policy` defaults to
`performance`, heterogeneous systems use only performance cores by default,
automatic placement uses all available cores and NUMA nodes, and the previous
64-thread automatic limit is gone.

Startup diagnostics since 3.3.0 trim oversized `thread-groups` ranges against
`nbthreads` with a warning and reject a group left empty. A root process with
no global `user` warns. Static builds warn when `user` or `group` should be
`uid` or `gid`. Leaving `expose-experimental-directives` set when no selected
feature requires it also warns.

`show dev` reports thread-to-CPU bindings for runtime verification.

## Build and command-line changes

- The default `linux-glibc` target requires Linux 4.17 since 3.3.0, enabling
  Kernel TLS support.
- Install `halog` with `make install-admin`; it is no longer installed by
  `make install` as of 3.3.0.
- Use `haproxy -vq` for the version, `-vqs` for the short version, and `-vqb`
  for the branch.
- `tune.disable-fast-forward` is stable since 3.3.0 and no longer needs
  `expose-experimental-directives`.

## Branch and patch maintenance

Since 1.8, HAProxy normally publishes two feature branches per year. Even
minor-number branches are LTS and maintained for five years. Odd-numbered
branches are short-lived stable branches, usually maintained for about 12 to
18 months, for operators able to upgrade and roll back more frequently.

Choose the feature branch and patch level separately. Conservative backports
mean a maintained branch can receive fixes without taking a new feature
branch. Keep the last component current and reproduce a problem on the latest
patch before reporting it.

At the 2026-07-28 branch-maintenance snapshot:

- 3.4.2 is fully maintained LTS through 2031-Q2.
- 3.3.12 is fully maintained stable through 2027-Q1.
- 3.2.21 is fully maintained LTS through 2030-Q2.
- 3.0.25 is fully maintained LTS through 2029-Q2.
- 2.8.26 receives critical fixes through 2028-Q2.
- 2.6.31 receives critical fixes through 2027-Q2.
- Every other released branch in that matrix is unmaintained.

Treat this list as a dated support snapshot, not a substitute for checking the
current maintenance table.

## Reading maintenance bug queues

The pending-fixes table lists changes already queued for that maintenance
branch's next release. A separate list of later development-branch fixes is
only a candidate set: a fix may not affect the maintenance branch, and an
applicable fix lands on development before backporting. At the snapshot, the
latest 3.4, 3.3, and 3.2 releases each had zero queued known bugs even though
later development fixes were listed.

Use severity to choose urgency:

- `MINOR`: limited impact; rarely update for this alone.
- `MEDIUM`: update or temporarily disable the affected feature.
- `MAJOR`: upgrade as soon as possible.
- `CRITICAL`: short-term reliability or security failure without a workaround;
  expect an immediate release and upgrade.
