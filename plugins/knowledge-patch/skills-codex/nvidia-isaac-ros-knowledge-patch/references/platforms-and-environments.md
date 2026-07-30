# Platforms and environments

## Choose a tested current target

Current quickstarts target ROS 2 Jazzy through the Isaac ROS CLI-managed
environment. Use one of the tested configurations below.

| Target | Tested configuration |
| --- | --- |
| Jetson | Thor T5000 or T4000, JetPack 7.1, at least 128 GB NVMe storage |
| x86_64 | Ampere-or-newer NVIDIA GPU with at least 8 GB RAM, Ubuntu 24.04, CUDA 13.0 or newer, driver 580 or newer, at least 32 GB storage |
| DGX Spark | DGX OS 7.2.3, at least 32 GB storage |

Other GB10 systems are outside the test matrix. Docker-optional Virtual
Environment or Bare Metal use must still follow this dependency and platform
matrix.

## Match the package set to JetPack and Jetson

The `v3.2-1` package set adds JetPack 6.2 and Jetson Orin Nano Super support
(3.2-1). Use that update rather than base `v3.2` for either platform.

Isaac ROS 4.0 adds Jetson AGX Thor and a JetPack 7.0 stack based on Ubuntu 24.04
and CUDA 13.0 (4.0.0). It was tested with Isaac Sim 5.1.

In the 4.0 package set, platform support did not imply that all sensor stacks
were mature:

- Isaac Perceptor and Nova packages were not yet optimized for AGX Thor.
- The ZED SDK was incompatible with Jetson Thor, so ZED cameras were not
  tested.
- RealSense SDK support on JetPack 7 could become unstable and stop publishing
  images.

Isaac ROS 4.1 adds `sensor_mounting_rig` support for the Jetson AGX Thor
RealSense Rig (4.1.0). For RealSense on JetPack 7, use the dedicated 4.1 setup
procedure, which addresses the platform's SDK stability issue.

## Select Docker, Virtual Environment, or Bare Metal

Virtual Environment and Bare Metal became supported development and deployment
modes in 4.1.0, so Docker is not required for those workflows. This choice
changes the environment setup, not the tested hardware and dependency matrix.

SAM2 visualization in the 4.1 Virtual Environment flow could fail because of a
NumPy mismatch; NumPy 1.26.4 was a possible workaround. The SAM2 quickstart
dependency problems affecting both Virtual Environment and Bare Metal flows
are fixed in 4.5.0.

CloudXR teleoperation can also run without Docker in 4.5.0. See the
manipulation and teleoperation reference for its XR capabilities.

## Interpret DGX Spark support by package set

DGX Spark was explicitly unsupported in 4.1.0. DGX Spark and JetPack 7.1
support arrived on 2026-02-19, so that older exclusion does not describe the
current tested environment. Current DGX Spark quickstarts use DGX OS 7.2.3 and
require at least 32 GB storage.

Do not generalize DGX Spark support to every GB10 system; other GB10 systems
remain outside the current test matrix.

## Account for current SIPL camera support

Early SIPL and Leopard Imaging Eagle stereo Camera-over-Ethernet support
followed on 2026-03-23. `isaac_ros_sipl_camera` publishes SIPL camera images
through zero-copy NITROS. Match an integration to this early support boundary
and the current JetPack 7.1 platform requirements.

## Isaac Sim 5.1 interoperability

Two simulator-specific failures need targeted handling:

- NITROS Bridge topics from Isaac Sim 5.1 might not arrive through DDS. This
  breaks the object-following manipulation simulation tutorial; treat it as a
  transport interoperability failure rather than an application-graph failure.
- If the Nvblox sample scene does not load normally, open it from Content
  Window → Samples → NvBlox → `nvblox_sample_scene.usd`.

The Isaac Sim stereo-image-processing workflow can also fail to visualize its
point cloud in RViz. In that case, inspect `PointCloud2` conversion and frame
metadata rather than assuming stereo inference produced no output.

