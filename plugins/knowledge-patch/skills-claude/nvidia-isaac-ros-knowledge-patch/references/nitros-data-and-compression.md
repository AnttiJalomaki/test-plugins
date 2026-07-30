# NITROS, data, compression, and cloud control

## Migrate away from the GXF foundation

NITROS sunsets its GXF implementation in 4.5.0. Revisit custom integrations
whose builds, runtime assumptions, or graph structure depend on the former GXF
foundation.

The same release adds CUDA streaming to NITROS messaging.

## Transport point clouds and camera images

NITROS supports point clouds with CUDA as of 4.1.0. Early SIPL camera support
uses `isaac_ros_sipl_camera` to publish images through zero-copy NITROS
(`current-runtime-and-packages`).

Isaac Sim 5.1 NITROS Bridge topics may fail to arrive over DDS. The observable
effect in 4.0.0 is a broken object-following manipulation simulation tutorial;
separate this bridge problem from failures inside the receiving pipeline.

## Encode and decode H.264

`isaac_ros_compression` adds native V4L2 H.264 encoding and decoding in 4.5.0.
It also supports dynamic image sizes and revises QoS behavior. Recheck QoS
assumptions when updating existing compression graphs.

Running `isaac_ros_h264_decoder` alongside `isaac_ros_h264_encoder` can
intermittently leave the decoder without output (4.0.0). Treat a no-output
event in that pairing as a known coexistence issue before diagnosing the input
stream.

## Convert MCAP data to LeRobot

`isaac_ros_data_tools` includes an MCAP-to-LeRobot converter in 4.5.0. It
supports:

- conversion across multiple sessions;
- FPS resampling; and
- export of `action.effort`.

## Integrate cloud-controlled fleet tasks

The current Cloud Control surface includes
`isaac_ros_scene_recorder`, `isaac_ros_vda5050_client`, and
`vda5050_action_handler` (`current-runtime-and-packages`).

Use these packages to receive fleet tasks and actions and to report progress,
state, and errors.
