# Servers, behaviors, and tools

Source batch attribution: `overview-and-distro-migrations`,
`behaviors-and-algorithm-selection`.

## Route Server

`nav2_route` computes and tracks routes over a predefined graph. It may replace
free-space global planning or provide long-range structure while a planner
produces the locally feasible path. Route progress can trigger contextual
operations on node and edge events, such as speed changes or equipment
activation.

Enable `smooth_corners` and set `smoothing_radius` to replace graph corners
with tangent circular arcs. Route Server falls back to linear interpolation
for nearly straight edges or when an arc does not fit inside its adjacent
edges.

## Loopback simulation

`nav2_loopback_sim` integrates commanded velocity into ideal odometry for
tests and high-level simulations that do not need physics or localization
error. In Lyrical it is a C++ node with an embedded clock publisher, so launch
only `loopback_simulator`. `speed_factor` is dynamically adjustable, and
`publish_scan`, `odom_publish_dur`, and `scan_noise_std` are available.

## Docking

Docking supports non-charging static infrastructure and dynamic docks,
including `simple_non_charging_dock` and an RViz docking panel.
Docking-server collision checking is enabled by default.

`dock_backwards` moved from the server into each plugin as `dock_direction`,
whose default is `forward` and alternative is `backward`.
`reverse_to_dock: true` allows simple plugins to detect from a forward staging
pose and then dead-reckon backward into the dock.

In Lyrical, external detection rotations for simple dock plugins change from
Rz→Rx→Ry to Rx→Ry→Rz. Recalculate non-default configurations that use all
three axes. Custom `ChargingDock` and `NonChargingDock` implementations must
provide `startDetectionProcess()` and `stopDetectionProcess()`. Simple plugins
also provide `detector_service_name`, `detector_service_timeout`, and
`subscribe_toggle` for on-demand perception.

## Behavior Server

Behavior plugins share the raw local and global costmaps, their published
footprints, and TF frame configuration. The server defaults to `10.0` Hz and
a `0.1`-second transform tolerance:

```yaml
behavior_server:
  ros__parameters:
    local_costmap_topic: local_costmap/costmap_raw
    global_costmap_topic: global_costmap/costmap_raw
    local_footprint_topic: local_costmap/published_footprint
    global_footprint_topic: global_costmap/published_footprint
    local_frame: odom
    global_frame: map
    robot_base_frame: base_link
    cycle_frequency: 10.0
    transform_tolerance: 0.1
```

Without an override, the server loads Spin, BackUp, DriveOnHeading, and Wait.
Configured plugin names become their action-server names. Wait duration, Spin
distance, and the BackUp or DriveOnHeading distance, speed, and time allowance
belong to each action request rather than server parameters.

```yaml
behavior_plugins: [spin, backup, drive_on_heading, wait]
spin:
  plugin: nav2_behaviors::Spin
backup:
  plugin: nav2_behaviors::BackUp
drive_on_heading:
  plugin: nav2_behaviors::DriveOnHeading
wait:
  plugin: nav2_behaviors::Wait
```

`DriveOnHeading`, `BackUp`, and `Spin` accept
`disable_collision_checks`, default false. The first two also provide
`acceleration_limit: 2.5`, `deceleration_limit: -2.5`, and
`minimum_speed: 0.10`.

Spin projects collision risk `2.0` seconds ahead by default and bounds angular
motion to `0.4`–`1.0` rad/s with a `3.2` rad/s² acceleration limit:

```yaml
simulate_ahead_time: 2.0
min_rotational_vel: 0.4
max_rotational_vel: 1.0
rotational_acc_lim: 3.2
```

AssistedTeleop is not loaded by default. Add it explicitly when needed. It
subscribes to `cmd_vel_teleop` and checks projected motion for `1.0` second in
`0.1`-second steps by default.

```yaml
behavior_plugins: [spin, backup, drive_on_heading, wait, assisted_teleop]
assisted_teleop:
  plugin: nav2_behaviors::AssistedTeleop
projection_time: 1.0
simulation_time_step: 0.1
cmd_vel_teleop: cmd_vel_teleop
```

## Vector objects and dynamic following

The Vector Object Server in `nav2_map_server` rasterizes configured circles,
polygons, and polygonal chains into an `OccupancyGrid`. Its `AddShapes`,
`GetShapes`, and `RemoveShapes` services support dynamic virtual obstacles,
keepout regions, and speed-filter masks.

The `opennav_following` server follows a dynamic detected object or a named
reference frame while maintaining configured distance. It can use topic-based
detections or TF tracking.

## RViz and isolated tests

The Nav2 RViz panel can select BT XML per request, accept exact coordinates
and frame IDs, and build, edit, save, or load multi-goal lists for
`NavigateThroughPoses` and Waypoint Following.

Build with `--cmake-args -DUSE_ISOLATED_TESTS=ON` to run
`rmw_zenoh_cpp` tests without launching a separate Zenoh router.
