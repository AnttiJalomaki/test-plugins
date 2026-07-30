# GUI, Transport, and ROS

## Component Inspector and visualization

- Pose attributes can be plotted from the Component Inspector again (since
  9.2.0).
- The Quick Start dialog is disabled by default on Windows (since 10.1.0).

## Selectable transport

Zenoh can be selected as the Gazebo Transport implementation for a process.
Set the environment variable before launching that process
(jetty-highlights):

```sh
export GZ_TRANSPORT_IMPLEMENTATION=zenoh
```

## ROS simulation interface

Gazebo's ROS integration supports the community standard simulation interface.
Robot code written to that interface can move between compatible simulators
without binding its simulation control path to Gazebo-specific APIs
(jetty-highlights).

## Occupancy-grid export

Gazebo can export occupancy-grid maps using `/scan_image`. Begin exploration by
publishing a Boolean start message (jetty-highlights):

```sh
gz topic -t /start_exploration -m gz.msgs.Boolean -p 'data: true'
```

## Wind publication

Wind information can be published to Gazebo and ROS topics for simulation and
external consumers (since 9.2.0).
