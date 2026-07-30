---
name: moveit2-knowledge-patch
description: MoveIt 2
version: 2.14.3
license: MIT
metadata:
  author: Nevaberry
---


# MoveIt 2

Use this skill when implementing or reviewing MoveIt 2 planning, trajectory
execution, industrial motion, Task Constructor tasks, or realtime Servo
control. Start with the migration cautions below, then open the reference that
matches the subsystem being changed.

## Reference index

| Reference | Topics |
| --- | --- |
| [planning-pipelines-and-python.md](references/planning-pipelines-and-python.md) | Jazzy integration changes, `moveit_py`, request profiles, constraints, planning-scene access, built-in adapters |
| [ompl-chomp-and-trajectory-processing.md](references/ompl-chomp-and-trajectory-processing.md) | OMPL objectives and persistent roadmaps, CHOMP tuning and initialization, STOMP and trajectory processing |
| [pilz-industrial-motion.md](references/pilz-industrial-motion.md) | Pilz limits, PTP/LIN/CIRC requests, circular paths, blended sequences and interfaces |
| [task-constructor.md](references/task-constructor.md) | MTC stage flow, lifecycle, planners, property forwarding, IK, scene transitions and diagnostics |
| [servo-control.md](references/servo-control.md) | Servo safety, IK plugins, realtime scheduling, direct C++ and ROS interfaces, smoothing |

## Migration-sensitive behavior

### Treat planning-pipeline adapters as a changed API

The planning pipeline now represents request and response adapters more
explicitly. Audit integrations carried forward from Humble instead of assuming
their adapter construction, ordering, or identifiers remain source-compatible.

Do not copy the legacy CHOMP and STOMP adapter tutorials into MoveIt 2. Those
examples use Catkin, `roslaunch`, XML launch files, and old plugin identifiers.
Their useful guidance is architectural: seed CHOMP from a base planner with
`fillTrajectory`, or smooth an existing path with STOMP configured to consume
that trajectory.

### Keep pipeline names separate from request-profile names

`planning_pipelines.pipeline_names` loads pipeline plugins. A
`MultiPipelinePlanRequestParameters` constructor instead receives names of
top-level parameter profiles, and each profile contains `plan_request_params`
that select a pipeline and planner.

```yaml
planning_pipelines:
  pipeline_names: [ompl, chomp]

ompl_fast:
  plan_request_params:
    planning_pipeline: ompl
    planner_id: RRTConnectkConfigDefault
    planning_attempts: 1
    planning_time: 1.0
```

```python
params = MultiPipelinePlanRequestParameters(moveit, ["ompl_fast", "chomp_profile"])
result = arm.plan(multi_plan_parameters=params)
```

### Respect Pilz namespaces and request rules

Load Pilz with
`pilz_industrial_motion_planner/CommandPlanner`. Cartesian limits must resolve
under `<robot_description>_planning.cartesian_limits`; with the normal
description parameter, load them under `robot_description_planning`.

Use `MotionPlanRequest.planner_id` values `PTP`, `LIN`, or `CIRC`. LIN and CIRC
require a zero-velocity start state and interpret request scaling factors as
Cartesian limits. For motion sequences, only the first item may provide a start
state, and adjacent blend spheres must not overlap.

### Forward Task Constructor properties explicitly

Properties do not automatically cross nested MTC containers and wrappers.
Expose task properties to a container, initialize them from `Stage::PARENT`,
and import generated values such as `target_pose` from `Stage::INTERFACE` at
the consuming wrapper.

```cpp
task.properties().exposeTo(
    pick->properties(), { "eef", "group", "ik_frame" });
pick->properties().configureInitFrom(
    mtc::Stage::PARENT, { "eef", "group", "ik_frame" });
ik->properties().configureInitFrom(
    mtc::Stage::INTERFACE, { "target_pose" });
```

## Planning with `moveit_py`

Get a group-specific planning component from `MoveItPy`. The component plans;
the owning `MoveItPy` object executes the returned trajectory.

```python
moveit = MoveItPy(node_name="moveit_py")
arm = moveit.get_planning_component("panda_arm")
arm.set_start_state_to_current_state()
arm.set_goal_state(pose_stamped_msg=goal, pose_link="panda_link8")

result = arm.plan()
if result:
    moveit.execute(result.trajectory, controllers=[])
```

Use `set_start_state(configuration_name=...)` and
`set_goal_state(configuration_name=...)` for named SRDF states. A target
`RobotState` is accepted through `set_goal_state(robot_state=...)`.

For constraint goals, pass a list through
`set_goal_state(motion_plan_constraints=[...])`. Build a joint constraint from
a populated state and its joint model group:

```python
state.joint_positions = {"panda_joint1": -1.0, "panda_joint2": 0.7}
constraint = construct_joint_constraint(
    robot_state=state,
    joint_model_group=moveit.get_robot_model().get_joint_model_group("panda_arm"),
)
arm.set_goal_state(motion_plan_constraints=[constraint])
```

## Planning-scene access

Use the planning-scene monitor's `read_write()` context for collision-object
changes and `read_only()` for queries. After changing state values or solving
IK, call `update()` before transforms or collision checks.

```python
monitor = moveit.get_planning_scene_monitor()
with monitor.read_write() as scene:
    scene.apply_collision_object(collision_object)
    scene.current_state.update()

with monitor.read_only() as scene:
    state = scene.current_state
    state.set_from_ik("panda_arm", pose_goal, "panda_hand")
    state.update()
    colliding = scene.is_state_colliding(
        robot_state=state, joint_model_group_name="panda_arm"
    )
```

## Planner selection

For OMPL, select an optimization objective and, where useful, a termination
condition. `allowed_planning_time` remains the hard time limit.

```yaml
RRTstarkConfigDefault:
  type: geometric::RRTstar
  optimization_objective: MaximizeMinClearanceObjective
  termination_condition: CostConvergence[10,.1]
```

PRM-family planners can retain roadmaps within the node when
`multi_query_planning_enabled` is set. Persist across runs with
`store_planner_data`, `load_planner_data`, and `planner_data_path`; remember
that storing happens when the planner instance is destroyed.

For CHOMP, tune both its termination limits and the smoothness/obstacle
objective. Use `fillTrajectory` when another planner supplies the seed:

```yaml
ridge_factor: 0.001
trajectory_initialization_method: fillTrajectory
```

## Task Constructor essentials

Choose stages according to result flow: generators create states and send them
both ways, propagators extend one neighboring result, connectors bridge two
independently generated states, wrappers modify one child, serial containers
require end-to-end child solutions, and parallel containers select, fall back,
or merge.

Set root properties before adding stages, load the robot model, call `init()`,
request a bounded number of solutions, then explicitly publish or execute a
selected solution. Catch `mtc::InitStageException` around initialization.

Pose generators must monitor the stage that owns the relevant scene state:
`GenerateGraspPose` monitors `CurrentState`, while `GeneratePlacePose` monitors
the saved attach-object stage. Move each generator into `ComputeIK`, configure
the IK count, solution separation, and IK frame, and forward `target_pose` from
the interface.

## Servo essentials

Servo scales velocities near singularities, self-collisions, and world
collisions while enforcing joint position and velocity limits. Toggle collision
checks and smoothing independently with `check_collisions` and `use_smoothing`.

For kinematics-plugin IK, pass generated `robot_description_kinematics`
parameters to `ServoNode`. Custom IK parameters must be declared by the plugin
because Servo does not accept undeclared `kinematics.yaml` parameters.

Twist and pose ROS commands require `header.frame_id`; twist commands use the
robot planning frame. Select the active command input with
`/<node_name>/switch_command_type`, pause with `/<node_name>/pause_servo`, and
watch `/<node_name>/status`.

Choose smoothing deliberately:

- `online_signal_smoothing::ButterworthFilterPlugin` is inexpensive and avoids
  joint-space overshoot, but does not explicitly constrain acceleration or jerk.
- `online_signal_smoothing::AccelerationLimitedPlugin` respects feasible
  acceleration limits and preserves direction when kinematics allow, but may
  overshoot and does not constrain jerk.
- `online_signal_smoothing::RuckigFilterPlugin` produces the smoothest
  joint-limit- and acceleration-aware output, but may overshoot or swirl at
  sharp Cartesian corners.
