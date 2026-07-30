# Troubleshooting

## Model conversion and engine generation

### MobileSAM fails to convert to ONNX

PyTorch 2.6 changes the `weights_only` default, which can break the MobileSAM
conversion flow in 4.0.0. Restore the earlier load behavior for the conversion
process:

```bash
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
```

Scope the environment variable to the conversion process; it is a loading
compatibility workaround, not a general runtime requirement.

### FoundationStereo FP16 conversion runs out of memory

FoundationStereo conversion in FP16 can exhaust memory because of the
TensorRT version used by Isaac ROS 4.1.0. Treat this as a known stack-specific
conversion limit.

### DOPE cannot build a TensorRT Plan on AGX Thor

The 4.1.0 DOPE quickstart cannot convert the model from ONNX to a TensorRT Plan
on Jetson AGX Thor because the model contains unsupported layers. This is
separate from the runtime issue where DOPE can fail to detect objects in
manipulation workflows.

### PeopleNet has no TensorRT engine

The quickstart expects `trtexec` at
`/usr/src/tensorrt/bin/trtexec`. If it is not installed there, manually
generating the TensorRT engine might be required.

## Environment and backend failures

### SAM2 visualization fails outside Docker

In the 4.1.0 Virtual Environment flow, a NumPy mismatch can break the SAM2
visualization script. Downgrading NumPy to 1.26.4 is a possible workaround.
The SAM2 quickstart dependency problems affecting Virtual Environment and Bare
Metal are fixed in 4.5.0.

### Triton PyTorch backend is selected

The Triton Inference Server PyTorch backend is not supported in Isaac ROS
4.1.0. Select a supported inference path rather than debugging that backend as
if it were part of the supported matrix.

### RealSense stops publishing on JetPack 7

The RealSense SDK could become unstable and stop publishing images in the
4.0.0 JetPack 7 flow. Use the dedicated Isaac ROS 4.1.0 RealSense-on-JetPack-7
setup procedure, which addresses that stability problem.

## Missing or intermittent output

### H.264 decoder has no output

Running `isaac_ros_h264_decoder` alongside `isaac_ros_h264_encoder` can
intermittently leave the decoder without output. Reproduce with both nodes
active; testing them only in isolation misses the known interaction.

When moving to the native V4L2 H.264 path in 4.5.0, also recheck revised QoS
behavior and dynamic image-size handling.

### DNN stereo omits disparity or point clouds

With RealSense, ZED, or Isaac Sim, the DNN stereo decoder can intermittently
omit disparity or point-cloud output because it does not synchronize
`CameraInfo` messages with disparity tensors (4.5.0). Inspect arrival timing
and synchronization before diagnosing the CUDA point-cloud path.

### NITROS Bridge topics do not arrive

NITROS Bridge topics from Isaac Sim 5.1 might not arrive through DDS in the
4.0.0 stack. This breaks the object-following manipulation simulation
tutorial. Isolate the bridge and DDS boundary before changing the manipulation
workflow.

## Incorrect or absent visualization

### Nvblox sample scene does not load

Open it through Content Window → Samples → NvBlox →
`nvblox_sample_scene.usd`.

### Isaac Sim stereo point cloud is absent in RViz

The stereo-image-processing workflow can fail to visualize its point cloud
because of `PointCloud2` conversion or frame metadata. Confirm the conversion
and frame information before concluding that stereo output is absent.

### Nvblox people segmentation has the wrong overlay

The Nvblox RealSense people-segmentation example can show an incorrect RViz
color overlay. Keep the display defect distinct from segmentation and mapping
correctness.

### DetectNet produces overlapping boxes

DetectNet can emit overlapping duplicate boxes because of its DBScan
implementation. Check detector post-processing before looking for duplicate
message publication.

## Hardware and policy boundaries

### Fast-FoundationStereo is selected for commercial work

Fast-FoundationStereo is a research-only model in 4.5.0. Use
FoundationStereo for commercial work.

### Unitree G1 hands lower during teleoperation

On physical hardware, G1 hands can lower after several minutes because of
motor temperature limits. Check hand motor temperature before debugging the
controller, firmware 1.5.1 acknowledgement handling, or revised bridge
defaults.

