# Planning Pipelines and Python

## Jazzy integration changes

The included `jazzy-release` guidance identifies MoveIt 2 version 2.10 as the
release for ROS 2 Jazzy Jalisco LTS and Rolling Ridley, replacing Humble as the
recommended MoveIt release. On Jazzy, install ROS Debian binaries on Ubuntu
Noble 24.04 or build from source.

Compared with Humble, this release adds multi-DOF trajectory execution and
parallel planning pipelines. It also refactors the planning pipeline and its
API so request and response adapters are represented more clearly. Treat
Humble-era pipeline integrations as migration-sensitive.

The release also supplies a new STOMP motion-planner implementation. Its
trajectory-processing changes cover Time-Optimal Trajectory Generation (TOTG),
Ruckig, and Butterworth filtering.

## `moveit_py` planning and execution

`MoveItPy.get_planning_component()` returns the planning API for a named group.
Planning returns a result containing the trajectory; execution belongs to the
`MoveItPy` instance.

```python
moveit = MoveItPy(node_name="moveit_py")
arm = moveit.get_planning_component("panda_arm")
arm.set_start_state_to_current_state()
arm.set_goal_state(pose_stamped_msg=goal, pose_link="panda_link8")

result = arm.plan()
if result:
    moveit.execute(result.trajectory, controllers=[])
```

Named SRDF configurations are accepted by
`set_start_state(configuration_name=...)` and
`set_goal_state(configuration_name=...)`. Supply a `RobotState` target with
`set_goal_state(robot_state=...)`.

## Request profiles and parallel planning

Use `planning_pipelines.pipeline_names` to select which pipelines load into the
node. Define reusable top-level profiles separately; each profile's
`plan_request_params` can select the pipeline, planner ID, attempt count,
velocity and acceleration scaling factors, and planning time.

```yaml
planning_pipelines:
  pipeline_names: [ompl, chomp]

ompl_fast:
  plan_request_params:
    planning_pipeline: ompl
    planner_id: RRTConnectkConfigDefault
    planning_attempts: 1
    max_velocity_scaling_factor: 0.8
    max_acceleration_scaling_factor: 0.8
    planning_time: 1.0

chomp_profile:
  plan_request_params:
    planning_pipeline: chomp
    planning_time: 1.5
```

`MultiPipelinePlanRequestParameters` takes the `MoveItPy` object and parameter
profile names, not raw planner-plugin names. Pass the result using the
dedicated `multi_plan_parameters` argument.

```python
params = MultiPipelinePlanRequestParameters(
    moveit, ["ompl_fast", "chomp_profile"]
)
result = arm.plan(multi_plan_parameters=params)
```

## Constraint goals

Pass constraint messages as a list through
`set_goal_state(motion_plan_constraints=...)`. The joint-constraint helper
constructs one from a populated `RobotState` and the target joint model group.

```python
state.joint_positions = {"panda_joint1": -1.0, "panda_joint2": 0.7}
constraint = construct_joint_constraint(
    robot_state=state,
    joint_model_group=moveit.get_robot_model().get_joint_model_group("panda_arm"),
)
arm.set_goal_state(motion_plan_constraints=[constraint])
```

## Planning-scene monitor contexts

Scope collision-object modifications with `read_write()` and checks with
`read_only()`. After changing the scene state or solving IK, call
`scene.current_state.update()` or `robot_state.update()` so later transforms
and collision tests see the new values.

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

## Built-in request adapters

- `CheckStartStateBounds` may clamp a slightly out-of-range start joint to its
  URDF limit when the difference falls inside the configured tolerance.
- `ValidateWorkspaceBounds` inserts a 10 m × 10 m × 10 m workspace only when
  the request does not provide one.
- `CheckStartStateCollision` samples nearby states according to
  `jiggle_fraction` and its retry limit.
- `ResolveConstraintFrames` rewrites constraints expressed in object subframes,
  such as `cup/handle`, into object or robot frames.
