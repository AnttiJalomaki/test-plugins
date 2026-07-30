# OMPL, CHOMP, and Trajectory Processing

## OMPL objectives and termination

An OMPL planner configuration can select any of these optimization objectives:

- `PathLengthOptimizationObjective`, the default
- `MechanicalWorkOptimizationObjective`
- `MaximizeMinClearanceObjective`
- `StateCostIntegralObjective`
- `MinimaxObjective`

`termination_condition` accepts `Iteration[num]`,
`CostConvergence[solutionsWindow,epsilon]`, or `ExactSolution`.
`allowed_planning_time` remains a hard upper bound regardless of that
condition.

```yaml
RRTstarkConfigDefault:
  type: geometric::RRTstar
  optimization_objective: MaximizeMinClearanceObjective
  termination_condition: CostConvergence[10,.1]
```

## Persistent OMPL roadmaps

PRM, PRM*, LazyPRM, and LazyPRM* can retain a roadmap across requests when
`multi_query_planning_enabled` is enabled. Use `store_planner_data`,
`load_planner_data`, and `planner_data_path` to persist it across process
lifetimes. Planner data is stored when the planner instance is destroyed.

```yaml
PersistentLazyPRMstar:
  type: geometric::LazyPRMstar
  multi_query_planning_enabled: 1
  store_planner_data: 1
  load_planner_data: 0
  planner_data_path: /tmp/roadmap.graph
```

With both load and store set to `0`, the roadmap is still reused and extended
for the lifetime of the node. Lazy variants revalidate nodes and edges and can
therefore tolerate modest planning-scene changes; use non-lazy variants only
for static scenes. A common durable workflow builds and saves with a star
planner, then loads with the corresponding non-star planner for faster queries.

## CHOMP objective and stopping controls

Configure CHOMP in `chomp_planning.yaml`. Bound optimization with
`planning_time_limit`, `max_iterations`, and
`max_iterations_after_collision_free`.

Its objective combines `smoothness_cost_weight`, `obstacle_cost_weight`, and
separate `smoothness_cost_velocity`, `smoothness_cost_acceleration`, and
`smoothness_cost_jerk` terms. An `obstacle_cost_weight` of `0.0` ignores
obstacles; `1.0` makes them a hard constraint.

## CHOMP numerical and recovery controls

- `learning_rate` and `joint_update_limit` bound optimization updates.
- `collision_clearance` and `collision_threshold` control collision handling.
- `ridge_factor` adds diagonal noise to the quadratic cost matrix. A value of
  at least `0.001` can help the path escape obstacles, at some cost to
  smoothness.
- `use_pseudo_inverse` uses its own `pseudo_inverse_ridge_factor`.
- `use_stochastic_descent` updates one random trajectory point rather than all
  points.
- `enable_failure_recovery` retries with varied parameters, bounded by
  `max_recovery_attempts`.

## CHOMP trajectory initialization

`trajectory_initialization_method` accepts `quintic-spline`, `linear`, `cubic`,
or `fillTrajectory`. The interpolation methods synthesize a seed from start to
goal. `fillTrajectory` consumes a trajectory from a planner such as OMPL and
can keep CHOMP away from poor local minima.

```yaml
ridge_factor: 0.001
trajectory_initialization_method: fillTrajectory
```

## Legacy optimizer and smoothing adapters

The available CHOMP planning-adapter examples are MoveIt 1/Melodic migration
material. They use Catkin, `roslaunch`, XML launch files, and legacy adapter
identifiers; do not treat them as direct MoveIt 2 configuration.

In that legacy API, adapter order is significant:
`chomp/OptimizerAdapter` calls the base planner, OMPL or STOMP, before CHOMP.
Both planners' YAML files must be loaded, and CHOMP must use
`fillTrajectory`.

The legacy `stomp_moveit/StompSmoothingAdapter` post-processes an OMPL or CHOMP
path when STOMP uses `initialization_method: 4` (`FILL_TRAJECTORY`). That
smoothing adapter is explicitly marked work in progress.

## Trajectory processing choices

The Jazzy-era trajectory-processing toolset includes TOTG, Ruckig, and
Butterworth filtering. STOMP is also available as a motion planner through its
new implementation; distinguish it from the legacy smoothing adapter above.
