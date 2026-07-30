# Upgrades and Maintenance

## Configuration and process compatibility

### Parsing and reload ownership

Workers parse their own configuration as of 3.1.0; the master only starts the
workers. This removes the master's former parse-and-undo path and avoids the
associated reload inconsistencies and file-descriptor leaks.

### Deprecated-directive policy

Deprecated directives warn starting in 3.1.0 unless the global
`expose-deprecated-directives` option is enabled. Use that option only as a
short migration aid; it exposes old directives rather than making them a good
long-term configuration choice.

The compatibility sequence is LTS-aware. A supported configuration on a
non-LTS line is expected to continue working on the next LTS line, and that LTS
does not normally add warnings for supported configuration that was
warning-free on the preceding non-LTS. Deprecation ordinarily begins with a
warning on a non-LTS and removal becomes an error on the next non-LTS, at least
one year later.

Specific migrations are:

- `program` sections and legacy C mailers were deprecated for removal in 3.3;
  use Lua mailers.
- The OpenTracing filter was scheduled for deprecation in 3.3 and removal in
  3.5. In 3.4.0 OpenTracing is officially deprecated; use the OpenTelemetry
  add-on.
- Backend `dispatch` and `option transparent` warn as deprecated in 3.3.0 and
  are planned for removal in 3.5.
- Replace global `tune.quic.frontend.*` names with `tune.quic.fe.*`.
- Replace the global `master-worker` directive with `-W` or `-Ws` on the
  command line.
- `tune.disable-fast-forward` is stable in 3.3.0 and no longer requires
  `expose-experimental-directives`.

To replace `dispatch <address>`, configure a regular server named `dispatch`
at the same address. If legacy servers must remain in the backend, set their
weights to zero to retain dispatch semantics:

```haproxy
backend legacy_dispatch
    server dispatch 192.0.2.10:8080
```

To replace `transparent` or `option transparent`, define a server at
`0.0.0.0`. This preserves forwarding to the original TPROXY destination:

```haproxy
backend original_destination
    server tproxy 0.0.0.0
```

### Errors and warnings to resolve before an upgrade

Duplicate names across proxy-section families (`frontend`, `listen`,
`backend`, `defaults`, and `log-forward`) and duplicate server names warn in
3.1.0 and become errors in 3.3.0. HAProxy 3.1 otherwise adds no breaking
changes.

Empty arguments, including empty environment variables expanded inside double
quotes, warn in 3.2.0 and are scheduled to become errors in the next version.
Use `${NAME[*]}` for an intentionally empty environment expansion.

An ACL can no longer specify multiple match types after `-m` in 3.3.0. The
configuration fails instead of silently using the final type. Ambiguous forms
such as `path_beg -m reg` also warn; rewrite them with one coherent match
method.

`http-send-name-header` may no longer target `connection`, `content-length`,
`host`, or `transfer-encoding` in 3.3.0, because replacing those fields would
create an invalid HTTP request.

Startup diagnostics added in 3.3.0 warn when:

- HAProxy runs as root without a global `user` directive;
- `expose-experimental-directives` remains enabled but no configured feature
  needs it;
- a `thread-groups` range exceeds `nbthreads` and must be trimmed; or
- a static build uses `user` or `group` where `uid` or `gid` is appropriate.

An empty thread group after trimming is fatal.

### Changed automatic behavior

Starting in 3.3.0, a backend without `balance` uses `random` rather than
`roundrobin`. The random policy samples two servers and selects the less-loaded
one. Set `balance roundrobin` to preserve the previous behavior. In 3.4.0, an
equal concurrent-connection count is broken using recent HTTP request rates;
this changes distribution in large pools with many apparent ties.

HTTP-mode backends enable `option abortonclose` by default in 3.3.0, so HAProxy
can stop work before an abandoned client request reaches a server. The option
also becomes valid in a frontend.

The global `dns-accept-family`, introduced in 3.2.0, defaults to `auto` in
3.3.0. See the networking reference for probe behavior and explicit choices.

### Command and build changes

The `linux-glibc` target requires Linux 4.17 as of 3.3.0 to support Kernel TLS.
The `halog` utility is installed with `make install-admin`, not `make install`.

Version-only command formats added in 3.3.0 are:

- `haproxy -vq` for the version;
- `haproxy -vqs` for the short version; and
- `haproxy -vqb` for the branch.

The Stats page stops showing the HAProxy version by default in 3.4.0. Add
`stats show-version` to expose it.

## Master-worker operation

For crash investigation, `master-worker no-exit-on-failure` in 3.3.0 keeps the
remaining workers running after one worker segfaults rather than terminating
all of them. The default `mworker-max-reloads` is 50.

## Selecting and maintaining a branch

Since 1.8, HAProxy normally publishes two feature branches each year.
Even-numbered feature branches are LTS lines maintained for five years.
Odd-numbered feature branches are stable lines maintained for roughly 12–18
months and suit operators able to upgrade and roll back more frequently.

Pick the feature branch separately from its patch release. Bug fixes are
conservatively backported within a maintained branch, so keeping the final
component current does not require adopting new features. Reproduce a problem
on the newest patch release before reporting it.

The branch-maintenance snapshot lists these states:

| Branch snapshot | Maintenance state |
| --- | --- |
| 3.4.2 | Fully maintained LTS through 2031-Q2 |
| 3.3.12 | Fully maintained stable through 2027-Q1 |
| 3.2.21 | Fully maintained LTS through 2030-Q2 |
| 3.0.25 | Fully maintained LTS through 2029-Q2 |
| 2.8.26 | Critical fixes only through 2028-Q2 |
| 2.6.31 | Critical fixes only through 2027-Q2 |

All other released branches in that snapshot are unmaintained.

## Reading maintenance queues

The pending-fixes table contains changes already queued for the next patch of
that maintenance branch. A separate list of fixes made later on the development
branch is only a candidate set: a candidate may not affect the maintenance
branch, and applicable fixes land on development before backporting. At the
snapshot, the newest 3.4, 3.3, and 3.2 releases each had zero queued known bugs
even though later development fixes were listed.

Use severity to decide urgency:

- `MINOR`: limited effect and seldom a reason to update by itself.
- `MEDIUM`: normally update or temporarily disable the affected feature.
- `MAJOR`: upgrade as soon as possible.
- `CRITICAL`: short-term reliability or security impact with no workaround;
  expect an immediate release and upgrade.
