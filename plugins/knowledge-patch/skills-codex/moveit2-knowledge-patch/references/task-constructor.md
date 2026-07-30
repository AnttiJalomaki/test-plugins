# MoveIt Task Constructor

## Stage flow and containers

Choose a stage by how results move:

- Generators create states independently and send them in both directions.
- Propagators extend a neighboring result either forward or backward.
- Connectors bridge independently generated states on their two interfaces.
- Wrappers modify or filter one child stage.
- Serial containers accept only end-to-end child solutions.
- Parallel containers select alternatives, provide fallbacks, or merge
  independent solutions.

## Task lifecycle and solutions

Set root properties before adding stages. Then initialize, request a bounded
number of successful plans, and explicitly choose a solution for introspection
or execution. `init()` can throw `mtc::InitStageException`; `plan(5)` stops
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

## Planner objects and connectors

MTC provides `PipelinePlanner(node)`, `JointInterpolationPlanner`, and
`CartesianPath`; stages share planner objects. `Connect` takes a
`GroupPlannerVector`, so different planners can bridge generated states for
different groups.

```cpp
auto pipeline = std::make_shared<mtc::solvers::PipelinePlanner>(node);
auto joint = std::make_shared<mtc::solvers::JointInterpolationPlanner>();
auto cartesian = std::make_shared<mtc::solvers::CartesianPath>();
cartesian->setStepSize(0.01);

auto connect = std::make_unique<mtc::stages::Connect>(
    "move to place", mtc::stages::Connect::GroupPlannerVector{
                         { arm_group_name, pipeline },
                         { hand_group_name, joint } });
connect->setTimeout(5.0);
connect->properties().configureInitFrom(mtc::Stage::PARENT);
task.add(std::move(connect));
```

## Property forwarding

Task properties are not automatically inherited through nested stages. Expose
selected properties to the container, initialize them from `Stage::PARENT`,
and import generated interface properties at the wrapper that consumes them.

```cpp
auto pick = std::make_unique<mtc::SerialContainer>("pick object");
task.properties().exposeTo(pick->properties(), { "eef", "group", "ik_frame" });
pick->properties().configureInitFrom(
    mtc::Stage::PARENT, { "eef", "group", "ik_frame" });
```

An IK wrapper commonly gets its task properties from the parent and its
generated pose from the interface:

```cpp
ik->properties().configureInitFrom(mtc::Stage::PARENT, { "eef", "group" });
ik->properties().configureInitFrom(mtc::Stage::INTERFACE, { "target_pose" });
```

## Monitored generators and IK

`GenerateGraspPose` must monitor the earlier `CurrentState` stage so it sees
the object's current scene state. Save the stage pointer before moving it into
the task.

```cpp
auto current = std::make_unique<mtc::stages::CurrentState>("current");
auto* current_state_ptr = current.get();
task.add(std::move(current));

auto poses = std::make_unique<mtc::stages::GenerateGraspPose>("grasp poses");
poses->setPreGraspPose("open");
poses->setObject("object");
poses->setAngleDelta(M_PI / 12);
poses->setMonitoredStage(current_state_ptr);

auto ik = std::make_unique<mtc::stages::ComputeIK>("grasp IK", std::move(poses));
ik->setMaxIKSolutions(8);
ik->setMinSolutionDistance(1.0);
ik->setIKFrame(grasp_frame_transform, hand_frame);
```

`GeneratePlacePose` instead monitors the saved attach-object stage so it knows
how the object is attached. Its `setPose()` accepts a stamped target expressed
in the object frame, and `ComputeIK::setIKFrame("object")` can make that object
the IK frame.

## Relative motion and scene transitions

`MoveRelative` combines a Cartesian planner, minimum and maximum travel
distances, and a stamped direction. The direction's frame determines how its
vector is interpreted.

```cpp
auto lift = std::make_unique<mtc::stages::MoveRelative>("lift", cartesian);
lift->properties().configureInitFrom(mtc::Stage::PARENT, { "group" });
lift->setMinMaxDistance(0.1, 0.3);
lift->setIKFrame(hand_frame);
geometry_msgs::msg::Vector3Stamped up;
up.header.frame_id = "world";
up.vector.z = 1.0;
lift->setDirection(up);
```

Build pick/place scene changes with `ModifyPlanningScene`: allow hand-object
collision, attach the object, later forbid collision, and detach it.

```cpp
auto attach = std::make_unique<mtc::stages::ModifyPlanningScene>("attach");
attach->attachObject("object", hand_frame);
auto detach = std::make_unique<mtc::stages::ModifyPlanningScene>("detach");
detach->detachObject("object", hand_frame);
```

When a `ModifyPlanningScene` stage propagates backward, its operation reverses.
In particular, allowing collisions in that direction uses
`allowCollisions(..., false)`, not `true`.

## Stage diagnostics

The terminal stage diagram reports backward-propagated, locally generated, and
forward-propagated solutions from left to right. Its arrows show propagation
direction. The first stage with zero generation or forwarding is the likely
composition failure.

Get the visualization identifier for a named stage through:

```cpp
task.stages()->findChild(name)->introspectionId()
```
