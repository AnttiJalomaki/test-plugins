# Migration and Integration

## Distribution support status

- Rolling Ridley is the development distribution.
- Kilted Kaiju and Jazzy Jalisco have active support.
- Humble Hawksbill is maintained.
- Iron Irwini and Galactic Geochelone are end-of-life.

Use the installed distribution and package manifests as the final authority
when a migration below is distribution-dependent.

## Action error propagation

Nav2 action results carry contextual `error_msg` strings through BT Navigator.
The old `error_code_names` parameter causes a startup exception. Replace it
with prefixes:

```yaml
error_code_name_prefixes: [compute_path, follow_path, spin, route]
```

Each relevant BT action node must expose matching `error_code_id` and
`error_msg` ports:

```xml
<FollowPath path="{path}"
            error_code_id="{follow_path_error_code}"
            error_msg="{follow_path_error_msg}"/>
```

Custom actions and navigators should propagate both fields so callers retain
the failure's component-specific context.

## Stamped velocity migration

Nav2 `cmd_vel` publishers and subscribers use `geometry_msgs/TwistStamped` by
default without changing topic names. The timestamp permits stale-command
rejection, but every endpoint in the command chain must agree on the message
type. Audit controllers, smoothers, collision safety, muxes, bridges, hardware
drivers, record/replay tools, and tests.

Only keep the legacy `geometry_msgs/Twist` interface by setting the following
on every affected node:

```yaml
enable_stamped_cmd_vel: false
```

## Multi-pose actions and waypoint statuses

`NavigateThroughPoses`, `ComputePathThroughPoses`, and their BT nodes use
`nav_msgs/Goals` instead of a raw vector of `PoseStamped` values. Access the
fields as follows:

- Navigation action poses: `poses.goals`
- Compute-path goals: `goals.goals`

`NavigateThroughPoses` reports `WaypointStatus` entries with `PENDING`,
`COMPLETED`, `SKIPPED`, or `FAILED`. This replaces `MissedWaypoint`. Any BT
node that prunes or transforms a multi-pose request must preserve the status
list through matching `input_waypoint_statuses` and
`output_waypoint_statuses` ports.

## Namespaced bringup

`use_namespace` was removed from `nav2_bringup`. The `namespace` launch
argument is always applied and defaults to `/`.

Shared parameter and RViz files should use relative topics such as `scan` so
they resolve beneath the robot namespace. An absolute topic such as `/scan`
deliberately remains global. A costmap layer that needs the parent costmap's
namespace instead of its own private layer namespace can use
`joinWithParentNamespace()`.

Test multi-robot launch files for accidental global topics, duplicated node
names, and parameters that were formerly guarded by `use_namespace`.

## Nav2 ROS lifecycle wrappers

`nav2_ros_common` replaces the lifecycle utilities formerly provided by
`nav2_util`. Custom plugins and task servers should use `nav2::LifecycleNode`
and create Nav2 wrappers through the lifecycle node's factories instead of
constructing wrappers directly:

```cpp
main_client_ = node->create_client<SrvT>(service_name, false);
action_client_ = node->create_action_client<ActionT>(action_name, callback_group);
```

The wrapper contract also changes in these ways:

- An omitted QoS profile uses `nav2::qos::StandardTopicQoS`: reliable,
  volatile, depth 10.
- An explicit subscription QoS argument comes after the callback.
- Service callbacks include the `rmw_request_id_t` request header.
- Wrappers use `introspection_mode` and `allow_parameter_qos_overrides`.
- Remove `action_server_result_timeout`; it no longer exists.

Create services, actions, publishers, and subscriptions with the matching
`create_*` factory so lifecycle activation and deactivation remain coherent.

## Lifecycle bonds

`bond_heartbeat_period` defaults to `0.25` seconds for lifecycle nodes and
Lifecycle Manager, increased from `0.1`. An explicit older value still wins.
Remove or update explicit values when the new default is intended, and account
for the chosen heartbeat in failure-detection timing tests.

## Service and behavior-tree introspection

`service_introspection_mode` accepts:

- `disabled` (the default)
- `metadata`
- `contents`

Choose `contents` only when request and response bodies may be exposed safely.
The two standard navigators also provide disabled-by-default Groot 2 live
monitoring, blackboard JSON inspection, and selection of a different BT XML in
a new goal request.

## RViz navigation panel

The Nav2 panel can select BT XML per request, accept exact coordinates and
frame IDs, and build, edit, save, or load multi-goal lists for
`NavigateThroughPoses` and Waypoint Following. Use these controls to reproduce
the same tree, frame, and goal list when investigating action behavior.

## Isolated middleware tests

Build with the following CMake option when `rmw_zenoh_cpp` tests should run
without a separately launched Zenoh router:

```text
--cmake-args -DUSE_ISOLATED_TESTS=ON
```
