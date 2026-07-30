# MoveIt Servo Control

## Safety scaling and optional checks

Servo scales commanded velocities down near singularities, self-collisions,
and world collisions while enforcing joint position and velocity limits.
Collision checks and command smoothing are independent: configure them with
`check_collisions` and `use_smoothing`.

## IK plugin configuration

Servo can use its built-in inverse-Jacobian path or the planning group's IK
plugin from `kinematics.yaml`. Pass generated
`robot_description_kinematics` parameters into `ServoNode`; they appear as
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

Servo does not accept undeclared parameters from `kinematics.yaml`. An IK
plugin with custom settings must declare those parameters from inside the
plugin.

## Realtime scheduling

When a realtime kernel is available, the main `ServoNode` loop automatically
attempts `SCHED_FIFO` scheduling at priority `40`. No separate scheduler
wrapper is required to obtain this lower-jitter loop.

## Direct C++ interface

Construct `Servo` with a generated parameter listener and a planning-scene
monitor. Select a `CommandType`, repeatedly supply a `JointJogCommand`,
`TwistCommand`, or `PoseCommand` to `getNextJointState()`, and consume the
returned `KinematicState`. That state contains joint names, positions,
velocities, and accelerations.

```cpp
using namespace moveit_servo;

auto node = std::make_shared<rclcpp::Node>("servo_tutorial");
auto listener =
    std::make_shared<const servo::ParamListener>(node, "moveit_servo");
auto params = listener->get_params();
auto monitor = createPlanningSceneMonitor(node, params);
Servo servo(node, listener, monitor);

servo.setCommandType(CommandType::TWIST);
TwistCommand command{"panda_link0", {0.1, 0.0, 0.0, 0.0, 0.0, 0.0}};
KinematicState next = servo.getNextJointState(command);
StatusCode status = servo.getStatus();
```

## ROS commands, output, and status

`ServoNode` accepts:

- `control_msgs::msg::JointJog` on the configured joint-command input topic
- `geometry_msgs::msg::TwistStamped` on the configured Cartesian input topic
- `geometry_msgs::msg::PoseStamped` on the configured pose input topic

Twist and pose inputs require `header.frame_id`. Twist commands currently must
use the robot's planning frame.

`command_out_type` selects
`trajectory_msgs::msg::JointTrajectory` or
`std_msgs::msg::Float64MultiArray` for publication on `command_out_topic`.

Use `/<node_name>/switch_command_type` to select the active command source,
`/<node_name>/pause_servo` to pause, and `/<node_name>/status` for status.

```bash
ros2 service call /servo_node/switch_command_type \
  moveit_msgs/srv/ServoCommandType "{command_type: 1}"
```

## Smoothing plugins

Set `smoothing_filter_plugin_name` when `use_smoothing` is enabled.

- `online_signal_smoothing::ButterworthFilterPlugin` is inexpensive and does
  not overshoot in joint space, but it does not explicitly constrain
  acceleration or jerk.
- `online_signal_smoothing::AccelerationLimitedPlugin` respects acceleration
  limits when feasible and preserves the requested direction when kinematics
  allow, but it may overshoot and does not constrain jerk.
- `online_signal_smoothing::RuckigFilterPlugin` gives the smoothest
  joint-limit- and acceleration-aware output, but it may overshoot or swirl
  around sharp Cartesian corners.
