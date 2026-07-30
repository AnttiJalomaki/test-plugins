# NITROS and media

## Migrate away from the GXF-backed foundation

NITROS sunsets its GXF implementation in 4.5.0. This changes the build and
runtime foundation for NITROS integrations. Review custom packages for GXF
build assumptions, runtime setup, and component dependencies instead of
treating the update as only a message-format change.

NITROS messaging also gains CUDA streaming support in 4.5.0. Evaluate it for
GPU-resident paths after removing old foundation assumptions.

## Use CUDA point clouds

`isaac_ros_nitros` adds point-cloud support for CUDA with NITROS in 4.1.0.
Keep this capability distinct from later DNN stereo decoder failures: missing
stereo point clouds can result from unsynchronized `CameraInfo` and disparity
tensors before the point-cloud path is exercised.

## Update H.264 pipelines

`isaac_ros_compression` adds native V4L2 H.264 encoding and decoding in 4.5.0.
It also supports dynamic image sizes and revises QoS behavior.

When adopting it:

1. Revalidate publisher and subscriber QoS compatibility.
2. Remove fixed-dimension assumptions if dynamic image sizes are required.
3. Verify the chosen path actually uses the native V4L2 implementation.
4. Exercise simultaneous encode and decode, not just each node independently.

Running `isaac_ros_h264_decoder` alongside `isaac_ros_h264_encoder` can
intermittently leave the decoder without output. Preserve that failure as a
known runtime possibility when building watchdogs or health checks.

## Separate simulator transport from NITROS graph failures

With Isaac Sim 5.1, NITROS Bridge topics might not arrive through DDS
(4.0.0). This breaks the object-following manipulation simulation tutorial.
If in-process NITROS paths work but bridged topics never appear, diagnose the
simulator-to-DDS boundary before rewriting type negotiation or graph topology.

The Isaac Sim stereo-image-processing workflow has another distinct symptom:
point-cloud visualization can fail in RViz because of `PointCloud2` conversion
or frame metadata. That does not necessarily indicate DDS loss.

## Zero-copy SIPL images

Current early SIPL and Leopard Imaging Eagle stereo Camera-over-Ethernet
support uses `isaac_ros_sipl_camera`. The package publishes SIPL camera images
through zero-copy NITROS and belongs to the current JetPack 7.1 support
surface.

