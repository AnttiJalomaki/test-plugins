# Pilz Industrial Motion

## Pipeline and motion limits

Load Pilz as `pilz_industrial_motion_planner/CommandPlanner`. Its Cartesian
limits must resolve under `<robot_description>_planning.cartesian_limits`.
When the URDF parameter is `robot_description`, load them in the
`robot_description_planning` namespace.

```yaml
cartesian_limits:
  max_trans_vel: 1
  max_trans_acc: 2.25
  max_trans_dec: -5
  max_rot_vel: 1.57
```

Joint parameters may additionally declare `has_deceleration` and a negative
`max_deceleration`. Parameterized limits must be at least as strict as the URDF
limits. Pilz applies the strictest limits shared by the involved joints.
Rotational acceleration and deceleration are derived from the corresponding
translational ratio and `max_rot_vel`.

## Request semantics

Set `MotionPlanRequest.planner_id` to:

- `PTP` for synchronized trapezoidal joint profiles governed by the slowest
  lead axis.
- `LIN` for synchronized Cartesian translation and quaternion-slerped
  rotation.
- `CIRC` for synchronized circular Cartesian motion and quaternion-slerped
  rotation.

LIN and CIRC require a zero-velocity start state. Their request velocity and
acceleration scaling factors scale Cartesian limits rather than joint limits.

## Circular-path constraints

CIRC takes its defining-point mode from `path_constraints.name`: use `center`
or `interim`. Put the actual point in
`path_constraints.position_constraints[].constraint_region.primitive_poses[]`
`.position`.

A center defines the shorter arc and cannot create a half-circle. An interim
point forces the arc through that point but cannot create a full circle.

```cpp
request.planner_id = "CIRC";
request.path_constraints.name = "interim";
moveit_msgs::msg::PositionConstraint via;
via.constraint_region.primitive_poses.emplace_back();
via.constraint_region.primitive_poses.back().position = via_point;
request.path_constraints.position_constraints.push_back(via);
```

## Blended motion sequences

Each `MotionSequenceItem` holds a normal request in `req` and a
`blend_radius`. A positive radius lets motion continue toward the next goal
without stopping.

Only the first item may specify a start state. Adjacent blend spheres must not
overlap: the two radii must sum to less than the distance between their goals.
A sequence may span multiple planning groups. Planning is atomic for execution:
if any item cannot be planned, no item executes.

## Service and action interfaces

Enable these `move_group` capabilities:

- `pilz_industrial_motion_planner/MoveGroupSequenceService`
- `pilz_industrial_motion_planner/MoveGroupSequenceAction`

The `/plan_sequence_path` service plans a `MotionSequenceRequest` and returns
trajectories without executing. The `/sequence_move_group` action plans and
executes unless `planning_options.plan_only` is set.

Unlike the ordinary MoveGroup action, the sequence action still executes the
planned path when the robot already satisfies the goal. This behavior preserves
circular and similar motions that should not be discarded merely because their
final goal is already met.
