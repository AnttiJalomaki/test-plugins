# Routing, Docking, and Behaviors

## Route Server

`nav2_route` computes and tracks routes over a predefined graph. It can replace
free-space global planning, or provide long-range graph structure while a
planner supplies the locally feasible path. Route progress can invoke
contextual operations on node and edge events, such as changing a speed limit
or activating equipment.

Choose graph edges and event operations with failure recovery in mind: graph
progress describes route context, while the local planner remains responsible
for nearby geometric feasibility when the two are combined.

### Corner smoothing

Route Server can replace graph corners with tangent circular arcs by enabling
`smooth_corners` and configuring `smoothing_radius`. It falls back to linear
interpolation for nearly straight edges or when an arc does not fit within its
adjacent edges. The requested radius is therefore a preference rather than a
guarantee at every graph node.

## Loopback simulation

`nav2_loopback_sim` integrates commanded velocity into ideal odometry for tests
and high-level simulations without physics or localization error.

In Lyrical it is a C++ node with an embedded clock publisher. Launch only
`loopback_simulator`, rather than a separate clock component. `speed_factor` is
dynamically adjustable, and these additional parameters are available:

- `publish_scan`
- `odom_publish_dur`
- `scan_noise_std`

Use it for navigation-logic tests, not for validating wheel slip, collision
dynamics, localization degradation, or actuator response.

## Docking capabilities and migration

Docking supports charging and non-charging static infrastructure as well as
dynamic docks. Available integration points include a
`simple_non_charging_dock` plugin and an RViz docking panel.

Docking-server collision checking is enabled by default. The former server
parameter `dock_backwards` moves to each plugin as `dock_direction`, whose
values are `forward` (the default) or `backward`.

For simple plugins, `reverse_to_dock: true` permits detection from a forward
staging pose followed by dead-reckoning backward into the dock. This is
different from globally reversing all docking motion; configure the plugin's
direction and detection approach together.

### Lyrical detection and plugin changes

External detection rotations for simple dock plugins change from the order
Rz→Rx→Ry to Rx→Ry→Rz. Recalculate any non-default transform configuration that
uses all axes; copying the same angles changes the resulting rotation.

Custom `ChargingDock` and `NonChargingDock` plugins must implement:

- `startDetectionProcess()`
- `stopDetectionProcess()`

Simple plugins also add `detector_service_name`, `detector_service_timeout`,
and `subscribe_toggle` for on-demand perception.

## Following Server

The `opennav_following` server follows either a dynamically detected object or
a named reference frame while maintaining a configured distance. It can use
topic-based detections or TF tracking. Choose the input mode based on whether
the target has a stable transform identity; configure loss-of-target behavior
and safety components around that choice.

## Behavior Server context

Behavior plugins share raw local and global costmaps, their published
footprints, and TF frame names. The server runs plugins at `10.0` Hz by default
and uses a `0.1`-second transform tolerance.

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

Verify that raw costmaps, published footprints, and frames all represent the
same robot namespace and timestamp domain.

## Default behaviors and request-owned inputs

Without an override, Behavior Server loads Spin, BackUp, DriveOnHeading, and
Wait. Configured plugin names also become their action-server names.

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

Wait duration and Spin distance come from the action request. BackUp and
DriveOnHeading also receive distance, speed, and time allowance from each
request rather than server parameters.

`DriveOnHeading`, `BackUp`, and `Spin` accept
`disable_collision_checks`, default `false`. The first two also have these
motion defaults:

```yaml
acceleration_limit: 2.5
deceleration_limit: -2.5
minimum_speed: 0.10
```

Disabling collision checks transfers safety responsibility outside the
behavior; it should be an explicit, bounded operational choice.

## Spin limits

Spin projects collision risk `2.0` seconds ahead by default. Its default
angular limits are:

```yaml
simulate_ahead_time: 2.0
min_rotational_vel: 0.4
max_rotational_vel: 1.0
rotational_acc_lim: 3.2
```

The velocity values are in rad/s and the acceleration limit is in rad/s².
Check the footprint, costmap update rate, and actuator limits when changing the
projection horizon or acceleration.

## Assisted teleoperation

`AssistedTeleop` is not in the default behavior list and must be added
explicitly. It listens on `cmd_vel_teleop` and, by default, projects motion for
`1.0` second in `0.1`-second steps.

```yaml
behavior_plugins: [spin, backup, drive_on_heading, wait, assisted_teleop]
assisted_teleop:
  plugin: nav2_behaviors::AssistedTeleop
projection_time: 1.0
simulation_time_step: 0.1
cmd_vel_teleop: cmd_vel_teleop
```

Treat this as collision-aware command projection, not autonomous path
planning; the operator remains the source of the desired command.
