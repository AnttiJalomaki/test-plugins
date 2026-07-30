# Planner Plugins and Adapters

## OMPL optimization and termination

An OMPL planner configuration can select any of these optimization objectives:

- `PathLengthOptimizationObjective` (default)
- `MechanicalWorkOptimizationObjective`
- `MaximizeMinClearanceObjective`
- `StateCostIntegralObjective`
- `MinimaxObjective`

`termination_condition` accepts `Iteration[num]`,
`CostConvergence[solutionsWindow,epsilon]`, or `ExactSolution`.
`allowed_planning_time` remains a hard upper bound.

```yaml
RRTstarkConfigDefault:
  type: geometric::RRTstar
  optimization_objective: MaximizeMinClearanceObjective
  termination_condition: CostConvergence[10,.1]
```

## Persistent OMPL roadmaps

PRM, PRM*, LazyPRM, and LazyPRM* can retain a roadmap between requests with
`multi_query_planning_enabled`. Use `store_planner_data` and
`load_planner_data` with `planner_data_path` to persist that roadmap. Storage
occurs when the planner instance is destroyed.

```yaml
PersistentLazyPRMstar:
  type: geometric::LazyPRMstar
  multi_query_planning_enabled: 1
  store_planner_data: 1
  load_planner_data: 0
  planner_data_path: /tmp/roadmap.graph
```

Lazy variants revalidate nodes and edges after modest planning-scene changes.
Use non-lazy variants only for static scenes. With both load and store set to
`0`, the node still reuses and extends its in-memory roadmap for its lifetime.
A typical persistent workflow builds and stores using a star planner, then
loads with the corresponding non-star planner for faster queries.

## Pilz pipeline and motion limits

Load Pilz using `pilz_industrial_motion_planner/CommandPlanner`. Cartesian
limits must resolve beneath `<robot_description>_planning.cartesian_limits`;
for the conventional `robot_description` URDF parameter, use the
`robot_description_planning` namespace.

```yaml
cartesian_limits:
  max_trans_vel: 1
  max_trans_acc: 2.25
  max_trans_dec: -5
  max_rot_vel: 1.57
```

Joint parameters can also set `has_deceleration` and a negative
`max_deceleration`. Parameterized limits must be at least as strict as the URDF
limits, and Pilz applies the strictest common joint limits. It derives rotational
acceleration and deceleration from the corresponding translational ratio and
`max_rot_vel`.

## Pilz request semantics

Set `MotionPlanRequest.planner_id` to `PTP`, `LIN`, or `CIRC`.

- PTP synchronizes trapezoidal joint profiles around the slowest lead axis.
- LIN and CIRC synchronize Cartesian translation and quaternion-slerped
  rotation, require a zero-velocity start state, and interpret the request's
  scaling factors as Cartesian limits.

### Circular paths

CIRC identifies its defining point through `path_constraints.name`, using
`center` or `interim`. Put the point in
`path_constraints.position_constraints[].constraint_region.primitive_poses[].position`.
A center chooses the shorter arc and cannot produce a half-circle.
An interim point forces the arc through that point but cannot produce a full
circle.

```cpp
request.planner_id = "CIRC";
request.path_constraints.name = "interim";
moveit_msgs::msg::PositionConstraint via;
via.constraint_region.primitive_poses.emplace_back();
via.constraint_region.primitive_poses.back().position = via_point;
request.path_constraints.position_constraints.push_back(via);
```

## Pilz blended sequences

Each `MotionSequenceItem` contains a normal request in `req` and a
`blend_radius`. A positive radius permits motion toward the next goal without
stopping. Only the first item may specify a start state. Adjacent blend spheres
must not overlap: their radii must sum to less than the distance between their
goals. A sequence may span several planning groups. If any item fails to plan,
none of the items execute.

Enable both of these as `move_group` capabilities:

- `pilz_industrial_motion_planner/MoveGroupSequenceService`
- `pilz_industrial_motion_planner/MoveGroupSequenceAction`

The `/plan_sequence_path` service plans a `MotionSequenceRequest` and returns
trajectories without executing. The `/sequence_move_group` action plans and
executes unless `planning_options.plan_only` is set. Unlike the ordinary
MoveGroup action, the sequence action still executes when the robot already
satisfies the goal; this preserves circular and similar motions.

## CHOMP objective and termination

Configure CHOMP in `chomp_planning.yaml`. Bound optimization with
`planning_time_limit`, `max_iterations`, and
`max_iterations_after_collision_free`. Its objective uses
`smoothness_cost_weight`, `obstacle_cost_weight`, plus separate
`smoothness_cost_velocity`, `smoothness_cost_acceleration`, and
`smoothness_cost_jerk` terms. An `obstacle_cost_weight` of `0.0` ignores
obstacles; `1.0` makes them a hard constraint.

## CHOMP numerical controls and recovery

`learning_rate`, `joint_update_limit`, `collision_clearance`, and
`collision_threshold` control updates and collision handling. `ridge_factor`
adds diagonal noise to the quadratic cost matrix; at least `0.001` can help
escape obstacles, at the cost of smoothness. `use_pseudo_inverse` has a separate
`pseudo_inverse_ridge_factor`.

`use_stochastic_descent` updates a random trajectory point rather than every
point. `enable_failure_recovery` retries with varied parameters up to
`max_recovery_attempts`.

## CHOMP trajectory initialization

`trajectory_initialization_method` accepts `quintic-spline`, `linear`, `cubic`,
or `fillTrajectory`. The interpolation modes synthesize a start-to-goal seed.
`fillTrajectory` consumes a trajectory from another planner, such as OMPL, and
can keep CHOMP out of poor local minima.

```yaml
ridge_factor: 0.001
trajectory_initialization_method: fillTrajectory
```

## Legacy optimizer and smoothing adapters

The CHOMP planning-adapter examples that use Catkin, `roslaunch`, XML launch
files, and legacy adapter identifiers are MoveIt 1/Melodic migration material,
not copy-ready MoveIt 2 configuration. In that legacy API, adapter order matters:
`chomp/OptimizerAdapter` invokes the pipeline's base planner (OMPL or STOMP)
before CHOMP. Load both planners' YAML and configure CHOMP with
`fillTrajectory`.

The legacy `stomp_moveit/StompSmoothingAdapter` post-processes an OMPL or CHOMP
path only when STOMP's `initialization_method` is `4` (`FILL_TRAJECTORY`). That
smoothing adapter is explicitly marked as work in progress.
