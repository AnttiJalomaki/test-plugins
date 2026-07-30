# Troubleshooting

## Conversion and engine generation

### MobileSAM under PyTorch 2.6

PyTorch 2.6 changes the `weights_only` default, which can break
MobileSAM-to-ONNX conversion. Restore the prior loading behavior for the
conversion process (4.0.0):

```bash
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
```

### FoundationStereo FP16

FoundationStereo conversion in FP16 can run out of memory because of the
TensorRT version used by Isaac ROS 4.1.0.

### DOPE on Jetson AGX Thor

The DOPE quickstart cannot convert its ONNX network to a TensorRT Plan on
Jetson AGX Thor because the network contains unsupported layers (4.1.0).

### PeopleNet

If `trtexec` is not installed at
`/usr/src/tensorrt/bin/trtexec`, the PeopleNet quickstart may require manual
TensorRT-engine generation (4.5.0).

## Dependency and environment failures

The Triton Inference Server PyTorch backend is unsupported in Isaac ROS 4.1.0.

The SAM2 visualization script can fail in the 4.1.0 Virtual Environment flow
because of a NumPy mismatch. Downgrading NumPy to 1.26.4 is a possible
workaround. The SAM2 quickstart dependency problems affecting Virtual
Environment and Bare Metal flows are fixed in 4.5.0.

Docker-optional modes still must satisfy the current hardware, operating
system, CUDA, driver, and storage matrix. See
[platforms-and-environments](platforms-and-environments.md).

## Missing output

### H.264 decoder

Running `isaac_ros_h264_decoder` alongside `isaac_ros_h264_encoder` can
intermittently leave the decoder without output (4.0.0).

### DNN stereo

With RealSense, ZED, or Isaac Sim, DNN stereo can intermittently omit
disparity or point-cloud output. The decoder does not synchronize
`CameraInfo` messages with disparity tensors (4.5.0).

### JetPack 7 RealSense

In 4.0.0, RealSense SDK instability on JetPack 7 could stop image
publication. Use the dedicated Isaac ROS 4.1.0 RealSense-on-JetPack-7 setup
procedure.

### Isaac Sim bridge

NITROS Bridge topics from Isaac Sim 5.1 may not arrive over DDS, breaking the
object-following manipulation simulation tutorial (4.0.0).

## Detection anomalies

DOPE can fail to detect objects in manipulation workflows (4.0.0).

DetectNet can produce overlapping duplicate boxes because of its DBScan
implementation (4.0.0).

## RViz and scene visualization

The Nvblox RealSense people-segmentation example can show an incorrect RViz
color overlay (4.0.0).

The Isaac Sim stereo-image-processing workflow can fail to display its point
cloud in RViz because of `PointCloud2` conversion or frame metadata (4.5.0).

If the Nvblox sample scene does not load normally in Isaac Sim 5.1, open it
through:

`Content Window → Samples → NvBlox → nvblox_sample_scene.usd`

## Hardware and platform boundaries

In 4.0.0, Isaac Perceptor and Nova packages were not optimized for AGX Thor,
the ZED SDK was incompatible with Jetson Thor, and ZED cameras were not
tested.

Isaac ROS 4.1.0 did not support DGX Spark, but that restriction is historical.
DGX Spark and JetPack 7.1 support arrived on 2026-02-19, and DGX Spark is part
of the current runtime matrix (`current-runtime-and-packages`).

During real-hardware Unitree G1 teleoperation, the hands can lower after
several minutes because of motor temperature limits (4.5.0).
