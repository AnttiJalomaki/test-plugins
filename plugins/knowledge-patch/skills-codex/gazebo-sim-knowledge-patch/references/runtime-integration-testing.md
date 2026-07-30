# Runtime, Integration, and Testing

## Start and stop servers

- Gazebo Sim handles `SIGTERM` gracefully (9.1.0), including termination by a
  service manager or container runtime.
- Startup detects an already-running server (9.2.0).
- `Server` behavior for a bad SDF file was corrected in 10.1.1. Do not treat
  invalid-SDF behavior from the affected regression as the intended contract.

## Select transport and ROS interfaces

Jetty can use Zenoh as an alternative Gazebo Transport implementation
(`jetty-highlights`). Select it per process:

```sh
export GZ_TRANSPORT_IMPLEMENTATION=zenoh
```

Gazebo's ROS integration supports the community standard simulation interface
(`jetty-highlights`), allowing robot code written to that interface to move
between compatible simulators.

## Export occupancy grids

Gazebo can export occupancy-grid maps through the `/scan_image` topic
(`jetty-highlights`). Start exploration by publishing:

```sh
gz topic -t /start_exploration -m gz.msgs.Boolean -p 'data: true'
```

## Handle commands and reset

- UserCommands services return the actual status of the command they execute
  (10.0.0). Treat the service result as command success or failure.
- Simulation reset is exposed through a public callable API (10.0.0).
- The test fixture supports `ISystemReset` (10.0.0), enabling reset-aware
  fixture systems.

## Load physics engines and host WebSockets

- Physics-engine plugins can load from the static plugin registry (10.0.0).
- The WebSocket server formerly housed in `gz-launch` is now owned by Gazebo
  Sim (10.0.0).
- The WebSocket server protocol includes top-level enums (10.1.0), exposing
  those enum declarations to schema consumers.

## Run distributed simulations

`NetworkManager` correctly creates entities on network secondaries (10.1.0).
Distributed-simulation code should expect secondary nodes to receive those
entities.

## Embed Python systems and fixtures

Python systems and the Python `TestFixture` use corrected GIL-release behavior
(9.2.0). Avoid workarounds that assume the earlier GIL handling.
