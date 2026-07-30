# Collision Monitor

Source batch attribution: `overview-and-distro-migrations`,
`collision-monitor`.

## Command pipeline and fail-safe timeouts

Collision Monitor normally receives desired commands on `cmd_vel_smoothed`
(`cmd_vel_raw` before Jazzy) and publishes safety-adjusted commands on
`cmd_vel`.

If an observation source is stale for `source_timeout`, the monitor stops the
robot. The node default is `2.0` seconds; `0.0` disables the check, and a
source-specific value overrides the node setting. `stop_pub_timeout`, default
`1.0`, controls how long zero commands continue afterward.

```yaml
cmd_vel_in_topic: cmd_vel_smoothed
cmd_vel_out_topic: cmd_vel
source_timeout: 2.0
stop_pub_timeout: 1.0
scan:
  source_timeout: 0.2
```

## Runtime control and state

The `Toggle` service and `ToggleCollisionMonitor` BT node can disable all
Collision Monitor polygons while sensor checking remains active.

`base_shift_correction` defaults to true and compensates sensor points for
base motion between the observation timestamp and the current cycle.
Disabling it saves processing time but is not recommended for fast robots at
modest sensor rates. Set the otherwise-empty `state_topic` to create a
publisher that reports the active polygon name and action type.

```yaml
base_shift_correction: true
state_topic: collision_monitor_state
```

Polygon `trigger_consecutive_points` and `release_consecutive_points` add
temporal debounce. Values of `1` and `1` retain single-cycle trigger and
release behavior.

## Zone actions

Each name in `polygons` identifies a zone with a `type` and `action_type`:

- `stop` zeros motion;
- `slowdown` multiplies speed by `slowdown_ratio`;
- `limit` caps linear and angular speed;
- `approach` scales motion to preserve `time_before_collision`.

At least `min_points` readings must fall inside a zone. Simultaneous active
zones resolve to the most restrictive action.

```yaml
polygons: [stop_zone, approach_zone]
stop_zone:
  type: circle
  radius: 0.3
  action_type: stop
  min_points: 4
approach_zone:
  type: polygon
  action_type: approach
  footprint_topic: local_costmap/published_footprint
  time_before_collision: 2.0
  simulation_time_step: 0.1
```

## Static and subscribed geometry

Polygon `points` is a string containing at least three `[x, y]` pairs;
circles use `radius`. When static geometry is omitted, stop, slowdown, and
limit zones may consume `polygon_sub_topic`, carrying polygon points or a
circle radius. An approach polygon may instead consume `footprint_topic`.
Static geometry wins when both forms are configured.
`polygon_subscribe_transient_local` selects transient-local durability.

```yaml
dynamic_stop:
  type: polygon
  polygon_sub_topic: safety_zone
  polygon_subscribe_transient_local: true
  action_type: stop
  min_points: 4
```

Humble's `max_points` represented the largest safe count. Newer releases use
`min_points`, the smallest count that triggers a zone. Preserve the threshold
with `min_points = max_points + 1`.

## Velocity-dependent zones

A `velocity_polygon` uses the first named sub-polygon whose linear and angular
ranges contain the current command. Order overlaps deliberately and end with
a fallback range that covers every velocity.

For non-holonomic robots, `linear_min` and `linear_max` are signed x velocity,
with reverse values negative. With `holonomic: true`, they are nonnegative
resultant-speed magnitudes and may also be constrained by
`direction_start_angle` and `direction_end_angle`.

```yaml
velocity_stop:
  type: velocity_polygon
  action_type: stop
  min_points: 4
  holonomic: false
  velocity_polygons: [forward, fallback]
  forward:
    points: "[[0.5, 0.3], [0.5, -0.3], [-0.2, -0.3], [-0.2, 0.3]]"
    linear_min: 0.0
    linear_max: 1.0
    theta_min: -1.0
    theta_max: 1.0
  fallback:
    points: "[[0.3, 0.3], [0.3, -0.3], [-0.3, -0.3], [-0.3, 0.3]]"
    linear_min: -1.0
    linear_max: 1.0
    theta_min: -1.0
    theta_max: 1.0
```

## Observation sources

Point-cloud sources keep points between `min_height` and `max_height` and
beyond `min_range`. `use_global_height: true` filters an existing global
`height` field instead of transformed `z`. Point-cloud sources may also use
compressed `point_cloud_transport` through `transport_type`.

Range sources synthesize arc points at `obstacles_angle` spacing, default one
degree. Polygon sources sample their boundary at `sampling_distance` spacing,
default `0.1`.

```yaml
observation_sources: [depth, sonar]
depth:
  type: pointcloud
  topic: camera/points
  min_height: 0.05
  max_height: 0.5
  min_range: 0.2
sonar:
  type: range
  topic: sonar
  obstacles_angle: 0.0174533
```

Collision Monitor and Collision Detector can consume `nav2_msgs/Costmap`
through `CostmapSource`. Use this source cautiously because its cell policy
directly becomes collision evidence. For an already configured costmap
source, `cost_threshold` defaults to `253`, so inscribed and lethal cells
become points. Cost `255` is governed separately by
`treat_unknown_as_obstacle`, which defaults to true. Disable it only when
large unknown regions should not trigger the monitor.

```yaml
costmap:
  type: costmap
  topic: local_costmap/costmap
  cost_threshold: 254
  treat_unknown_as_obstacle: false
```
