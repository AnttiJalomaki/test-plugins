# MoveIt Servo

## Safety scaling and optional checks

Servo scales commanded velocities down near singularities, self-collisions,
and world collisions. It also enforces joint position and velocity limits.
Collision checking and command smoothing are independent options, controlled by
`check_collisions` and `use_smoothing`.

## IK plugin configuration

Servo may use its inverse-Jacobian implementation or the planning group's IK
plugin from `kinematics.yaml`. Pass generated `robot_description_kinematics`
parameters to `ServoNode`; the resulting parameter names are
`robot_description_kinematics.<group_name>.<param_name>`.

```python
servo_node = Node(
    package="moveit_servo",
    executable="servo_node",
    parameters=[
        servo_params,
        moveit_config.robot_description,
        moveit_config.robot_description_semantic,
        moveit_config.robot_description_kinematics,
    ],
)
```

Servo does not accept undeclared parameters from `kinematics.yaml`. An IK plugin
with custom parameters must declare those parameters inside the plugin.

## Realtime thread scheduling

When a realtime kernel is available, the main `ServoNode` loop automatically
attempts `SCHED_FIFO` scheduling at priority `40`. No separate scheduling
wrapper is needed to obtain this control-loop jitter reduction.

## Direct C++ interface

Construct `Servo` from a generated parameter listener and a planning-scene
monitor. Choose a `CommandType`, repeatedly give `getNextJointState()` a
`JointJogCommand`, `TwistCommand`, or `PoseCommand`, and consume its
`KinematicState`. The state contains joint names, positions, velocities, and
accelerations.

```cpp
using namespace moveit_servo;

auto node = std::make_shared<rclcpp::Node>("servo_tutorial");
auto listener =
    std::make_shared<const servo::ParamListener>(node, "moveit_servo");
auto params = listener->get_params();
auto monitor = createPlanningSceneMonitor(node, params);
Servo servo(node, listener, monitor);

servo.setCommandType(CommandType::TWIST);
TwistCommand command{
    "panda_link0", {0.1, 0.0, 0.0, 0.0, 0.0, 0.0}};
KinematicState next = servo.getNextJointState(command);
StatusCode status = servo.getStatus();
```

## ROS command and status interfaces

`ServoNode` accepts these messages on its parameterized joint, Cartesian, and
pose input topics:

- `control_msgs::msg::JointJog`
- `geometry_msgs::msg::TwistStamped`
- `geometry_msgs::msg::PoseStamped`

Twist and pose commands require `header.frame_id`. Twist commands currently
must use the robot's planning frame.

`command_out_type` selects either
`trajectory_msgs::msg::JointTrajectory` or
`std_msgs::msg::Float64MultiArray` on `command_out_topic`.

Change the active input with `/<node_name>/switch_command_type`, pause with
`/<node_name>/pause_servo`, and monitor `/<node_name>/status`.

```bash
ros2 service call /servo_node/switch_command_type \
  moveit_msgs/srv/ServoCommandType "{command_type: 1}"
```

## Smoothing plugins

Set `smoothing_filter_plugin_name` when `use_smoothing` is enabled.

### Butterworth

`online_signal_smoothing::ButterworthFilterPlugin` is inexpensive and does not
overshoot in joint space. It does not explicitly constrain acceleration or
jerk.

### Acceleration-limited

`online_signal_smoothing::AccelerationLimitedPlugin` respects acceleration
limits when feasible and preserves the requested direction when kinematics
allow. It may overshoot and does not constrain jerk.

### Ruckig

`online_signal_smoothing::RuckigFilterPlugin` produces the smoothest output
while accounting for joint limits and acceleration. It may overshoot or swirl
around sharp Cartesian corners.
