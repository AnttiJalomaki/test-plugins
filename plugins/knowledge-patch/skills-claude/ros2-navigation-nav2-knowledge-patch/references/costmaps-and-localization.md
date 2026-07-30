# Costmaps and localization

Source batch attribution: `overview-and-distro-migrations`,
`costmaps-and-localization`.

## Costmap API and composition

`Costmap2DROS.map_topic` was removed. Configure `map_topic` on
`StaticLayer`:

```yaml
static_layer:
  plugin: "nav2_costmap_2d::StaticLayer"
  map_topic: my_map
```

Constructors consolidate to
`Costmap2DROS(name, parent_namespace = "/", use_sim_time = false)`. The local
namespace is inferred from the node name. The Plugin Container Layer can group
selected layers and then combine their result with the parent costmap.

## Layer and filter ordering

Entries in `filters` are loaded as plugins but remain separate from ordinary
`plugins`. Filters run on top of the already combined layered costmap, keeping
keepout, speed, and binary filters from interfering with ordinary layers. Each
listed filter name needs a `plugin` parameter in the same namespace.

```yaml
filters: [keepout_filter, speed_filter]
keepout_filter:
  plugin: "nav2_costmap_2d::KeepoutFilter"
speed_filter:
  plugin: "nav2_costmap_2d::SpeedFilter"
```

Speed Filter's `enable_path_lookahead` defaults to false. When enabled, it
examines a velocity-dependent window along the planned path and applies the
strictest speed limit early enough to decelerate before entering a speed zone.

## Clearing selected plugins

Requests to `ClearEntireCostmap`, `ClearCostmapAroundRobot`,
`ClearCostmapAroundPose`, and `ClearCostmapExceptRegion` have a `plugins`
list. An empty list clears everything as before. Named entries clear only
loaded, clearable plugins. If any name is invalid or not clearable, the entire
request fails and nothing is cleared.

## Point-cloud transport

Obstacle and voxel layers can consume compressed `point_cloud_transport`
streams. `transport_type` defaults to `raw`; supported selections may include
`zstd`, `zlib`, or `draco`. Collision Monitor supports the same transport
setting for point-cloud sources.

```yaml
pointcloud:
  data_type: "PointCloud2"
  topic: /camera/points
  transport_type: "zstd"
```

## Cost conversion and inflation

`inscribed_obstacle_cost_value` defaults to `99`. It controls conversion of
`INSCRIBED_INFLATED_OBSTACLE` between `Costmap2D` and `OccupancyGrid`, avoiding
the former 253→99→251 mismatch.

Inflation Layer can create an asymmetric inflation field to bias a
path-dependent Voronoi boundary for keep-left or keep-right behavior.
`custom_inscribed_radius` defaults to `-1.0` and can override the
footprint-derived radius. Values such as `0.0` bypass the inscribed region and
are unsafe for ordinary planners and controllers unless they are explicitly
designed for that cost representation.

## AMCL repeatability and reset policy

AMCL's `random_seed` controls the particle-filter RNG. Any nonnegative value
makes runs repeatable; the default `-1` seeds from current time and preserves
nondeterministic behavior.

```yaml
amcl:
  ros__parameters:
    random_seed: 42
```

AMCL normally reuses its last known pose after reset and accepts replacement
maps. Set `always_reset_initial_pose: true` to require a fresh pose from the
initial-pose topic or from `initial_pose` with `set_initial_pose: true`.
Set `first_map_only: true` only when subsequent maps on `map_topic` must be
ignored.

```yaml
amcl:
  ros__parameters:
    always_reset_initial_pose: true
    set_initial_pose: true
    initial_pose: {x: 1.0, y: 2.0, z: 0.0, yaw: 0.5}
    first_map_only: false
```

## Footprints, transforms, and visualization

`subscribe_to_stamped_footprint: true` changes a costmap's footprint
subscription from `Polygon` to `PolygonStamped`, allowing every dynamic update
to provide its timestamp and frame.

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      subscribe_to_stamped_footprint: true
```

During configuration, a costmap waits `initial_transform_timeout` seconds for
the robot-base-to-global-frame transform. The default is `60.0`; expiry aborts
configuration.

```yaml
initial_transform_timeout: 60.0
```

`map_vis_z` offsets the visualized costmap vertically without changing
planning data. A small value such as `-0.008` can stop coplanar RViz displays
from flickering.

```yaml
map_vis_z: -0.008
```
