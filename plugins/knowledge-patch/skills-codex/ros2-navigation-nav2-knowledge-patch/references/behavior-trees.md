# Behavior Trees

## Node additions and action-style renames

Kilted adds these BT nodes:

- `GetPoseFromPath`
- `RemoveInCollisionGoals`
- `IsStopped`

`GoalUpdater` also supports lists of goals. In Lyrical, nodes that can return
`RUNNING` use action-style names:

| Earlier name | Lyrical name |
| --- | --- |
| `IsStopped` | `CheckStopStatus` |
| `IsPathValid` | `ValidatePath` |
| `IsPoseOccupied` | `CheckPoseOccupancy` |

Do not place these in a tree under assumptions that apply only to instantaneous
conditions; their `RUNNING` state affects parent control-node behavior.

## Control-node semantics

`NonblockingSequence` continues ticking later children while an earlier child
is `RUNNING`.

`PauseResumeController` exposes pause and resume services. Pair it with
`PersistentSequence`, whose bidirectional child-index port retains the point
from which execution should resume.

`RoundRobin` now defaults to `wrap_around="false"`. It returns failure after
its final child rather than returning to the first. Set
`wrap_around="true"` explicitly when cyclic legacy behavior is required.

## Navigator blackboards and reusable subtrees

Blackboard-ID parameters belong under each navigator plugin rather than at the
top level of BT Navigator. Examples include:

```yaml
navigate_to_pose:
  goal_blackboard_id: goal
navigate_through_poses:
  waypoint_statuses_blackboard_id: waypoint_statuses
```

BT XML can load reusable subtrees from directories configured in
`bt_search_directories`. Select a tree with a unique tree ID; do not reuse the
shared `MainTree` identifier for multiple selectable trees.

## Runtime navigation-plugin selectors

The default tree has selectors for progress checker, goal checker, path
handler, controller, and planner. Each selector:

1. Subscribes to its named selector topic.
2. Writes the selected plugin ID to a blackboard port.
3. Uses its configured default ID before an override arrives.

```xml
<ProgressCheckerSelector selected_progress_checker="{selected_progress_checker}"
  default_progress_checker="progress_checker"
  topic_name="progress_checker_selector"/>
<GoalCheckerSelector selected_goal_checker="{selected_goal_checker}"
  default_goal_checker="general_goal_checker"
  topic_name="goal_checker_selector"/>
<PathHandlerSelector selected_path_handler="{selected_path_handler}"
  default_path_handler="PathHandler"
  topic_name="path_handler_selector"/>
<ControllerSelector selected_controller="{selected_controller}"
  default_controller="FollowPath"
  topic_name="controller_selector"/>
<PlannerSelector selected_planner="{selected_planner}"
  default_planner="GridBased"
  topic_name="planner_selector"/>
```

The default IDs must name loaded plugins. A topic update changes behavior at
runtime without replacing the tree.

## Near-goal replanning suppression

The default `ComputePathToPose` subtree keeps the current plan when all three
gates hold: the global goal has not changed, the robot is near the goal, and
the remaining path is valid. This prevents localization drift or final
tracking error from causing repeated feasible-planner loops. If any gate
fails, the fallback computes a new path.

```xml
<Fallback name="FallbackComputePathToPose">
  <ReactiveSequence name="CheckIfNewPathNeeded">
    <Inverter><GlobalUpdatedGoal/></Inverter>
    <IsGoalNearby path="{path}" proximity_threshold="4.0"
                  max_robot_pose_search_dist="1.5"/>
    <TruncatePathLocal input_path="{path}" output_path="{remaining_path}"
                       distance_forward="-1" distance_backward="0.0"/>
    <ValidatePath path="{remaining_path}"/>
  </ReactiveSequence>
  <ComputePathToPose goal="{goal}" path="{path}"
                     planner_id="{selected_planner}"
                     error_code_id="{compute_path_error_code}"
                     error_msg="{compute_path_error_msg}"/>
</Fallback>
```

Tune the proximity and robot-pose search distances for the map scale and
controller behavior rather than removing the validity gate.

## Path validation migration

The `ValidatePath` BT node and `IsPathValid` service replace
`check_full_path` with the oppositely worded `stop_at_first_collision`.
Earlier `check_full_path: false` is equivalent to
`stop_at_first_collision: true`, which remains the default.

`max_lookahead_distance` defaults to `-1.0`, meaning full-path validation. A
positive value restricts collision checking to that forward distance.

## TruncatePathLocal port migration

`TruncatePathLocal` renames its `robot_frame` input to `robot_base_frame`. If
the port is absent, it inherits BT Navigator's `robot_base_frame` parameter.

```xml
<TruncatePathLocal robot_base_frame="base_link" ... />
```

## Cancellation, logging, and tracking bounds

BT Navigator adds:

- `default_cancel_timeout`, default `50` ms, for action cancellation.
- `bt_log_idle_transitions`, default `true`; set it false to suppress idle
  transition noise.
- `IsWithinPathTrackingBounds`, which tests whether the robot is still inside
  configured path-tracking bounds.

Choose a cancel timeout that permits the underlying action to acknowledge
cancellation without making tree replacement unresponsive.

## Multi-pose pruning ports

Trees using `NavigateThroughPoses` or `ComputePathThroughPoses` receive
`nav_msgs/Goals` containers. Nodes that remove goals, including collision
pruning, must keep waypoint statuses aligned with the retained poses by using
the paired `input_waypoint_statuses` and `output_waypoint_statuses` ports.

## Live tree inspection

The standard navigators can enable Groot 2 live monitoring, inspect the
blackboard as JSON, and accept a new BT XML with a goal request. These features
are disabled by default. When enabled, combine them with action `error_code_id`
and `error_msg` ports to distinguish tree-structure failures from server or
plugin failures.
