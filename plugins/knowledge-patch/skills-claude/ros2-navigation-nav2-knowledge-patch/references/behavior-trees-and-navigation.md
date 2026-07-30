# Behavior trees and navigation

Source batch attribution: `overview-and-distro-migrations`,
`behavior-trees`.

## Node migrations and additions

Kilted adds `GetPoseFromPath`, `RemoveInCollisionGoals`, and `IsStopped`, and
extends `GoalUpdater` to goal lists. In Lyrical, action-like nodes that may
return `RUNNING` are renamed:

| Earlier name | Lyrical name |
| --- | --- |
| `IsStopped` | `CheckStopStatus` |
| `IsPathValid` | `ValidatePath` |
| `IsPoseOccupied` | `CheckPoseOccupancy` |

`TruncatePathLocal` renames its `robot_frame` input to `robot_base_frame`. If
omitted, it inherits BT Navigator's `robot_base_frame` parameter.

```xml
<TruncatePathLocal robot_base_frame="base_link" ... />
```

## Path validation semantics

`ValidatePath` and the `IsPathValid` service replace `check_full_path` with the
oppositely worded `stop_at_first_collision`. Old `check_full_path: false`
means new `stop_at_first_collision: true`, which is the default.
`max_lookahead_distance` defaults to `-1.0` for full-path checking; a positive
value limits validation to that forward distance.

## Control-node behavior

`NonblockingSequence` continues ticking later children while an earlier child
is `RUNNING`. `PauseResumeController` supplies pause and resume services and
pairs with `PersistentSequence`; the latter has a bidirectional child-index
port so execution resumes at the paused child.

`RoundRobin` now defaults to `wrap_around="false"` and fails after its last
child. Set `wrap_around="true"` to retain the former cycling behavior.

## Navigator configuration, subtrees, and introspection

Blackboard-ID parameters live under each navigator plugin, not at BT
Navigator's top level. Examples include
`navigate_to_pose.goal_blackboard_id` and
`navigate_through_poses.waypoint_statuses_blackboard_id`.

Reusable subtrees may be loaded from directories in `bt_search_directories`.
A tree selected by ID must have a unique ID; do not reuse the shared
`MainTree`.

`service_introspection_mode` accepts `disabled`, `metadata`, or `contents` and
defaults to `disabled`. The standard navigators also provide disabled-by-
default Groot 2 live monitoring, blackboard JSON inspection, and per-request
BT XML selection.

## Cancellation, logging, and tracking bounds

BT Navigator's `default_cancel_timeout` defaults to `50` ms for action
cancellation. `bt_log_idle_transitions` defaults to `true`; set it false to
suppress idle transition logs. `IsWithinPathTrackingBounds` tests whether the
robot remains inside configured path-tracking bounds.

## Runtime plugin selectors

The default tree can select progress checker, goal checker, path handler,
controller, and planner plugins at runtime. Each selector writes a selected
plugin ID to a blackboard port, listens on its selector topic, and supplies a
default ID:

```xml
<ProgressCheckerSelector selected_progress_checker="{selected_progress_checker}"
  default_progress_checker="progress_checker" topic_name="progress_checker_selector"/>
<GoalCheckerSelector selected_goal_checker="{selected_goal_checker}"
  default_goal_checker="general_goal_checker" topic_name="goal_checker_selector"/>
<PathHandlerSelector selected_path_handler="{selected_path_handler}"
  default_path_handler="PathHandler" topic_name="path_handler_selector"/>
<ControllerSelector selected_controller="{selected_controller}"
  default_controller="FollowPath" topic_name="controller_selector"/>
<PlannerSelector selected_planner="{selected_planner}"
  default_planner="GridBased" topic_name="planner_selector"/>
```

## Near-goal replanning suppression

The default `ComputePathToPose` subtree keeps the current plan only when the
global goal is unchanged, the robot is near it, and the remaining path is
valid. This avoids feasible-planner loops caused by localization drift or
tracking error during final approach. If any check fails, the fallback
computes a new path:

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
  <ComputePathToPose goal="{goal}" path="{path}" planner_id="{selected_planner}"
                     error_code_id="{compute_path_error_code}"
                     error_msg="{compute_path_error_msg}"/>
</Fallback>
```
