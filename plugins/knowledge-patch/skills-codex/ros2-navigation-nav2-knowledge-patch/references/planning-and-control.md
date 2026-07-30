# Planning and Control

## Select algorithms by geometry and task

### Global planners

NavFn, Smac 2D, and Theta Star work in grid space and are recommended for
circular differential or omnidirectional robots. They do not guarantee a
drivable path for a non-circular body or a curvature-constrained Ackermann or
legged platform. NavFn in particular checks a circular radius derived from the
footprint.

Smac Hybrid-A* performs full-footprint collision checking and uses Dubins or
Reeds-Shepp motion for arbitrary-shaped Ackermann and legged robots. Smac
Lattice supplies kinematically valid control sets for differential,
omnidirectional, and Ackermann vehicles of any shape.

### Local controllers

- DWB normally targets differential and omnidirectional robots. Ackermann or
  legged use needs a curvature-aware trajectory generator.
- TEB and MPPI support differential, omnidirectional, Ackermann, and legged
  motion.
- RPP and Vector Pursuit do not provide omnidirectional motion.
- Choose DWB or TEB for dynamic-obstacle avoidance, RPP for exact path
  tracking, MPPI for model-predictive control, and Vector Pursuit for fast or
  resource-constrained tracking.

These are selection constraints, not substitutes for footprint, kinematics,
velocity, and acceleration validation on the actual platform.

## Controller and Planner Server execution

### Costmap freshness deadlines

Controller Server waits for a fully updated local costmap for
`costmap_update_timeout`, default `0.3` seconds, before computing control.
Planner Server uses the same parameter with a `1.0`-second default.

```yaml
controller_server:
  ros__parameters:
    costmap_update_timeout: 0.3
planner_server:
  ros__parameters:
    costmap_update_timeout: 1.0
```

A short deadline can reject work during delayed sensor or transform updates; a
long deadline increases worst-case action latency.

### Soft real-time execution

`use_realtime_priority: true` raises the controller execution thread to
priority `90`. The option defaults to `false`. Before enabling it, give the
process user an adequate `rtprio` limit, for example:

```text
robot soft rtprio 99
robot hard rtprio 99
```

Confirm the deployed process actually runs as that user and can acquire the
priority; otherwise configuration alone does not provide real-time scheduling.

### Failure tolerance

`failure_tolerance` controls how long exceptions from a controller plugin may
continue before `FollowPath` fails:

- `0.0` (default): no tolerance.
- `-1.0`: tolerate failures indefinitely.
- Positive value: tolerance in seconds.

```yaml
controller_server:
  ros__parameters:
    failure_tolerance: 0.3
```

### Odometry speed estimate

Controller Server buffers odometry for `odom_duration` seconds to estimate
robot speed. The default topic is `odom` and the default window is `0.3`
seconds.

```yaml
controller_server:
  ros__parameters:
    odom_topic: odom
    odom_duration: 0.3
```

### Frequency and final command

`control_frequency` is no longer dynamically changeable; set it before
startup. `publish_zero_velocity` defaults to `true`, causing Controller Server
to publish a final zero command. Disable it only when another component owns
stop-command publication and the command pipeline remains safe.

## Centralized path handling

Controller Server owns transformed-plan pruning through a configurable
path-handler plugin such as `nav2_controller::FeasiblePathHandler`. Move
path-transformation and pruning parameters formerly under DWB, RPP, Graceful,
or MPPI to the server's path handler.

Custom interfaces change with this ownership:

- `GoalChecker::isGoalReached()` receives the transformed plan.
- `Controller::setPlan()` is replaced by lightweight `newPathReceived()`.
- `computeVelocityCommands()` receives the processed plan and global goal.

Do not retain a second independent pruning implementation in the controller;
it can make the controller's goal and the goal checker's plan disagree.

## Goal checkers

`PositionGoalChecker` ignores goal orientation.

Kilted RPP's `stateful` mode retained XY completion while yaw was aligned. In
Lyrical, that controller parameter is removed and the behavior moves into goal
checkers through `isGoalXYReached()`, making it available consistently to RPP,
Graceful, and Rotation Shim.

Additional goal-checker choices include:

- Symmetric yaw tolerance, so symmetric robots need not make a needless
  180-degree final rotation.
- `AxisGoalChecker`, with independent along-path and cross-track tolerances and
  optional valid overshoot.
- Adaptive Tolerance Goal Checker, which accepts either fine tolerance or a
  coarse tolerance combined with evidence that the robot stopped, stalled, or
  passed the finish line.

## DWB and Graceful Controller

DWB's `limit_vel_cmd_in_traj` defaults to `false`. When enabled, trajectory
generation is limited using the current robot velocity.

Graceful Controller migrations and additions:

- Replace `motion_target_dist` with `min_lookahead` and `max_lookahead`.
- Rename `final_rotation` to `prefer_final_rotation`.
- Missing orientations in the path are synthesized.
- `v_angular_min_in_place` configures minimum in-place angular velocity.

Retune lookahead and final-rotation behavior together because they influence
where translation yields to orientation alignment.

## Rotation Shim

Rotation Shim can:

- Disengage at `angular_disengage_threshold`.
- Decelerate toward its target with `max_angular_accel`.
- Use orientations from path points with `use_path_orientations`, default
  `false`.

`closed_loop` defaults to `true` and uses odometry. With `closed_loop: false`,
the shim estimates state from its last command; this requires responsive
hardware and realistic acceleration limits.

The primary controller is a nested parameter group. Move both its plugin and
parameters under `primary_controller`:

```yaml
primary_controller:
  plugin: nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController
  desired_linear_vel: 1.0
  lookahead_dist: 0.6
```

## Regulated and Dynamic Window Pure Pursuit

RPP can opt into Dynamic Window Pure Pursuit so velocity and acceleration
constraints participate directly in command generation. In this mode the
maximum-speed parameter is `max_linear_vel`, renamed from
`desired_linear_vel`. Velocity Smoother limits must be at least as permissive
as the controller's corresponding limits, or the downstream smoother changes
the feasible command set.

RPP also enforces `min_distance_to_obstacle` beyond a velocity-scaled carrot,
capped by `max_lookahead_dist`. `allow_obstacle_checking_beyond_goal` defaults
to `false`; when true, checking continues past the goal up to that minimum
distance. The option requires velocity-scaled lookahead and a positive
`min_distance_to_obstacle`.

## SMAC planning

Smac Hybrid and Smac Lattice accept `goal_heading_mode`, allowing one planning
request to consider multiple acceptable goal orientations. This supports
bidirectional or all-direction approaches without separate plans. Smac Lattice
also provides `coarse_search_resolution` for its orientation search.

Multiple SMAC planner instances can coexist and be switched at runtime. Smac
Lattice automatically enables omnidirectional analytic expansion when its
motion-primitives metadata specifies:

```yaml
motion_model: omni
```

Custom `BaseGlobalPlanner` implementations must update `createPath()` to accept
a vector of intermediate `PoseStamped` viapoints.

## Partial multi-pose planning

Planner Server's dynamic `allow_partial_planning` parameter applies to
`compute_path_through_poses`. It defaults to disabled. When enabled, the
server returns the reachable prefix instead of failing the complete request;
the result's `last_reached_index` identifies the final reached goal. Callers
must inspect this field rather than treating action success as evidence that
every requested pose was reached.

## MPPI configuration and observability

### Motion-model plugins and validation

Motion models use named plugin groups, even though `motion_model` defaults to
`diff_drive`:

```yaml
motion_model: diff_drive
diff_drive:
  plugin: mppi::DiffDriveMotionModel
```

This replaces the former `DiffDrive`, `Omni`, and `Ackermann` string values.
`OptimalTrajectoryValidator` separately validates the selected trajectory;
the default validator checks collisions.

### Safety and trajectory output

The cost critic's `near_collision_cost` defaults to `253`, applying critical
cost before actual collision. With `publish_optimal_trajectory: true`, MPPI
publishes poses, velocities, and timestamps as `nav2_msgs/Trajectory` for
visualization or downstream control.

### Initial state, delay, and visualization

`open_loop: true` estimates the initial state from the last command. Model
delay can be compensated independently with `model_delay_vx`,
`model_delay_vy`, and `model_delay_wz`; all default to `0.0`.

When visualization is enabled, `critic_index_to_visualize` chooses total or
per-critic trajectory coloring. Remove `publish_critics_stats`; critic
statistics publish automatically in visualization mode.

## Smoothers

The Savitzky-Golay smoother exposes:

- `window_size`, default `7`
- `poly_order`, default `3`

The Constrained Smoother corrects its objective from effectively quartic
weighted residuals to `weight * residual²`. Existing weights need retuning;
the same numeric values no longer express the same optimization.
