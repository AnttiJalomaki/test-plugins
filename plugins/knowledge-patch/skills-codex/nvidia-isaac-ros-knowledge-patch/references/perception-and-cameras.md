# Perception and cameras

## Find model integrations by package

The perception model surface added in 4.0.0 includes:

| Capability | Package |
| --- | --- |
| FoundationStereo | `isaac_ros_dnn_stereo_depth` |
| GroundingDINO | `isaac_ros_object_detection` |
| Segment Anything 2 | `isaac_ros_image_segmentation` |

The current package index also includes:

| Capability | Package |
| --- | --- |
| RT-DETR | `isaac_ros_rtdetr` |
| YOLOv8 | `isaac_ros_yolov8` |
| CenterPose | `isaac_ros_centerpose` |
| FoundationPose | `isaac_ros_foundationpose` |

## Integrate DNN stereo depth

In 4.5.0, `isaac_ros_dnn_stereo_depth` adds a DNN stereo-decoder package and
moves the ESS and FoundationStereo workflows into it. The same package adds
Fast-FoundationStereo.

RealSense, ZED, and Isaac Sim workflows now resize without retaining aspect
ratio. Recheck camera calibration and downstream dimensional assumptions when
moving one of those workflows.

Observe two operational boundaries:

- The decoder can intermittently omit disparity or point-cloud output because
  it does not synchronize `CameraInfo` messages with disparity tensors.
- Fast-FoundationStereo is research-only. Use FoundationStereo for commercial
  work.

FoundationStereo FP16 conversion can run out of memory with the TensorRT
version used by the 4.1.0 stack. Treat that as a conversion-stack memory limit,
not evidence that the source model or stereo graph is invalid.

## Configure segmentation and detection conversions

PyTorch 2.6 changes the `weights_only` default and can break
MobileSAM-to-ONNX conversion in the 4.0.0 flow. Restore the previous loading
behavior for that conversion process:

```bash
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
```

For SAM2, the NumPy mismatch affecting its visualization script in the 4.1.0
Virtual Environment flow can be worked around with NumPy 1.26.4. The SAM2
quickstart dependency problems in Virtual Environment and Bare Metal flows are
fixed in 4.5.0.

DOPE has two distinct failure modes:

- It can fail to detect objects in manipulation workflows.
- On Jetson AGX Thor, the 4.1 quickstart cannot convert its model from ONNX to
  a TensorRT Plan because the model contains unsupported layers.

DetectNet can emit overlapping duplicate boxes because of its DBScan
implementation. Do not attribute those duplicates to transport-level message
replay without checking the detector behavior.

If `trtexec` is not installed at
`/usr/src/tensorrt/bin/trtexec`, the PeopleNet quickstart might require manual
TensorRT engine generation.

## Use RGB-D, lidar, and fiducial capabilities

`isaac_ros_nvblox` adds dynamics support for lidar inputs and lidar motion
compensation in 4.1.0. Its RealSense people-segmentation example can show an
incorrect RViz color overlay; that visualization defect does not by itself
establish that the people mask or map is wrong.

`isaac_ros_visual_slam` adds RGB-D camera support in 4.1.0.

Isaac ROS 4.0 fixes TF broadcasting for AprilTags detected by
`isaac_ros_apriltag` (4.0.0). When carrying forward an older workaround, first
check whether it compensates for the pre-fix broadcasting behavior.

## Keep camera support boundaries explicit

For the 4.0.0 stack on AGX Thor, ZED SDK incompatibility meant ZED cameras were
not tested, while the RealSense SDK could become unstable on JetPack 7 and
stop publishing images. The dedicated 4.1.0 RealSense-on-JetPack-7 setup
procedure addresses that RealSense stability problem.

`sensor_mounting_rig` supports the Jetson AGX Thor RealSense Rig in 4.1.0.
Current early Camera-over-Ethernet support includes SIPL and Leopard Imaging
Eagle stereo cameras; `isaac_ros_sipl_camera` publishes their images through
zero-copy NITROS.

