# Manipulation, teleoperation, and robots

## Use Isaac for Manipulation workflows

Isaac Manipulator is now named Isaac for Manipulation. Its
`isaac_manipulator` reference workflows in 4.0.0 include:

- gear assembly using a contact-rich insertion policy; and
- multi-object pick-and-place orchestrated by behavior trees.

In 4.1.0, `isaac_manipulator` also adds a sim-to-real gear-assembly
reach-policy tutorial for UR10e.

The bundled `topic_based_ros2_control` package is based on a forked pull
request. The bundled `ur` package is forked from Universal Robots ROS 2 Driver
and Universal Robots Client Library instead of using those upstream packages
directly (4.0.0). Account for those forks when comparing behavior or
dependencies with upstream.

## Plan with cuMotion

`isaac_ros_cumotion` updates to cuMotion 1.1.0 in 4.5.0. It improves
self-consistent ESDF planning and adds AABB clearing for drop-pose planning.

The controller changes the order of hands in its `PoseArray` to match Isaac
ROS Teleop. Audit consumers that map array indices to left and right hands.

## Integrate a robot

`isaac_ros_manipulation` adds Flexiv Rizon support and a Bring Your Own Robot
integration guide in 4.5.0. The release also updates:

- Flexiv workflows;
- Universal Robots workflows;
- static-planning-scene workflows; and
- cloud pick-and-place workflows.

## Configure XR teleoperation

`isaac_ros_teleop` can run CloudXR without Docker as of 4.5.0. It adds:

- Meta Quest 3 support;
- raw controller-data publication;
- configurable XR pose transforms; and
- RViz visualization.

Its `PoseArray` hand order changes alongside the cuMotion controller. Update
publishers and consumers together.

## Deploy Unitree G1 workflows

In 4.5.0, `isaac_ros_physical_ai` adds:

- Unitree G1 data-recording workflows;
- GR00T deployment workflows; and
- G1 teleoperation for simulation and real hardware.

Hardware operation includes firmware 1.5.1 acknowledgement handling.

`isaac_ros_robots` changes the G1 bridge and bringup defaults, topic and frame
names, controller configuration, GR00T launch behavior, and acknowledgement
checks. Do not carry an older G1 configuration forward without reconciling all
of those surfaces.

During real-hardware teleoperation, the G1 hands can lower after several
minutes because of motor temperature limits.

## Diagnose manipulation simulation

NITROS Bridge topics from Isaac Sim 5.1 may not arrive through DDS, breaking
the object-following manipulation simulation tutorial (4.0.0). Confirm bridge
delivery before debugging the behavior workflow.

DOPE can fail to detect objects in manipulation workflows (4.0.0). Separate a
perception failure from planning or behavior-tree orchestration.
