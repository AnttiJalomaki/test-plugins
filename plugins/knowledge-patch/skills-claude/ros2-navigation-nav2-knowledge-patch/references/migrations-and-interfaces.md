# Migrations and interfaces

Source batch attribution: `overview-and-distro-migrations`.

## Distribution posture

Rolling Ridley is the development distribution. Kilted Kaiju and Jazzy
Jalisco have active support, Humble Hawksbill is maintained, and Iron Irwini
and Galactic Geochelone are end-of-life.

## Velocity command type

Nav2 publishers and subscribers on the existing `cmd_vel` topics use
`geometry_msgs/TwistStamped` by default. The stamp allows stale commands to be
rejected. Set `enable_stamped_cmd_vel: false` on every affected node only when
retaining the legacy `Twist` interface; changing only part of the command
pipeline creates a type mismatch.

## Action errors

Action results carry contextual `error_msg` strings through BT Navigator.
Remove the old `error_code_names` parameter because it causes a startup
exception. Configure prefixes and expose both matching ports on relevant BT
action nodes:

```yaml
error_code_name_prefixes: [compute_path, follow_path, spin, route]
```

```xml
<FollowPath path="{path}" error_code_id="{follow_path_error_code}"
            error_msg="{follow_path_error_msg}"/>
```

## Multi-pose messages and waypoint status

`NavigateThroughPoses`, `ComputePathThroughPoses`, and their BT nodes use
`nav_msgs/Goals`, not a raw vector of `PoseStamped`. Navigation poses are in
`poses.goals`; compute-path goals are in `goals.goals`.

`NavigateThroughPoses` reports `WaypointStatus` values: `PENDING`,
`COMPLETED`, `SKIPPED`, or `FAILED`. This replaces `MissedWaypoint`. A BT node
that prunes goals must also pass matching `input_waypoint_statuses` and
`output_waypoint_statuses` ports so pose and status lists remain aligned.

## Namespaced bringup

`nav2_bringup` no longer has `use_namespace`. `namespace` is always applied and
defaults to `/`. Shared RViz and parameter files should normally use relative
topics such as `scan`, which resolve under the robot namespace. Keep `/scan`
only when the topic is deliberately global.

Inside a costmap plugin, use `joinWithParentNamespace()` when a resource must
resolve under the costmap's parent rather than the layer's private namespace.

## Lifecycle and ROS wrappers

In Lyrical, `nav2_ros_common` replaces the lifecycle utilities from
`nav2_util`. Custom plugins and task servers use `nav2::LifecycleNode` and its
`create_*` factories for Nav2 services, actions, publishers, and
subscriptions; do not construct the wrappers directly.

```cpp
main_client_ = node->create_client<SrvT>(service_name, false);
action_client_ = node->create_action_client<ActionT>(action_name, callback_group);
```

When QoS is omitted, wrappers use `nav2::qos::StandardTopicQoS`: reliable,
volatile, depth 10. An explicit subscription QoS argument now follows the
callback. Service callbacks include an `rmw_request_id_t` header. Wrapper
configuration uses `introspection_mode` and
`allow_parameter_qos_overrides`. Remove `action_server_result_timeout`; it no
longer exists.

## Lifecycle bonds and immutable frequency

The default `bond_heartbeat_period` is `0.25` seconds for all lifecycle nodes
and Lifecycle Manager, up from `0.1`. An explicit old value still wins, so
remove or update it to adopt the new default.

Controller Server's `control_frequency` is no longer dynamically mutable.
Choose it before startup instead of relying on a parameter update at runtime.
