# Planning, control, and smoothing

Source batch attribution: `overview-and-distro-migrations`,
`planners-and-controllers`, `behaviors-and-algorithm-selection`.

## Choose algorithms by geometry and task

NavFn, Smac 2D, and Theta Star plan in grid space. They suit circular
differential or omnidirectional robots, but do not guarantee a drivable path
for non-circular bodies or curvature-constrained Ackermann and legged
platforms. NavFn specifically checks a circular radius derived from the
footprint.

Smac Hybrid-A* uses full-footprint collision checks with Dubins or
Reeds-Shepp motion for arbitrary-shaped Ackermann and legged robots. Smac
Lattice offers kinematically valid control sets for differential,
omnidirectional, and Ackermann robots of any shape.

DWB ordinarily targets differential and omnidirectional robots; Ackermann and
legged use needs a curvature-aware trajectory generator. TEB and MPPI cover
differential, omnidirectional, Ackermann, and legged motion. RPP and Vector
Pursuit do not provide omnidirectional motion. As a task heuristic:

- use DWB or TEB for dynamic-obstacle avoidance;
- use RPP for exact path following;
- use MPPI for model-predictive control;
- use Vector Pursuit for high-speed or resource-constrained tracking.

## Controller Server execution

Controller Server waits up to `costmap_update_timeout` for a fully updated
local costmap before computing control; the default is `0.3` seconds. Planner
Server has the same parameter with a `1.0`-second default.

```yaml
controller_server:
  ros__parameters:
    costmap_update_timeout: 0.3
planner_server:
  ros__parameters:
    costmap_update_timeout: 1.0
```

Set `use_realtime_priority: true` to raise the controller execution thread to
priority `90`. It defaults to false, and the process user first needs an
adequate `rtprio` limit:

```text
robot soft rtprio 99
robot hard rtprio 99
```

`failure_tolerance` is the number of seconds controller-plugin exceptions may
continue before `FollowPath` fails. `0.0` (default) disables tolerance, `-1.0`
permits failures indefinitely, and a positive number sets the timeout.

```yaml
controller_server:
  ros__parameters:
    failure_tolerance: 0.3
```

Controller Server buffers odometry over `odom_duration` when estimating speed.
The window defaults to `0.3` seconds and `odom_topic` defaults to `odom`.

```yaml
controller_server:
  ros__parameters:
    odom_topic: odom
    odom_duration: 0.3
```

`publish_zero_velocity` defaults to true. Disable it only when Controller
Server must not publish the final zero command.

## Centralized path handling and partial plans

Controller Server owns transformed-plan pruning through a configurable
path-handler plugin such as `nav2_controller::FeasiblePathHandler`. Move
path-handling parameters formerly owned by DWB, RPP, Graceful, and MPPI to the
server. Custom interfaces also change:

- `GoalChecker::isGoalReached()` receives the transformed plan.
- `Controller::setPlan()` becomes lightweight `newPathReceived()`.
- `computeVelocityCommands()` receives the processed plan and global goal.

Planner Server's dynamic `allow_partial_planning` parameter lets
`compute_path_through_poses` return the reachable prefix rather than fail the
whole request. It defaults to false. When enabled, read the result's
`last_reached_index` to identify the final reached goal.

Custom `BaseGlobalPlanner` implementations must update `createPath()` to
accept a vector of intermediate `PoseStamped` viapoints. Multiple SMAC planner
instances may coexist and be selected at runtime. Smac Lattice automatically
enables omnidirectional analytic expansion when its primitives metadata has
`motion_model: "omni"`.

## Goal headings and goal checkers

Smac Hybrid and Smac Lattice support multiple acceptable goal orientations in
one request through `goal_heading_mode`. Lattice also provides
`coarse_search_resolution` for the orientation search. These settings support
bidirectional or all-direction arrival without multiple plan requests.

`PositionGoalChecker` ignores orientation. Kilted RPP's `stateful` mode
retains XY completion while yaw aligns. Lyrical removes that RPP parameter and
moves the behavior into goal checkers through `isGoalXYReached()`, making it
consistent across RPP, Graceful, and Rotation Shim.

Symmetric yaw tolerance avoids an unnecessary 180-degree final rotation for a
symmetric robot. `AxisGoalChecker` provides independent along-path and
cross-track tolerances plus optional valid overshoot. Adaptive Tolerance Goal
Checker accepts either a fine tolerance or a coarse tolerance combined with
evidence that the robot is stopped, stalled, or past the finish line.

## Rotation Shim

Rotation Shim can disengage at `angular_disengage_threshold`, decelerate
toward the target using `max_angular_accel`, and use path-point orientations
with `use_path_orientations` (default false).

`closed_loop` defaults to true and uses odometry. False uses the last
commanded velocity, which requires responsive hardware and suitable
acceleration limits.

The primary controller is now a nested parameter group. Move its plugin and
parameters beneath `primary_controller`:

```yaml
primary_controller:
  plugin: "nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController"
  desired_linear_vel: 1.0
  lookahead_dist: 0.6
```

## DWB, Graceful, and RPP

DWB's `limit_vel_cmd_in_traj` defaults to false. When enabled it limits
trajectory generation using current robot velocity.

Graceful Controller replaces `motion_target_dist` with `min_lookahead` and
`max_lookahead`, renames `final_rotation` to `prefer_final_rotation`,
synthesizes missing path orientations, and adds
`v_angular_min_in_place`.

RPP can use Dynamic Window Pure Pursuit so velocity and acceleration
constraints participate directly in command generation. In that mode, its
maximum-speed parameter is `max_linear_vel`, renamed from
`desired_linear_vel`. Velocity Smoother limits must be at least as permissive
as the corresponding controller limits.

RPP also enforces `min_distance_to_obstacle` beyond a velocity-scaled carrot,
capped by `max_lookahead_dist`. `allow_obstacle_checking_beyond_goal`
defaults to false; enabling it checks past the goal up to the minimum
distance, but requires velocity-scaled lookahead and a positive minimum.

## MPPI

The cost critic's `near_collision_cost` defaults to `253` and applies critical
cost before actual collision. `publish_optimal_trajectory` publishes poses,
velocities, and timestamps as `nav2_msgs/Trajectory`.

Motion models require a named plugin group even though `motion_model` defaults
to `diff_drive`. Plugin groups replace the old `DiffDrive`, `Omni`, and
`Ackermann` string values:

```yaml
motion_model: "diff_drive"
diff_drive:
  plugin: "mppi::DiffDriveMotionModel"
```

`OptimalTrajectoryValidator` separately validates the chosen trajectory; its
default validator checks collisions. `open_loop: true` estimates initial state
from the last command. Compensate per-axis delay with `model_delay_vx`,
`model_delay_vy`, and `model_delay_wz`, all defaulting to `0.0`.

With visualization enabled, `critic_index_to_visualize` selects total or
per-critic trajectory coloring. Remove `publish_critics_stats`; critic
statistics publish automatically in visualization mode.

## Smoothers

Savitzky-Golay Smoother exposes `window_size` (default `7`) and `poly_order`
(default `3`) instead of fixed values.

Constrained Smoother now uses `weight * residual²`, correcting the effectively
quartic weighted residual formulation. Retune existing weights because the
same numeric values no longer represent the same optimization.
