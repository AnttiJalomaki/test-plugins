# MoveIt Task Constructor

## Stage result flow and containers

Stage order is constrained by how results move:

- Generators create states independently and send them both forward and
  backward.
- Propagators extend a neighboring result in one direction.
- Connectors bridge independently produced states on their two interfaces.
- Wrappers modify or filter one child stage.
- Serial containers accept only end-to-end child solutions.
- Parallel containers select alternatives, provide fallbacks, or merge
  independent solutions.

## Task lifecycle and solutions

Set properties on the root before adding stages. Then initialize, request a
bounded number of successful plans, and explicitly choose which solution to
visualize or execute. `init()` may throw `InitStageException`. `plan(5)` stops
after five successful solutions.

```cpp
mtc::Task task;
task.stages()->setName("pick and place");
task.loadRobotModel(node);
task.setProperty("group", arm_group_name);
task.setProperty("eef", hand_group_name);
task.setProperty("ik_frame", hand_frame);

task.init();
if (task.plan(5)) {
  const auto& solution = *task.solutions().front();
  task.introspection().publishSolution(solution);
  auto result = task.execute(solution);
}
```

## Solvers and connector stages

Available solver objects include `PipelinePlanner(node)`,
`JointInterpolationPlanner`, and `CartesianPath`. Stages receive shared solver
instances. `Connect` accepts a `GroupPlannerVector`, allowing different planners
for multiple groups while bridging generated states.

```cpp
auto pipeline = std::make_shared<mtc::solvers::PipelinePlanner>(node);
auto joint = std::make_shared<mtc::solvers::JointInterpolationPlanner>();
auto cartesian = std::make_shared<mtc::solvers::CartesianPath>();
cartesian->setStepSize(0.01);

auto connect = std::make_unique<mtc::stages::Connect>(
    "move to place", mtc::stages::Connect::GroupPlannerVector{
                         {arm_group_name, pipeline},
                         {hand_group_name, joint}});
connect->setTimeout(5.0);
connect->properties().configureInitFrom(mtc::Stage::PARENT);
task.add(std::move(connect));
```

## Property forwarding

Task properties are not automatically inherited through nested stages. Expose
the required task properties to a container and initialize them from
`Stage::PARENT`.

```cpp
auto pick = std::make_unique<mtc::SerialContainer>("pick object");
task.properties().exposeTo(
    pick->properties(), {"eef", "group", "ik_frame"});
pick->properties().configureInitFrom(
    mtc::Stage::PARENT, {"eef", "group", "ik_frame"});
```

For an IK wrapper around a pose generator, initialize parent-level properties
such as `eef` and `group` from `Stage::PARENT`, and import the generated
`target_pose` from `Stage::INTERFACE`.

## Monitored generators and IK wrappers

`GenerateGraspPose` must monitor the earlier `CurrentState` stage so it observes
the object state. Move the generator into `ComputeIK`, then set the IK solution
count, minimum joint-space separation, and IK frame.

```cpp
auto current = std::make_unique<mtc::stages::CurrentState>("current");
auto* current_state_ptr = current.get();
task.add(std::move(current));

auto poses =
    std::make_unique<mtc::stages::GenerateGraspPose>("grasp poses");
poses->setPreGraspPose("open");
poses->setObject("object");
poses->setAngleDelta(M_PI / 12);
poses->setMonitoredStage(current_state_ptr);

auto ik =
    std::make_unique<mtc::stages::ComputeIK>("grasp IK", std::move(poses));
ik->setMaxIKSolutions(8);
ik->setMinSolutionDistance(1.0);
ik->setIKFrame(grasp_frame_transform, hand_frame);
ik->properties().configureInitFrom(mtc::Stage::PARENT, {"eef", "group"});
ik->properties().configureInitFrom(mtc::Stage::INTERFACE, {"target_pose"});
```

For placement, `GeneratePlacePose` monitors the saved attach-object stage so it
knows how the object is attached. Its `setPose()` accepts a stamped target that
may be expressed in the object frame. Calling `ComputeIK::setIKFrame("object")`
makes the object itself the IK frame.

## Relative motion

`MoveRelative` combines a Cartesian solver with minimum and maximum travel
distances. Its stamped direction's frame controls how the direction vector is
interpreted.

```cpp
auto lift = std::make_unique<mtc::stages::MoveRelative>("lift", cartesian);
lift->properties().configureInitFrom(mtc::Stage::PARENT, {"group"});
lift->setMinMaxDistance(0.1, 0.3);
lift->setIKFrame(hand_frame);
geometry_msgs::msg::Vector3Stamped up;
up.header.frame_id = "world";
up.vector.z = 1.0;
lift->setDirection(up);
```

## Planning-scene transitions

Pick/place tasks use `ModifyPlanningScene` stages to allow hand-object
collisions, attach the object, later forbid collisions, and detach it.

```cpp
auto attach =
    std::make_unique<mtc::stages::ModifyPlanningScene>("attach");
attach->attachObject("object", hand_frame);
auto detach =
    std::make_unique<mtc::stages::ModifyPlanningScene>("detach");
detach->detachObject("object", hand_frame);
```

When a `ModifyPlanningScene` stage propagates backward, its operation is
reversed. In particular, allowing collisions in that direction uses
`allowCollisions(..., false)`, not `true`.

## Stage diagnostics

The terminal stage diagram reports, from left to right, solutions propagated
backward, generated locally, and propagated forward. Its arrows indicate
propagation direction. The first stage with zero generation or forwarding is
where composition failed.

Retrieve a stage's visualization identifier with:

```cpp
task.stages()->findChild(name)->introspectionId()
```
