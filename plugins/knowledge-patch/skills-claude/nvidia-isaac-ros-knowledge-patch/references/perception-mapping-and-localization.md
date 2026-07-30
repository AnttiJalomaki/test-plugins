# Perception, mapping, and localization

## Build stereo-depth pipelines

FoundationStereo was added to `isaac_ros_dnn_stereo_depth` in 4.0.0.

In 4.5.0, `isaac_ros_dnn_stereo_depth` adds a DNN stereo-decoder package and
moves the ESS and FoundationStereo workflows into that package. It also adds
Fast-FoundationStereo. RealSense, ZED, and Isaac Sim workflows resize images
without retaining aspect ratio.

Fast-FoundationStereo is research-only; use FoundationStereo for commercial
work.

DNN stereo pipelines using RealSense, ZED, or Isaac Sim can intermittently
omit disparity or point-cloud output. The decoder does not synchronize
`CameraInfo` messages with disparity tensors.

FoundationStereo FP16 conversion can run out of memory because of the TensorRT
version used by Isaac ROS 4.1.0.

## Configure detection, segmentation, and pose

The 4.0.0 integrations include:

| Capability | Package |
| --- | --- |
| GroundingDINO | `isaac_ros_object_detection` |
| Segment Anything 2 | `isaac_ros_image_segmentation` |

The current package index also includes
`isaac_ros_rtdetr`, `isaac_ros_yolov8`, `isaac_ros_centerpose`, and
`isaac_ros_foundationpose` for RT-DETR, YOLOv8, CenterPose, and FoundationPose
respectively (`current-runtime-and-packages`).

DOPE can fail to detect objects in manipulation workflows (4.0.0). On Jetson
AGX Thor, the 4.1.0 DOPE quickstart cannot convert its ONNX network to a
TensorRT Plan because the network contains unsupported layers.

DetectNet can emit overlapping duplicate boxes because of its DBScan
implementation (4.0.0).

If PyTorch 2.6 breaks MobileSAM-to-ONNX conversion because of its changed
`weights_only` default, set this for the conversion process (4.0.0):

```bash
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
```

For SAM2 in Docker-optional environments, consult
[platforms-and-environments](platforms-and-environments.md).

## Use visual SLAM and AprilTags

`isaac_ros_visual_slam` supports RGB-D cameras as of 4.1.0.

Isaac ROS 4.0.0 fixes TF broadcasting for AprilTags detected by
`isaac_ros_apriltag`.

## Configure Nvblox with lidar

`isaac_ros_nvblox` adds dynamics support for lidar inputs and lidar motion
compensation in 4.1.0.

The Nvblox RealSense people-segmentation example can display an incorrect RViz
color overlay (4.0.0). This is a visualization caveat, not evidence that the
segmentation output itself has the same color interpretation.

If the Isaac Sim Nvblox sample scene fails to load normally, use the Content
Window path documented in
[platforms-and-environments](platforms-and-environments.md).

## Resolve mapping and localization packages

The current mapping and localization surface consists of:

- `isaac_ros_visual_global_localization`
- `isaac_mapping_ros`
- `isaac_ros_visual_mapping`
- `isaac_ros_occupancy_grid_localizer`
- `isaac_ros_pointcloud_utils`

Supported packages are release-dependent, and 4.4 included package renames.
Resolve older launch files against the current package index instead of
assuming their package names remain valid (`current-runtime-and-packages`).
