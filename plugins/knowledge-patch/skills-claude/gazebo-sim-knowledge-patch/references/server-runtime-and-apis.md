# Server, Runtime, and APIs

## Process and server lifecycle

- Gazebo Sim handles `SIGTERM` gracefully. Service units and container
  runtimes can use their normal termination path instead of requiring a custom
  signal workaround (since 9.1.0).
- Startup detects an already-running server. Treat that as an explicit startup
  condition rather than assuming every process invocation owns a new server
  (since 9.2.0).
- Gazebo Sim 10.1.1 corrects a `Server` regression triggered by bad SDF input.
  Do not treat invalid-SDF behavior from an affected build as the intended
  server contract.

## Python execution

GIL-release behavior is corrected for Python systems and the Python
`TestFixture`. Reassess locking or serialization workarounds that were added
for the older behavior (since 9.2.0).

## Static plugin registries

- System plugins can be loaded through the static plugin registry, making
  statically registered systems available through the standard system-loading
  path (since 9.2.0).
- Physics-engine plugins can also be loaded from the static plugin registry
  (since 10.0.0).

## Commands and Entity Component Manager state

- UserCommands services report the actual status of the command they execute.
  Branch on that value; a successfully delivered service call does not by
  itself mean the requested command succeeded (since 10.0.0).
- The Entity Component Manager provides APIs for clearing its internal
  tracking of additions and removals. Clear tracking after a consumer has
  processed the relevant changes (since 10.0.0).

## WebSocket ownership and protocol

The WebSocket server moved from `gz-launch` into Gazebo Sim. Update launch,
ownership, and debugging assumptions accordingly (since 10.0.0).

Its protocol definitions expose top-level enums for schema consumers (since
10.1.0).

## Distributed simulation

`NetworkManager` creates entities correctly on network secondaries. Distributed
simulation logic can expect secondary entity creation instead of compensating
for the earlier failure (since 10.1.0).

## Time API spelling

Replace `systemTimeISO` with `systemTimeIso`; the former spelling is no longer
the callable API (since 10.0.0).
