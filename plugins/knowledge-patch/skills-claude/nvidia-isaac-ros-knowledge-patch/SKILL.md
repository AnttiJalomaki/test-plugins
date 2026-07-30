---
name: nvidia-isaac-ros-knowledge-patch
description: NVIDIA Isaac ROS
version: 4.5.0
license: MIT
metadata:
  author: Nevaberry
---


# NVIDIA Isaac ROS Knowledge Patch

Use this skill when planning, configuring, upgrading, or troubleshooting NVIDIA
Isaac ROS deployments, especially when platform support, package names, NITROS
behavior, perception pipelines, or manipulation workflows may have changed.

## How to use this skill

1. Identify the exact Isaac ROS package set and the target hardware.
2. Match the deployment against the current runtime matrix before debugging an
   individual package.
3. Check the breaking changes and renamed surfaces below.
4. Open the topic reference that matches the task.
5. Treat release-specific limitations as scoped facts; later current-runtime
   guidance can supersede an earlier platform exclusion.
6. Verify older launch files against the installed package index before
   carrying package names forward.

## Reference index

| Reference | Topics |
| --- | --- |
| [platforms-and-environments](references/platforms-and-environments.md) | JetPack, Jetson, x86_64, DGX Spark, Isaac Sim, Docker-optional modes, cameras |
| [nitros-data-and-compression](references/nitros-data-and-compression.md) | GXF removal, CUDA streaming, point clouds, H.264, MCAP and fleet-task packages |
| [perception-mapping-and-localization](references/perception-mapping-and-localization.md) | Stereo depth, segmentation, detection, SLAM, Nvblox, mapping and localization packages |
| [manipulation-teleoperation-and-robots](references/manipulation-teleoperation-and-robots.md) | Isaac for Manipulation, cuMotion, CloudXR, robot integration, Unitree G1 |
| [troubleshooting](references/troubleshooting.md) | Conversion failures, missing output, RViz issues, dependency and hardware limits |

## Breaking changes and renamed surfaces

### NITROS no longer uses GXF

NITROS sunsets its GXF implementation in 4.5. Integrations that assume the GXF
build or runtime foundation must be revisited. NITROS messaging also supports
CUDA streaming.

### Stereo workflows moved

The DNN stereo-decoder package now owns the ESS and FoundationStereo workflows.
Fast-FoundationStereo is also available. RealSense, ZED, and Isaac Sim stereo
workflows resize without preserving aspect ratio.

### Hand order changed

The cuMotion controller and Isaac ROS Teleop revised the hand ordering in their
`PoseArray` messages. Update consumers that interpret positions by array index
and verify both sides use the same ordering.

### Manipulator naming

Isaac Manipulator is now called Isaac for Manipulation. Its documented
reference workflows are identified under `isaac_manipulator`.

### Mapping and localization names are release-sensitive

Resolve old launch files against the current package index. The current surface
includes:

- `isaac_ros_visual_global_localization`
- `isaac_mapping_ros`
- `isaac_ros_visual_mapping`
- `isaac_ros_occupancy_grid_localizer`
- `isaac_ros_pointcloud_utils`

Package renames occurred in 4.4, so do not infer a package name from an older
launch file.

## Platform quick reference

### Current tested targets

Use the Isaac ROS CLI-managed ROS 2 Jazzy environment on one of these tested
targets:

| Target | Required environment |
| --- | --- |
| Jetson Thor T5000 or T4000 | JetPack 7.1 and at least 128 GB NVMe storage |
| x86_64 | Ampere-or-newer NVIDIA GPU with at least 8 GB RAM, Ubuntu 24.04, CUDA 13.0 or newer, driver 580 or newer, and at least 32 GB storage |
| DGX Spark | DGX OS 7.2.3 and at least 32 GB storage |

Other GB10 systems are outside the tested matrix. Virtual Environment and Bare
Metal modes remove the Docker requirement, but not the platform and dependency
requirements.

### Earlier platform decisions

- For JetPack 6.2 or Jetson Orin Nano Super, use the `v3.2-1` package set
  rather than base `v3.2`.
- Isaac ROS 4.0 introduced Jetson AGX Thor and a JetPack 7.0 stack based on
  Ubuntu 24.04 and CUDA 13.0; it was tested with Isaac Sim 5.1.
- The 4.1 DGX Spark exclusion is historical. DGX Spark and JetPack 7.1 support
  arrived on 2026-02-19.
- Use the dedicated 4.1 RealSense-on-JetPack-7 setup procedure when working in
  that environment.

## Perception quick reference

### Added integrations

| Capability | Package or integration |
| --- | --- |
| FoundationStereo | `isaac_ros_dnn_stereo_depth` |
| GroundingDINO | `isaac_ros_object_detection` |
| Segment Anything 2 | `isaac_ros_image_segmentation` |
| RT-DETR | `isaac_ros_rtdetr` |
| YOLOv8 | `isaac_ros_yolov8` |
| CenterPose | `isaac_ros_centerpose` |
| FoundationPose | `isaac_ros_foundationpose` |

`isaac_ros_visual_slam` supports RGB-D cameras. `isaac_ros_nvblox` supports
lidar dynamics and lidar motion compensation.

### Stereo production choice

Fast-FoundationStereo is research-only. Use FoundationStereo for commercial
work. With RealSense, ZED, or Isaac Sim, the DNN stereo decoder can
intermittently omit disparity or point-cloud output because `CameraInfo`
messages are not synchronized with disparity tensors.

## Manipulation and robot quick reference

### Planning and integration

cuMotion 1.1.0 improves self-consistent ESDF planning and adds AABB clearing
for drop-pose planning. Manipulation integrations include Flexiv Rizon and a
Bring Your Own Robot guide; Flexiv, Universal Robots, static-planning-scene,
and cloud pick-and-place workflows were updated.

Reference workflows include:

- gear assembly with a contact-rich insertion policy;
- behavior-tree orchestration for multi-object pick-and-place; and
- a sim-to-real UR10e gear-assembly reach-policy tutorial.

### Teleoperation

CloudXR teleoperation can run without Docker. The teleoperation surface adds
Meta Quest 3 support, raw controller-data publication, configurable XR pose
transforms, and RViz visualization.

### Unitree G1

The physical-AI packages add data recording, GR00T deployment, and G1
teleoperation for simulation and hardware. Check bridge and bringup defaults,
topic and frame names, controller configuration, launch behavior,
acknowledgement handling, and firmware 1.5.1 acknowledgement requirements
before reusing older configurations.

## NITROS and data quick reference

- CUDA point clouds are supported through NITROS.
- `isaac_ros_compression` provides native V4L2 H.264 encoding and decoding,
  supports dynamic image sizes, and has revised QoS behavior.
- The MCAP-to-LeRobot converter supports multi-session conversion, FPS
  resampling, and `action.effort` export.
- SIPL cameras publish through zero-copy NITROS using
  `isaac_ros_sipl_camera`.

## High-priority troubleshooting

### Conversion and dependency failures

- For MobileSAM-to-ONNX conversion under PyTorch 2.6, restore the earlier
  loading behavior for the conversion process:

  ```bash
  export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
  ```

- FoundationStereo FP16 conversion can exhaust memory with the TensorRT
  version in Isaac ROS 4.1.
- DOPE cannot convert its ONNX network to a TensorRT Plan on Jetson AGX Thor
  when unsupported layers are present.
- The Triton Inference Server PyTorch backend is unsupported in Isaac ROS 4.1.
- If `trtexec` is absent from `/usr/src/tensorrt/bin/trtexec`, the PeopleNet
  quickstart may need a manually generated TensorRT engine.

### Missing or misleading runtime output

- An H.264 decoder run beside the encoder can intermittently produce no
  decoder output.
- DNN stereo can omit disparity or point clouds.
- DetectNet can emit overlapping duplicate boxes.
- The Nvblox RealSense people-segmentation example can show the wrong RViz
  overlay color.
- Isaac Sim stereo processing can fail to visualize a point cloud in RViz.

Open [troubleshooting](references/troubleshooting.md) for causes, scopes, and
the remaining simulation, SAM2, DOPE, and Unitree G1 caveats.

## Working rules

- Prefer current package-index names over copied historical launch files.
- Keep hardware, operating system, CUDA, driver, and storage checks together;
  a Docker-optional flow is not a platform-optional flow.
- Scope workarounds to the affected release and environment.
- Revalidate indexed `PoseArray` consumers after cuMotion or Teleop updates.
- For stereo pipelines, distinguish conversion failure, decoder
  synchronization loss, and visualization metadata failure.
- For G1 hardware, include acknowledgement and motor-temperature behavior in
  operational checks.
