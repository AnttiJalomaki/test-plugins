# Operations, Upgrades, and Recovery

## Configuration drift

Since 2.11.0, the server's `-t` flag generates a hash of its configuration
file, and `varz.config_digest` reports the running configuration's hash.
Compare them to detect an on-disk change that has not been loaded.

## Health checks and shutdown

Since 2.11.0:

- `js-server-only` no longer checks meta-leader health. Use `js-meta-only`
  when the intended signal is meta-group health.
- Graceful `SIGTERM` shutdown exits with status `0`.

Supervisors and probes should not interpret a successful graceful shutdown as
a failure, and should select the JetStream check that matches the component
whose health matters.

## Stream-state downgrade rebuilds

The first 2.11.0-to-2.10 restart rebuilds changed stream-state files by
rescanning message blocks. No data is lost, but CPU use rises and the node
takes longer to become healthy.

The first 2.12.0-to-2.11 restart performs the same kind of rescan. Use 2.11.9
or newer as the downgrade target so assets using features introduced in 2.12
are placed safely offline.

Allow additional maintenance and health-check time for these first restarts.

## Replicated in-memory recovery

Since 2.12.0, recovery of a replicated in-memory stream after all but one
replica have restarted may require every replica to be available, rather than
only a quorum, while the server chooses the state that preserves data.

Do not assume quorum availability is sufficient for this recovery case.

## Filestore memory sizing

Since 2.12.0, elastic filestore caches can be released under memory pressure.
Resident memory can therefore be higher or lower than the earlier pattern,
depending on workload.

Size `GOMEMLIMIT` for the memory actually available to the server, including
container reservations, instead of deriving it from historical RSS behavior.

## Filestore write failures

Since 2.14.0, a filestore write error freezes only the affected stream. The
server:

- logs a `write error`;
- fails health checks;
- continues serving core traffic and other streams.

A replicated stream can fail over, but the affected server must restart to
recover.

## Raft overload containment

Since 2.14.0, Raft bounds memory and disk growth when proposals arrive faster
than they can be committed. A lagging leader steps down so a healthier peer
can take over.

If a majority is overloaded, no healthy peer can restore progress and the
cluster remains degraded until capacity catches up.
