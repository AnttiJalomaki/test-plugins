# Costmaps, Localization, and Mapping

## Costmap API and composition

`Costmap2DROS.map_topic` was removed. Configure the map topic on the
`StaticLayer`:

```yaml
static_layer:
  plugin: nav2_costmap_2d::StaticLayer
  map_topic: my_map
```

`Costmap2DROS` constructors consolidate to:

```cpp
Costmap2DROS(name, parent_namespace = "/", use_sim_time = false)
```

The local namespace is inferred from the node name. When a layer must resolve a
topic against the parent costmap rather than its private namespace, use
`joinWithParentNamespace()`.

The Plugin Container Layer can group selected costmap layers, combine them,
and then contribute that grouped result to the parent costmap. Use it where a
subset of layers needs an explicit composition boundary.

## Ordinary layers and filters

Names in `filters` are loaded as plugins but kept separate from `plugins`.
Filters run on top of the fully combined ordinary layered costmap, preventing
keepout, speed, or binary filters from interfering with ordinary layer
combination. Every listed name must have a `plugin` parameter in the matching
namespace.

```yaml
filters: [keepout_filter, speed_filter]
keepout_filter:
  plugin: nav2_costmap_2d::KeepoutFilter
speed_filter:
  plugin: nav2_costmap_2d::SpeedFilter
```

## Point-cloud transports

Obstacle and voxel costmap layers can consume compressed
`point_cloud_transport` streams. `transport_type` defaults to `raw` and can
select a configured format such as `zstd`, `zlib`, or `draco`.

```yaml
pointcloud:
  data_type: PointCloud2
  topic: /camera/points
  transport_type: zstd
```

Collision Monitor supports the same transport setting for point-cloud sources.
Confirm the producer and consumer have the requested transport plugin and
budget decompression latency into freshness timeouts.

## Costmap conversion values

`inscribed_obstacle_cost_value` defaults to `99`. It controls conversion of
`INSCRIBED_INFLATED_OBSTACLE` between `Costmap2D` and `OccupancyGrid`, avoiding
the earlier `253 → 99 → 251` mismatch. Components interpreting the converted
grid must use the configured conversion value rather than assuming the native
costmap constant.

## Selective clearing

Requests for `ClearEntireCostmap`, `ClearCostmapAroundRobot`,
`ClearCostmapAroundPose`, and `ClearCostmapExceptRegion` accept a `plugins`
list.

- An empty list preserves the prior behavior and clears everything.
- Named entries clear only loaded plugins that support clearing.
- Any invalid or non-clearable name fails the complete request without
  clearing anything.

Validate names before issuing an operational recovery request; partial success
is not possible when the list contains an invalid entry.

## Speed-zone anticipation

Speed Filter's `enable_path_lookahead` is disabled by default. When enabled, it
examines a velocity-dependent window along the planned path and applies the
strictest speed limit in that window, allowing deceleration before the robot
enters a restricted zone. Configure controller and smoother deceleration limits
that can actually meet the anticipated transition.

## Inflation-layer extensions

An asymmetric inflation field can bias the path-dependent Voronoi boundary for
keep-left or keep-right behavior.

`custom_inscribed_radius` defaults to `-1.0`, which retains the
footprint-derived radius. A nonnegative value overrides it. Values such as
`0.0` bypass the inscribed region and are unsafe for ordinary planners or
controllers unless they were explicitly designed for that cost
representation.

## Vector Object Server

`nav2_map_server` provides a Vector Object Server that rasterizes configured
circles, polygons, and polygonal chains into an `OccupancyGrid`. Its services
are:

- `AddShapes`
- `GetShapes`
- `RemoveShapes`

Use it for dynamic virtual obstacles, keepout areas, or speed-filter masks. The
result is a raster grid, so choose geometry and resolution with the downstream
costmap or filter semantics in mind.

## AMCL repeatability

`random_seed` controls the particle-filter RNG. Any nonnegative value makes
runs repeatable. The default `-1` seeds from the current time and retains
nondeterministic behavior.

```yaml
amcl:
  ros__parameters:
    random_seed: 42
```

Set a fixed seed in regression tests that compare convergence or pose output,
but retain time-based seeding when varied runs are desired.

## AMCL reset and map reload

By default, AMCL reuses its last known pose on reset and accepts replacement
maps. Set `always_reset_initial_pose: true` to require a new pose from the
initial-pose topic or from `initial_pose` with `set_initial_pose: true`.

Set `first_map_only: true` only when maps received later on `map_topic` must be
ignored.

```yaml
amcl:
  ros__parameters:
    always_reset_initial_pose: true
    set_initial_pose: true
    initial_pose: {x: 1.0, y: 2.0, z: 0.0, yaw: 0.5}
    first_map_only: false
```

## Timestamped dynamic footprints

`subscribe_to_stamped_footprint: true` changes a costmap footprint subscription
from `Polygon` to `PolygonStamped`. Each update can then carry its own timestamp
and frame.

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      subscribe_to_stamped_footprint: true
```

Update the publisher and subscription together; the topic name alone does not
signal the message-type change.

## Startup transform deadline

During configuration, a costmap waits `initial_transform_timeout` seconds for
the robot-base-to-global-frame transform. The default is `60.0`; expiry aborts
configuration.

```yaml
initial_transform_timeout: 60.0
```

Treat an expiry as a transform or launch-order failure, not merely slow costmap
data. Increasing the deadline can mask a transform that will never appear.

## Visualization height

`map_vis_z` offsets only the visualized costmap height; it does not alter
planning data. A small value can prevent coplanar RViz surfaces from
flickering:

```yaml
map_vis_z: -0.008
```
