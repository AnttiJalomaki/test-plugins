# Platforms and environments

## Select a tested runtime

Current quickstarts use ROS 2 Jazzy through the Isaac ROS CLI-managed
environment. The tested targets are:

| Target | Runtime and capacity |
| --- | --- |
| Jetson Thor T5000 or T4000 | JetPack 7.1 with at least 128 GB NVMe storage |
| x86_64 | Ampere-or-newer NVIDIA GPU with at least 8 GB RAM, Ubuntu 24.04, CUDA 13.0 or newer, driver 580 or newer, and at least 32 GB storage |
| DGX Spark | DGX OS 7.2.3 with at least 32 GB storage |

Other GB10 systems are outside the current test matrix. Docker-optional
Virtual Environment and Bare Metal deployments must still satisfy this
dependency and platform matrix (`current-runtime-and-packages`).

## Choose an environment mode

Virtual Environment and Bare Metal became supported development and deployment
modes in 4.1.0, so Docker is not required for those flows. This changes the
container requirement, not the target matrix.

SAM2 initially had a Virtual Environment NumPy mismatch. Pinning NumPy 1.26.4
was a possible 4.1.0 workaround. The dependency problems affecting SAM2
quickstarts in both Virtual Environment and Bare Metal flows are fixed in
4.5.0.

## Match JetPack and Jetson hardware

### JetPack 6.2 and Orin Nano Super

The 3.2-1 update adds JetPack 6.2 and Jetson Orin Nano Super support. Select
the `v3.2-1` package set rather than the base `v3.2` release for either target.

### JetPack 7 and AGX Thor

Isaac ROS 4.0.0 adds Jetson AGX Thor support and a JetPack 7.0 stack based on
Ubuntu 24.04 and CUDA 13.0. That combination was tested with Isaac Sim 5.1.

At that point, Isaac Perceptor and Nova packages were not optimized for AGX
Thor. The ZED SDK was incompatible with Jetson Thor, so ZED cameras were not
tested. RealSense SDK support on JetPack 7 could become unstable and stop
publishing images.

In 4.1.0, use the dedicated RealSense-on-JetPack-7 setup procedure, which
addresses that SDK stability problem. `sensor_mounting_rig` also supports the
Jetson AGX Thor RealSense Rig.

## Interpret DGX Spark support by date

Isaac ROS 4.1.0 did not support DGX Spark. That exclusion no longer represents
the current environment: DGX Spark and JetPack 7.1 support arrived on
2026-02-19. Current DGX Spark quickstarts use DGX OS 7.2.3 and require at least
32 GB storage (`current-runtime-and-packages`).

## Use SIPL Camera-over-Ethernet

Early SIPL and Leopard Imaging Eagle stereo Camera-over-Ethernet support
arrived on 2026-03-23. `isaac_ros_sipl_camera` publishes SIPL camera images
through zero-copy NITROS (`current-runtime-and-packages`).

## Work with Isaac Sim 5.1

NITROS Bridge topics from Isaac Sim 5.1 might not arrive through DDS. This can
break the object-following manipulation simulation tutorial (4.0.0).

If the Nvblox sample scene does not load normally, open it through:

`Content Window → Samples → NvBlox → nvblox_sample_scene.usd`

For stereo-image-processing visualization failures, see
[troubleshooting](troubleshooting.md).
