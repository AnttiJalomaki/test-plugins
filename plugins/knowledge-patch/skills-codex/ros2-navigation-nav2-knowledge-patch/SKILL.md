---
name: ros2-navigation-nav2-knowledge-patch
description: Nav2
version: 1.3.12
license: MIT
metadata:
  author: Nevaberry
---


# Nav2 Knowledge Patch

Use this skill when configuring, extending, migrating, or debugging Nav2. Start
with the migration checks below, then open the topic reference that matches the
component being changed. Verify distribution-specific behavior against the
project's packages, launch files, parameters, BT XML, and tests.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration and integration](references/migration-and-integration.md) | Distribution status, action errors, velocity messages, multi-pose APIs, namespaces, lifecycle wrappers, introspection, RViz, test builds |
| [Behavior trees](references/behavior-trees.md) | Node renames, control semantics, selectors, blackboards, subtree loading, validation, cancellation, near-goal replanning |
| [Planning and control](references/planning-and-control.md) | Planner/controller selection, Controller Server, SMAC, MPPI, RPP, Rotation Shim, DWB, smoothing, goal checkers |
| [Costmaps, localization, and mapping](references/costmaps-localization-and-mapping.md) | Costmap APIs and layers, filters, transports, map conversion, vector objects, AMCL, footprint and transform settings |
| [Routing, docking, and behaviors](references/routing-docking-and-behaviors.md) | Route Server, docking, loopback simulation, Following Server, Behavior Server, Spin, assisted teleoperation |
| [Collision Monitor](references/collision-monitor.md) | Command pipeline, fail-safe timing, sources, zones, thresholds, velocity polygons, debounce, runtime controls |

## Migration checks first

### Replace removed action-error configuration

`error_code_names` now causes startup failure. Configure prefixes and expose
both error ports on the relevant BT action nodes:

```yaml
error_code_name_prefixes: [compute_path, follow_path, spin, route]
```

```xml
<FollowPath path="{path}"
            error_code_id="{follow_path_error_code}"
            error_msg="{follow_path_error_msg}"/>
```

Preserve the contextual `error_msg` through custom action and navigator code.

### Audit every velocity endpoint

Nav2 command-velocity publishers and subscribers use
`geometry_msgs/TwistStamped` by default without changing topic names. Convert
bridges, muxes, safety nodes, hardware drivers, and tests together. Only retain
the legacy message by setting this on every affected node:

```yaml
enable_stamped_cmd_vel: false
```

### Update multi-pose code and trees

Multi-pose actions use `nav_msgs/Goals`, not a raw `PoseStamped` vector. Read
navigation poses from `poses.goals` and compute-path poses from `goals.goals`.
`NavigateThroughPoses` reports `WaypointStatus` values: `PENDING`, `COMPLETED`,
`SKIPPED`, and `FAILED`. Pruning nodes must carry matching
`input_waypoint_statuses` and `output_waypoint_statuses` ports.

### Fix namespace assumptions

`nav2_bringup` no longer has `use_namespace`. `namespace` is always applied and
defaults to `/`. Relative topics such as `scan` resolve within a robot
namespace; `/scan` remains global. In costmap layers, use
`joinWithParentNamespace()` when a resource should resolve against the costmap
rather than the layer's private namespace.

### Move configuration to its new owner

- Put `map_topic` on the costmap `StaticLayer`, not `Costmap2DROS`.
- Nest the Rotation Shim's plugin and parameters under `primary_controller`.
- Move transformed-plan pruning parameters from DWB, RPP, Graceful, and MPPI to
  Controller Server's path-handler plugin.
- Replace Graceful Controller's `motion_target_dist` with `min_lookahead` and
  `max_lookahead`, and rename `final_rotation` to `prefer_final_rotation`.
- Rename Dynamic Window Pure Pursuit's `desired_linear_vel` to
  `max_linear_vel`.
- Remove `action_server_result_timeout`; it no longer exists.

### Update custom plugin interfaces

Custom task servers and plugins should derive from `nav2::LifecycleNode` and
create Nav2 service, action, publisher, and subscription wrappers through the
node's `create_*` factories. Service callbacks include the
`rmw_request_id_t` header. Custom planner, controller, goal-checker, and docking
plugins have additional signature and lifecycle changes documented in the
topic references.

## Behavior-tree quick reference

### Treat renamed checks as actions

The following nodes may return `RUNNING`, so use their action-style names:

| Earlier name | Current name |
| --- | --- |
| `IsStopped` | `CheckStopStatus` |
| `IsPathValid` | `ValidatePath` |
| `IsPoseOccupied` | `CheckPoseOccupancy` |

`ValidatePath` replaces `check_full_path` with the oppositely worded
`stop_at_first_collision`. Earlier `false` maps to current `true`, which is the
default. A positive `max_lookahead_distance` limits validation; `-1.0` checks
the full path.

### Choose control-node semantics deliberately

- `NonblockingSequence` ticks later children while an earlier child is
  `RUNNING`.
- `PauseResumeController` provides pause/resume services and works with
  `PersistentSequence` to resume from its bidirectional child-index port.
- `RoundRobin` defaults to `wrap_around="false"` and fails after the last child;
  enable wrapping explicitly for legacy behavior.

### Select navigation plugins at runtime

The default tree can select progress checker, goal checker, path handler,
controller, and planner. Each selector writes a plugin ID to its blackboard
port, listens on its selector topic, and needs a valid default plugin ID.

```xml
<ControllerSelector selected_controller="{selected_controller}"
  default_controller="FollowPath" topic_name="controller_selector"/>
<PlannerSelector selected_planner="{selected_planner}"
  default_planner="GridBased" topic_name="planner_selector"/>
```

## Planning and control quick reference

### Match algorithms to geometry

- Use NavFn, Smac 2D, or Theta Star primarily for circular differential or
  omnidirectional robots; they do not guarantee body- or curvature-feasible
  paths for arbitrary shapes.
- Use Smac Hybrid-A* for full-footprint collision checking with Dubins or
  Reeds-Shepp motion, or Smac Lattice for kinematically valid differential,
  omnidirectional, and Ackermann control sets.
- DWB normally targets differential and omnidirectional bases. TEB and MPPI
  cover differential, omnidirectional, Ackermann, and legged robots. RPP and
  Vector Pursuit do not provide omnidirectional motion.

Choose DWB or TEB for dynamic-obstacle avoidance, RPP for exact path tracking,
MPPI for model-predictive control, and Vector Pursuit for fast or
resource-constrained tracking.

### Configure Controller Server timing explicitly

```yaml
controller_server:
  ros__parameters:
    costmap_update_timeout: 0.3
    failure_tolerance: 0.3
    odom_topic: odom
    odom_duration: 0.3
```

`failure_tolerance: 0.0` disables exception tolerance, `-1.0` permits failures
indefinitely, and positive values set a duration in seconds. Controller
Server's `control_frequency` is startup configuration, not a dynamic setting.

### Migrate MPPI motion models

Use a named plugin group even when the selected model is differential drive:

```yaml
motion_model: diff_drive
diff_drive:
  plugin: mppi::DiffDriveMotionModel
```

The plugin form replaces the former `DiffDrive`, `Omni`, and `Ackermann`
strings. Keep trajectory validation separate through
`OptimalTrajectoryValidator`; the default validator checks collisions.

## Costmap and safety quick reference

### Keep filter and layer stages separate

Costmap `filters` run after the ordinary `plugins` have been combined. Define a
`plugin` parameter under every listed filter namespace. Use the Plugin
Container Layer when selected ordinary layers should be grouped before their
result is combined with the parent costmap.

### Preserve fail-safe Collision Monitor behavior

Collision Monitor normally consumes `cmd_vel_smoothed` and publishes the
safety-adjusted command on `cmd_vel`. A stale observation source stops the robot
after `source_timeout` (`2.0` seconds by default); `0.0` disables this check.
`stop_pub_timeout` controls how long zero commands continue. Source-specific
timeouts override the node setting.

```yaml
cmd_vel_in_topic: cmd_vel_smoothed
cmd_vel_out_topic: cmd_vel
source_timeout: 2.0
stop_pub_timeout: 1.0
scan:
  source_timeout: 0.2
```

For velocity polygons, order overlapping ranges intentionally because the
first match wins, and end with a range that covers every command. Simultaneous
zones resolve to the most restrictive action.

## Verification checklist

1. Confirm the installed ROS distribution and package behavior before applying
   distribution-specific migrations.
2. Search launch, YAML, XML, C++, and Python for removed parameter and port
   names.
3. Check topic message types and namespace resolution end to end.
4. Validate plugin IDs, interface signatures, and owning parameter namespaces.
5. Exercise lifecycle startup, transforms, costmap freshness, and stale-sensor
   fail-safe paths in tests.
6. Inspect action errors, waypoint statuses, selected plugins, controller
   trajectories, and Collision Monitor state when diagnosing runtime behavior.
