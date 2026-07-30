---
name: moveit2-knowledge-patch
description: MoveIt 2
version: 2.14.3
license: MIT
metadata:
  author: Nevaberry
---


# MoveIt 2 Knowledge Patch

Use this skill when implementing or reviewing MoveIt 2 planning, planner-plugin,
MoveIt Task Constructor, or MoveIt Servo code. Start with the quick reference,
then open the topic file that matches the task. Check the project's MoveIt and
ROS distribution pins before applying version-dependent advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [Core planning](references/core-planning.md) | Jazzy changes, `moveit_py`, profiles, parallel pipelines, constraints, planning-scene access, built-in adapters |
| [Planner plugins](references/planner-plugins.md) | OMPL optimization and roadmaps, Pilz limits and sequences, CHOMP controls, legacy CHOMP/STOMP adapters |
| [Task Constructor](references/task-constructor.md) | Stage flow, lifecycle, solvers, properties, monitored generators, IK, relative motion, scene transitions, diagnostics |
| [Servo](references/servo.md) | Safety scaling, IK configuration, realtime scheduling, direct C++, ROS interfaces, smoothing plugins |

## Migration-sensitive behavior

### Treat planning-pipeline adapters as an API migration

The planning pipeline now models request and response adapters more explicitly.
Do not assume a Humble-era integration remains source- or configuration-compatible;
inspect its adapter API and ordering when moving it to a newer setup.

### Do not copy MoveIt 1 CHOMP adapter examples into MoveIt 2

Examples using Catkin, `roslaunch`, XML launch files,
`chomp/OptimizerAdapter`, or `stomp_moveit/StompSmoothingAdapter` describe the
legacy MoveIt 1/Melodic API. Use them only to understand pipeline composition.
The legacy CHOMP arrangement runs the base planner before CHOMP, loads both
planners' YAML, and requires `fillTrajectory`. The legacy STOMP smoothing
adapter requires initialization method `4` (`FILL_TRAJECTORY`) and is marked
work in progress.

### Use the correct Pilz parameter namespace

Pilz Cartesian limits resolve under
`<robot_description>_planning.cartesian_limits`. With the conventional URDF
parameter, load them under `robot_description_planning`, not an arbitrary
planner namespace.

### Update changed robot states before checking them

After changing planning-scene state or solving IK, call
`scene.current_state.update()` or `robot_state.update()`. Otherwise transforms
and collision queries can observe stale derived state.

### Declare custom Servo IK parameters in the plugin

Pass `robot_description_kinematics` to `ServoNode`. Servo does not accept
undeclared custom parameters from `kinematics.yaml`, so an IK plugin that adds
parameters must declare them itself.

## Core planning quick reference

### Plan and execute with `moveit_py`

Obtain a group-specific component from the `MoveItPy` instance. Planning returns
a result whose `trajectory` is executed by the parent instance.

```python
moveit = MoveItPy(node_name="moveit_py")
arm = moveit.get_planning_component("panda_arm")
arm.set_start_state_to_current_state()
arm.set_goal_state(pose_stamped_msg=goal, pose_link="panda_link8")

result = arm.plan()
if result:
    moveit.execute(result.trajectory, controllers=[])
```

For named SRDF configurations, call
`set_start_state(configuration_name=...)` or
`set_goal_state(configuration_name=...)`. A `RobotState` may instead be passed
as `set_goal_state(robot_state=...)`.

### Select reusable and parallel planning profiles

`planning_pipelines.pipeline_names` chooses loaded pipelines. Separate top-level
profiles contain `plan_request_params` and can select pipeline, planner ID,
attempts, scaling factors, and planning time.

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

For parallel planning, construct `MultiPipelinePlanRequestParameters` with the
`MoveItPy` instance and profile names—not raw plugin names—and pass it via
`multi_plan_parameters`.

```python
params = MultiPipelinePlanRequestParameters(moveit, ["ompl_fast", "chomp_profile"])
result = arm.plan(multi_plan_parameters=params)
```

### Access the planning scene through scoped contexts

Use `read_write()` for scene mutations and `read_only()` for checks.

```python
monitor = moveit.get_planning_scene_monitor()
with monitor.read_write() as scene:
    scene.apply_collision_object(collision_object)
    scene.current_state.update()
```

## Planner-plugin quick reference

### Control OMPL optimization and termination

The default objective is `PathLengthOptimizationObjective`; alternatives include
`MechanicalWorkOptimizationObjective`, `MaximizeMinClearanceObjective`,
`StateCostIntegralObjective`, and `MinimaxObjective`. `termination_condition`
accepts `Iteration[num]`, `CostConvergence[solutionsWindow,epsilon]`, or
`ExactSolution`; `allowed_planning_time` remains a hard cap.

```yaml
RRTstarkConfigDefault:
  type: geometric::RRTstar
  optimization_objective: MaximizeMinClearanceObjective
  termination_condition: CostConvergence[10,.1]
```

### Configure Pilz requests deliberately

Use planner ID `PTP`, `LIN`, or `CIRC`. PTP synchronizes trapezoidal joint
profiles using the slowest lead axis. LIN and CIRC synchronize Cartesian
translation with quaternion-slerped rotation, require a zero-velocity start,
and interpret request scaling factors as Cartesian limits.

For blended sequences, only the first item may specify a start state. Each
positive `blend_radius` permits continuous motion toward the next goal, but
adjacent radii must sum to less than the distance between goals. Planning is
all-or-nothing even when a sequence spans multiple groups.

### Seed CHOMP when local minima are likely

`trajectory_initialization_method` accepts `quintic-spline`, `linear`, `cubic`,
or `fillTrajectory`. The first three interpolate start to goal;
`fillTrajectory` consumes another planner's path and is useful when CHOMP needs
a better seed.

## Task Constructor quick reference

### Match stages to result flow

- Generators independently create states and send them in both directions.
- Propagators extend a neighboring result forward or backward.
- Connectors bridge states produced independently on their two interfaces.
- Wrappers modify or filter one child.
- Serial containers accept end-to-end child solutions.
- Parallel containers select alternatives, provide fallbacks, or merge results.

Set root properties before adding stages. Then call `init()`, plan a bounded
number of successful solutions, select a solution explicitly, and publish or
execute it. `init()` may throw `InitStageException`; `plan(5)` stops after five
successful solutions.

### Forward properties explicitly

Nested stages do not automatically inherit task properties. Expose the needed
task properties to a container, initialize them from `Stage::PARENT`, and have
an IK wrapper import its generated `target_pose` from `Stage::INTERFACE`.

### Monitor the correct scene-producing stage

`GenerateGraspPose` monitors the earlier `CurrentState`. `GeneratePlacePose`
monitors the saved attach-object stage so it sees the attachment. Move each
generator into `ComputeIK`, then configure solution count, joint-space
separation, and IK frame.

## Servo quick reference

### Respect safety and command-frame requirements

Servo scales velocity near singularities and collisions and enforces joint
position and velocity limits. Collision checking and smoothing are independent
(`check_collisions` and `use_smoothing`). Twist and pose messages need
`header.frame_id`; twist commands currently must use the robot planning frame.

### Choose output and runtime controls

`command_out_type` selects `trajectory_msgs::msg::JointTrajectory` or
`std_msgs::msg::Float64MultiArray` on `command_out_topic`. Switch active input
with `/<node_name>/switch_command_type`, pause with
`/<node_name>/pause_servo`, and observe `/<node_name>/status`.

### Choose smoothing by constraint needs

- `online_signal_smoothing::ButterworthFilterPlugin` is inexpensive and avoids
  joint-space overshoot, but does not explicitly constrain acceleration or jerk.
- `online_signal_smoothing::AccelerationLimitedPlugin` respects acceleration
  limits when feasible and preserves direction when kinematics allow, but can
  overshoot and does not constrain jerk.
- `online_signal_smoothing::RuckigFilterPlugin` gives the smoothest joint-limit-
  and acceleration-aware output, but can overshoot or swirl at sharp Cartesian
  corners.

Open the topic references for complete configuration constraints, service and
action semantics, stage examples, and planner-specific edge cases.
