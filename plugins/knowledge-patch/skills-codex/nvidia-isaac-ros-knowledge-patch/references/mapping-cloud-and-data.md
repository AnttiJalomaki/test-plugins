# Mapping, cloud, and data

## Resolve current mapping and localization packages

The current mapping and localization surface comprises:

- `isaac_ros_visual_global_localization`
- `isaac_mapping_ros`
- `isaac_ros_visual_mapping`
- `isaac_ros_occupancy_grid_localizer`
- `isaac_ros_pointcloud_utils`

Resolve older launch files against the current package index. The supported
set is release-dependent, and 4.4 included package renames. Do not mechanically
preserve an older package name merely because its launch arguments still look
familiar.

Related sensor capabilities include RGB-D camera support in
`isaac_ros_visual_slam` and lidar dynamics plus motion compensation in
`isaac_ros_nvblox` (4.1.0).

## Build Cloud Control task handling

The current Cloud Control surface includes:

- `isaac_ros_scene_recorder`
- `isaac_ros_vda5050_client`
- `vda5050_action_handler`

Use these packages for receiving fleet tasks and actions and for reporting
progress, state, and errors. Preserve all three reporting dimensions when
designing orchestration and observability around the client and action
handler.

## Convert MCAP sessions to LeRobot

`isaac_ros_data_tools` adds an MCAP-to-LeRobot converter in 4.5.0. It supports:

- Multi-session conversion.
- FPS resampling.
- `action.effort` export.

Decide the target FPS and session grouping before conversion, and retain
`action.effort` when downstream training or evaluation needs effort signals.

## Distinguish data from visualization failures

The Nvblox RealSense people-segmentation example can display an incorrect RViz
color overlay. The Isaac Sim stereo workflow can fail to display a point cloud
because of `PointCloud2` conversion or frame metadata. These visualization
symptoms do not, by themselves, prove that the underlying mapped or converted
data is absent.

