---
name: ros2-navigation-nav2-knowledge-patch
description: Nav2
version: 1.3.12
license: MIT
metadata:
  author: Nevaberry
---


# Nav2 compatibility guide

Use this skill when configuring, migrating, extending, or debugging Nav2.
Inspect the project's ROS distribution, parameter files, behavior trees,
plugin implementations, and message types before applying guidance. Preserve
the project's observed interfaces when they differ from an assumed default.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-interfaces.md](references/migrations-and-interfaces.md) | Distribution posture, stamped velocity, action errors, multi-pose messages, namespaces, lifecycle wrappers |
| [behavior-trees-and-navigation.md](references/behavior-trees-and-navigation.md) | BT node and port migrations, control nodes, selectors, subtree loading, validation, replanning |
| [planning-control-and-smoothing.md](references/planning-control-and-smoothing.md) | Planner and controller choice, path handling, goal checking, RPP, MPPI, Rotation Shim, smoothers |
| [servers-behaviors-and-tools.md](references/servers-behaviors-and-tools.md) | Route, loopback, docking, Behavior Server, following, vector objects, RViz, tests |
| [costmaps-and-localization.md](references/costmaps-and-localization.md) | Costmap APIs, layers, filters, clearing, transport, inflation, AMCL, transforms |
| [collision-monitor.md](references/collision-monitor.md) | Command pipeline, fail-safe timeouts, zones, observation sources, costmap sources |

## Migration triage

Check these surfaces first when an existing system stops configuring,
activating, or exchanging messages.

### Align command-velocity types

Nav2 uses `geometry_msgs/TwistStamped` by default on the existing `cmd_vel`
topic names. Either migrate the complete pipeline or set
`enable_stamped_cmd_vel: false` on every affected node. A mixed pipeline does
not interoperate.

### Replace action-error configuration

Remove `error_code_names`; it now causes startup failure. Configure
`error_code_name_prefixes`, then wire both `error_code_id` and `error_msg`
ports on the corresponding BT action nodes.

```yaml
error_code_name_prefixes: [compute_path, follow_path, spin, route]
```

### Update multi-goal messages

Use `nav_msgs/Goals` for navigation-through-poses and
compute-path-through-poses flows. Read navigation poses from `poses.goals`
and planning goals from `goals.goals`. Keep `WaypointStatus` lists synchronized
when pruning goals.

### Remove obsolete bringup switches

Delete `use_namespace`. `namespace` is always applied, and relative topic names
resolve under it. Use absolute topics only when they intentionally bypass the
robot namespace.

Move `map_topic` from `Costmap2DROS` to the `StaticLayer` configuration.

```yaml
static_layer:
  plugin: "nav2_costmap_2d::StaticLayer"
  map_topic: my_map
```

### Migrate custom lifecycle code

For Lyrical custom plugins and task servers, use `nav2::LifecycleNode` and the
node's `create_*` factories from `nav2_ros_common`. Update service callbacks
for the `rmw_request_id_t` header, place explicit subscription QoS after the
callback, and remove `action_server_result_timeout`.

### Update controller plugin interfaces

Controller Server owns transformed-plan pruning through a path-handler plugin.
Move path-handling parameters out of individual controllers. Update custom
controllers and goal checkers for `newPathReceived()`, the processed-plan and
global-goal command inputs, and the transformed-plan goal-checker input.

### Correct inverted path-validation naming

Replace `check_full_path` with `stop_at_first_collision`. The polarity is
opposite: old false maps to new true. A negative
`max_lookahead_distance` retains full-path validation.

### Migrate MPPI motion models

Replace old motion-model string values with named plugin groups:

```yaml
motion_model: "diff_drive"
diff_drive:
  plugin: "mppi::DiffDriveMotionModel"
```

Remove `publish_critics_stats`; visualization mode publishes the statistics
automatically.

### Retune corrected smoothing costs

Constrained Smoother now applies `weight * residual²`. Retune existing weights
instead of carrying forward values tuned against the formerly quartic
behavior.

## Defaults that change behavior

- `RoundRobin` defaults to `wrap_around="false"` and fails after the final
  child.
- `bond_heartbeat_period` defaults to `0.25` seconds.
- Controller Server's `control_frequency` is startup configuration, not a
  dynamically mutable setting.
- Rotation Shim `closed_loop` defaults to true and uses odometry.
- Docking collision checking is enabled by default.
- `publish_zero_velocity` defaults to true on Controller Server.
- Partial multi-pose planning is disabled unless `allow_partial_planning` is
  enabled.
- Service introspection and navigator live-monitoring features are disabled
  unless configured.
- Savitzky-Golay Smoother exposes `window_size: 7` and `poly_order: 3`.

## High-value behavior-tree capabilities

Use runtime selectors to switch progress checker, goal checker, path handler,
controller, and planner IDs through selector topics without replacing the
tree.

Use `NonblockingSequence` when later children must tick while an earlier child
is running. Pair `PauseResumeController` with `PersistentSequence` when a
paused flow must resume at its previous child.

Store blackboard-ID parameters under each navigator plugin. Load reusable
subtrees from `bt_search_directories` and give selectable trees unique IDs.

Near the goal, preserve a valid remaining path when the goal has not changed.
The standard gated fallback uses `GlobalUpdatedGoal`, `IsGoalNearby`,
`TruncatePathLocal`, and `ValidatePath` before deciding to replan.

## Planning and controller selection

Match the algorithm to both footprint and kinematics:

- NavFn, Smac 2D, and Theta Star suit circular differential or
  omnidirectional robots.
- Smac Hybrid-A* provides full-footprint, curvature-aware planning for
  arbitrary-shaped Ackermann and legged platforms.
- Smac Lattice supplies kinematically valid control sets for differential,
  omnidirectional, and Ackermann robots.
- DWB ordinarily serves differential and omnidirectional motion.
- RPP prioritizes exact path following but is not omnidirectional.
- MPPI covers differential, omnidirectional, Ackermann, and legged motion with
  model-predictive control.
- Vector Pursuit targets high-speed or resource-constrained path tracking.

Controller Server waits for costmap freshness before control, supports
bounded plugin-failure tolerance, and can run its controller thread at
real-time priority `90` when the operating-system limits permit it.

Use `goal_heading_mode` when Smac should consider multiple arrival
orientations in one request. Use goal-checker capabilities rather than
controller-specific state to preserve XY completion during final yaw
alignment.

## Costmap and localization checkpoints

Keep ordinary layers in `plugins` and keep filters in `filters`; filters run
after the combined layered costmap. Selective clear requests are atomic: an
invalid or non-clearable plugin name makes the request fail without clearing
anything.

Compressed point-cloud transport is available to obstacle and voxel layers
and Collision Monitor. Leave `transport_type: raw` unless the source and
consumer share a configured compressed transport.

Use a nonnegative AMCL `random_seed` for repeatable tests. Decide independently
whether reset requires a new initial pose and whether replacement maps should
be accepted.

Treat `custom_inscribed_radius: 0.0` as a specialized representation; it
bypasses the normal inscribed region and is unsafe for ordinary planning and
control algorithms.

## Collision Monitor safety checklist

Confirm the command chain enters Collision Monitor on `cmd_vel_smoothed` and
leaves on `cmd_vel`. Keep `source_timeout` enabled unless another mechanism
provides equivalent stale-sensor fail safety.

Order overlapping `velocity_polygon` entries deliberately because the first
matching range wins, and finish with a range that covers every command.

When migrating a Humble zone threshold, convert the largest safe
`max_points` value to the smallest triggering count with
`min_points = max_points + 1`.

Use the `Toggle` service only with the understanding that it disables polygons
while sensor checking remains active. Treat a `CostmapSource` carefully:
`cost_threshold` and `treat_unknown_as_obstacle` directly determine which map
cells become collision evidence.

## New servers and operational tools

Use Route Server for navigation over a predefined graph and contextual
node/edge operations. It can provide the entire global route or long-range
structure for a nearby free-space planner.

Use Loopback Simulator for ideal odometry without physics or localization
error. In Lyrical, launch only `loopback_simulator` because it includes the
clock publisher.

Docking supports charging and non-charging infrastructure, dynamic docks,
and per-plugin docking direction. Recalculate custom three-axis detector
rotations for Lyrical and implement the detection start/stop hooks in custom
dock plugins.

Use Vector Object Server to rasterize dynamic geometric regions into an
`OccupancyGrid`. Use `opennav_following` for topic-detected or TF-tracked
dynamic targets.
