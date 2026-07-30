# Manipulation and teleoperation

## Use current naming without losing package identity

Isaac Manipulator is now named Isaac for Manipulation in 4.0.0. Its reference
workflows still use the `isaac_manipulator` package name.

Those reference workflows include:

- Gear assembly using a contact-rich insertion policy.
- Multi-object pick-and-place orchestrated by behavior trees.

`isaac_manipulator` adds a sim-to-real gear-assembly reach-policy tutorial for
UR10e in 4.1.0.

## Account for forked ROS 2 control dependencies

The 4.0.0 package set bundles `topic_based_ros2_control` from a forked pull
request. Its `ur` package is also forked from Universal Robots ROS 2 Driver and
Universal Robots Client Library rather than consuming the upstream packages
directly.

When debugging behavior or API differences, inspect the bundled forks instead
of assuming upstream package behavior applies unchanged.

## Update cuMotion planning

`isaac_ros_cumotion` updates to cuMotion 1.1.0 in 4.5.0. It also:

- Improves self-consistent ESDF planning.
- Adds AABB clearing for drop-pose planning.
- Changes the controller's `PoseArray` hand order to match Isaac ROS Teleop.

Treat the hand-order change as an interface migration. Check every producer,
consumer, recording, and test that accesses poses by array position.

## Build Docker-free XR teleoperation

`isaac_ros_teleop` can run CloudXR without Docker in 4.5.0. It adds:

- Meta Quest 3 support.
- Raw controller-data publication.
- Configurable XR pose transforms.
- RViz visualization.
- A revised `PoseArray` hand order aligned with the cuMotion controller.

Virtual Environment or Bare Metal selection still has to satisfy the current
platform and dependency matrix.

## Integrate additional manipulators

`isaac_ros_manipulation` adds Flexiv Rizon support and a Bring Your Own Robot
integration guide in 4.5.0. That release also updates the Flexiv, Universal
Robots, static-planning-scene, and cloud pick-and-place workflows.

DOPE can fail to detect objects in manipulation workflows. On Jetson AGX Thor,
the 4.1.0 DOPE quickstart also cannot convert its ONNX model into a TensorRT
Plan because the model contains unsupported layers. Separate runtime detection
failure from model-conversion failure during triage.

## Integrate Unitree G1 workflows

`isaac_ros_physical_ai` adds the following Unitree G1 capabilities in 4.5.0:

- Data-recording workflows.
- GR00T deployment workflows.
- G1 teleoperation for simulation and physical hardware.
- Firmware 1.5.1 acknowledgement handling.

`isaac_ros_robots` changes the accompanying integration surface:

- G1 bridge and bringup defaults.
- Topic and frame names.
- Controller configuration.
- GR00T launch behavior.
- Acknowledgement checks.

Apply these as one coordinated migration; retaining only some old defaults can
produce mismatches across bridge, launch, controller, and policy components.

On physical hardware, Unitree G1 hands can lower after several minutes of
teleoperation because of motor temperature limits. Check thermal conditions
before treating the lowering as a command, acknowledgement, or controller
regression.

