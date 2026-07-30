# Core Planning and Pipelines

## ROS 2 Jazzy planning changes

Source batch: `jazzy-release`.

MoveIt 2 2.10 targets ROS 2 Jazzy Jalisco LTS and Rolling Ridley, replacing
Humble as the recommended MoveIt release. On Jazzy, install ROS Debian binaries
on Ubuntu Noble 24.04 or build from source.

Relative to Humble, the Jazzy release adds:

- execution of multi-DOF trajectories;
- parallel planning pipelines;
- a new STOMP motion-planner implementation; and
- trajectory-processing updates for TOTG, Ruckig, and Butterworth filtering.

The planning pipeline and its API were refactored to represent request and
response adapters more clearly. Treat existing Humble-era pipeline integrations
as migration-sensitive.

## Planning and execution with `moveit_py`

`MoveItPy.get_planning_component()` returns the planning API for one group. The
plan result carries the trajectory; the `MoveItPy` instance executes it.

```python
moveit = MoveItPy(node_name="moveit_py")
arm = moveit.get_planning_component("panda_arm")
arm.set_start_state_to_current_state()
arm.set_goal_state(pose_stamped_msg=goal, pose_link="panda_link8")

result = arm.plan()
if result:
    moveit.execute(result.trajectory, controllers=[])
```

Named SRDF states use `set_start_state(configuration_name=...)` or
`set_goal_state(configuration_name=...)`. To use a `RobotState` as the goal,
call `set_goal_state(robot_state=...)`.

## Named planning-parameter profiles

`planning_pipelines.pipeline_names` selects the pipelines loaded in a
`moveit_py` node. Separate top-level profiles map reusable names to
`plan_request_params`. A profile can select the pipeline, planner ID, attempt
count, velocity and acceleration scaling, and planning time.

```yaml
planning_pipelines:
  pipeline_names: [ompl, chomp]

ompl_fast:
  plan_request_params:
    planning_pipeline: ompl
    planner_id: RRTConnectkConfigDefault
    planning_attempts: 1
    planning_time: 1.0

chomp_profile:
  plan_request_params:
    planning_pipeline: chomp
    planning_time: 1.5
```

For parallel planning, `MultiPipelinePlanRequestParameters` receives the
`MoveItPy` instance and profile names, not just planner-plugin names. Supply it
through the dedicated `multi_plan_parameters` argument.

```python
params = MultiPipelinePlanRequestParameters(
    moveit, ["ompl_fast", "chomp_profile"]
)
result = arm.plan(multi_plan_parameters=params)
```

## Constraint goals

Pass constraint messages as a list through
`set_goal_state(motion_plan_constraints=...)`. The joint-constraint helper
builds a constraint from a populated `RobotState` and its target joint model
group.

```python
state.joint_positions = {"panda_joint1": -1.0, "panda_joint2": 0.7}
constraint = construct_joint_constraint(
    robot_state=state,
    joint_model_group=moveit.get_robot_model().get_joint_model_group("panda_arm"),
)
arm.set_goal_state(motion_plan_constraints=[constraint])
```

## Planning-scene monitor contexts

Planning-scene access is scoped. Use `read_write()` for collision-object or
state changes and `read_only()` for checks. After changing scene state or
solving IK, update the state so transforms and collision checks see the new
values.

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

Calling `robot_state.update()` is equivalent when working with a standalone
state object.

## Built-in planning adapters

- `CheckStartStateBounds` can clamp a slightly out-of-bounds start joint to its
  URDF limit, subject to its configured tolerance.
- `ValidateWorkspaceBounds` supplies a 10 m x 10 m x 10 m workspace only when
  the request does not specify one.
- `CheckStartStateCollision` samples nearby states according to
  `jiggle_fraction` and a retry limit.
- `ResolveConstraintFrames` rewrites constraints expressed in object subframes,
  such as `cup/handle`, into object or robot frames.
