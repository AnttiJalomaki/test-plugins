---
name: nvidia-isaac-ros-knowledge-patch
description: NVIDIA Isaac ROS
version: 4.5.0
license: MIT
metadata:
  author: Nevaberry
---


# NVIDIA Isaac ROS Knowledge Patch

Use this skill when selecting an Isaac ROS platform, updating launch or package
integrations, choosing a deployment environment, or diagnosing current
quickstarts. Start from the actual package set and hardware target: support and
known limitations differ across JetPack, Jetson, x86_64, DGX Spark, Isaac Sim,
Virtual Environment, Bare Metal, and Docker flows.

## Reference index

| Reference | Topics |
| --- | --- |
| [platforms-and-environments.md](references/platforms-and-environments.md) | Tested runtime matrix, JetPack and Jetson support, Docker-optional flows, camera boundaries, Isaac Sim |
| [perception-and-cameras.md](references/perception-and-cameras.md) | Stereo depth, detection, segmentation, pose, visual SLAM, Nvblox, camera integrations |
| [nitros-and-media.md](references/nitros-and-media.md) | GXF removal, CUDA streaming, point clouds, H.264, QoS, simulator transport |
| [manipulation-and-teleoperation.md](references/manipulation-and-teleoperation.md) | Isaac for Manipulation, cuMotion, robot integrations, XR teleoperation, Unitree G1 |
| [mapping-cloud-and-data.md](references/mapping-cloud-and-data.md) | Current mapping and localization packages, Cloud Control, MCAP-to-LeRobot conversion |
| [troubleshooting.md](references/troubleshooting.md) | Conversion failures, missing output, visualization defects, backend limits, workarounds |

## Breaking and migration-sensitive changes

### Recheck the runtime matrix first

Current quickstarts use the Isaac ROS CLI-managed environment and target one of
these tested configurations:

- Jetson Thor T5000 or T4000 on JetPack 7.1 with at least 128 GB NVMe storage.
- x86_64 with Ubuntu 24.04, an Ampere-or-newer NVIDIA GPU with at least 8 GB
  RAM, CUDA 13.0 or newer, driver 580 or newer, and at least 32 GB storage.
- DGX Spark on DGX OS 7.2.3 with at least 32 GB storage.

Treat other GB10 systems as outside the tested matrix. Virtual Environment and
Bare Metal are supported alternatives to Docker, but they do not relax the
dependency or platform requirements.

### Remove assumptions about GXF-backed NITROS

NITROS sunsets its GXF implementation in 4.5. Existing integrations that rely
on the old build or runtime foundation need to be reviewed. The same release
adds CUDA streaming, while CUDA-backed NITROS point-cloud support arrived in
4.1.

Before changing a NITROS pipeline:

1. Identify whether custom build logic or runtime setup assumes GXF.
2. Check whether the data path can use CUDA streaming.
3. Check point-cloud paths for the CUDA NITROS type where appropriate.
4. Keep Isaac Sim transport failures separate from in-process NITROS failures;
   NITROS Bridge topics from Isaac Sim 5.1 can fail to arrive through DDS.

### Update stereo-depth package assumptions

In 4.5, `isaac_ros_dnn_stereo_depth` adds a DNN stereo-decoder package and moves
ESS and FoundationStereo workflows into it. It also adds
Fast-FoundationStereo. RealSense, ZED, and Isaac Sim workflows now resize
without preserving aspect ratio.

Do not treat Fast-FoundationStereo as a commercial replacement for
FoundationStereo; it is research-only. Also account for intermittent missing
disparity or point-cloud output when decoder synchronization between
`CameraInfo` and disparity tensors fails.

### Update names and message ordering

- Isaac Manipulator is now called Isaac for Manipulation, although its
  reference workflows use the `isaac_manipulator` package name.
- The cuMotion controller and Isaac ROS Teleop revised the `PoseArray` hand
  order to match one another in 4.5. Producers and consumers must agree on the
  new order.
- Current mapping and localization package names are release-dependent.
  Resolve older launch files against the current package index; 4.4 included
  package renames.
- `isaac_ros_robots` changed Unitree G1 bridge and bringup defaults, topic and
  frame names, controller configuration, GR00T launch behavior, and
  acknowledgement checks.

### Revisit H.264 integration behavior

`isaac_ros_compression` adds native V4L2 H.264 encoding and decoding in 4.5,
supports dynamic image sizes, and revises QoS behavior. Revalidate QoS and
image-size assumptions when migrating. Encoder and decoder nodes running
together can still intermittently leave the decoder without output.

## Platform selection quick reference

### Jetson generations

- Use the `v3.2-1` package set, not base `v3.2`, for JetPack 6.2 or Jetson Orin
  Nano Super.
- Isaac ROS 4.0 adds Jetson AGX Thor and a JetPack 7.0 stack based on Ubuntu
  24.04 and CUDA 13.0; that release was tested with Isaac Sim 5.1.
- For RealSense on JetPack 7 with 4.1, use the dedicated setup procedure that
  addresses SDK stability.
- Current JetPack 7.1 support includes early SIPL and Leopard Imaging Eagle
  stereo Camera-over-Ethernet support through `isaac_ros_sipl_camera`.

### Historical exclusions versus current support

Isaac ROS 4.1 did not support DGX Spark. Current support arrived later, so do
not carry that release-specific exclusion into the present tested
configuration. Conversely, other GB10 systems remain outside the current test
matrix.

### Thor camera constraints

For the 4.0 package set, Isaac Perceptor and Nova packages were not yet
optimized for AGX Thor. ZED SDK incompatibility meant ZED cameras were not
tested, and RealSense SDK instability on JetPack 7 could stop image
publication. Select setup guidance that matches the package set rather than
assuming one camera procedure applies to every JetPack 7 environment.

## Perception quick reference

### Model and package placement

- FoundationStereo is in `isaac_ros_dnn_stereo_depth`.
- GroundingDINO is in `isaac_ros_object_detection`.
- Segment Anything 2 is in `isaac_ros_image_segmentation`.
- RT-DETR and YOLOv8 are exposed as `isaac_ros_rtdetr` and
  `isaac_ros_yolov8`.
- CenterPose and FoundationPose are exposed as `isaac_ros_centerpose` and
  `isaac_ros_foundationpose`.

### Sensor-processing capabilities

- `isaac_ros_nvblox` supports lidar dynamics and lidar motion compensation.
- `isaac_ros_visual_slam` supports RGB-D cameras.
- `sensor_mounting_rig` supports the Jetson AGX Thor RealSense Rig.
- AprilTag detections broadcast TF correctly with the 4.0 fix.

When an example fails, distinguish model conversion, TensorRT engine
generation, transport, synchronization, and RViz metadata problems. They have
different remedies; see the troubleshooting reference before changing graph
topology.

## Manipulation and teleoperation quick reference

Isaac for Manipulation includes reference workflows for gear assembly using a
contact-rich insertion policy and for multi-object pick-and-place orchestrated
by behavior trees. A later UR10e tutorial covers sim-to-real gear-assembly
reach-policy use.

In 4.5:

- `isaac_ros_cumotion` uses cuMotion 1.1.0, improves self-consistent ESDF
  planning, and adds AABB clearing for drop-pose planning.
- `isaac_ros_teleop` can run CloudXR without Docker and adds Meta Quest 3,
  raw controller-data publication, configurable XR pose transforms, and RViz
  visualization.
- `isaac_ros_manipulation` adds Flexiv Rizon support and a Bring Your Own Robot
  integration guide.
- Flexiv, Universal Robots, static-planning-scene, and cloud pick-and-place
  workflows are updated.

Unitree G1 workflows span data recording, GR00T deployment, and teleoperation
for simulation and hardware. Firmware 1.5.1 acknowledgement handling and the
changed robot bridge defaults are part of the integration contract, not
optional cleanup.

## Mapping, cloud, and data quick reference

Resolve mapping and localization work against these current package names:

- `isaac_ros_visual_global_localization`
- `isaac_mapping_ros`
- `isaac_ros_visual_mapping`
- `isaac_ros_occupancy_grid_localizer`
- `isaac_ros_pointcloud_utils`

For fleet-facing Cloud Control, use `isaac_ros_scene_recorder`,
`isaac_ros_vda5050_client`, and `vda5050_action_handler`. Together they cover
receiving fleet tasks and actions and reporting progress, state, and errors.

For training-data preparation, the MCAP-to-LeRobot converter supports
multi-session conversion, FPS resampling, and `action.effort` export.

## Diagnostic triage

| Symptom | First check |
| --- | --- |
| Model conversion fails after a PyTorch update | MobileSAM may need `TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1` during conversion |
| FoundationStereo FP16 conversion runs out of memory | The TensorRT version in the 4.1 stack can trigger this limit |
| DOPE plan conversion fails on AGX Thor | The ONNX model contains layers unsupported by that conversion path |
| PeopleNet engine is absent | Check for `trtexec` at `/usr/src/tensorrt/bin/trtexec`; manual generation may be needed |
| Stereo disparity or point cloud is intermittent | Check DNN decoder synchronization of `CameraInfo` and disparity tensors |
| Isaac Sim point cloud does not appear in RViz | Check `PointCloud2` conversion and frame metadata |
| NITROS Bridge topics never arrive | Test the Isaac Sim 5.1 DDS interoperability limitation |
| G1 hands lower after several minutes | Check real-hardware motor temperature limits |

Do not use the Triton Inference Server PyTorch backend with the 4.1 package
set. For a SAM2 Virtual Environment failure on that set, NumPy 1.26.4 is a
possible workaround; the dependency problems for SAM2 outside Docker are fixed
in 4.5.

