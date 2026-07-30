# Collision Monitor

## Command pipeline and fail-safe timeouts

Collision Monitor normally receives desired commands on `cmd_vel_smoothed`
(`cmd_vel_raw` before Jazzy) and emits the safety-adjusted command on
`cmd_vel`.

If any observation source becomes stale for `source_timeout`, the monitor stops
the robot. The node default is `2.0` seconds; `0.0` disables the freshness
check, and a source-specific value overrides the node value.
`stop_pub_timeout`, default `1.0`, controls how long zero commands continue
afterward.

```yaml
cmd_vel_in_topic: cmd_vel_smoothed
cmd_vel_out_topic: cmd_vel
source_timeout: 2.0
stop_pub_timeout: 1.0
scan:
  source_timeout: 0.2
```

Set the timeout from the source's actual publication and transport latency.
Disabling it removes a fail-safe against a dead or disconnected sensor.

## Motion compensation and state output

`base_shift_correction` defaults to `true`. It compensates source points for
base motion between the observation timestamp and the current monitor cycle.
Disabling it saves processing, but is not recommended for fast robots using
modest-rate sensors.

Setting the otherwise-empty `state_topic` creates a publisher containing the
active polygon's name and action type.

```yaml
base_shift_correction: true
state_topic: collision_monitor_state
```

Use the state topic to correlate command limiting with the zone that caused it.

## Runtime control and debounce

The `Toggle` service and `ToggleCollisionMonitor` BT node can disable all
Collision Monitor polygons while leaving observation-source checking active.
This is not a complete sensor shutdown: stale-source checks still participate
in safety behavior.

Each polygon can set `trigger_consecutive_points` and
`release_consecutive_points` for temporal debounce. Values `1` and `1`
preserve single-cycle trigger and release behavior. Larger values trade
reaction or release latency for resistance to transient readings.

## Zone actions and geometry

Every name in `polygons` defines a zone with a geometry `type` and an
`action_type`:

- `stop` sets motion to zero.
- `slowdown` multiplies speed by `slowdown_ratio`.
- `limit` caps linear and angular speed.
- `approach` scales motion to maintain `time_before_collision`.

At least `min_points` observations must lie in the zone before it triggers.
When multiple zones trigger, the most restrictive action wins.

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

## Static and subscribed zone geometry

Polygon `points` is a string containing at least three `[x, y]` pairs. Circles
use `radius`.

If static geometry is omitted, `stop`, `slowdown`, and `limit` zones can read
`polygon_sub_topic`, which supplies polygon points or a circle radius. An
`approach` polygon can read `footprint_topic`. Static geometry wins if both
static and subscribed forms are configured.

`polygon_subscribe_transient_local` selects transient-local durability for a
subscribed zone:

```yaml
dynamic_stop:
  type: polygon
  polygon_sub_topic: safety_zone
  polygon_subscribe_transient_local: true
  action_type: stop
  min_points: 4
```

Use transient-local durability when a late-joining monitor must immediately
receive the last zone definition.

## Humble point-threshold migration

Humble's `max_points` represented the greatest safe point count. Newer
releases use `min_points`, the smallest count that triggers an action. Preserve
the boundary with:

```text
min_points = max_points + 1
```

Do not copy the numeric value unchanged; that shifts the trigger threshold by
one point.

## Velocity-dependent polygons

A `velocity_polygon` chooses the first named sub-polygon whose linear and
angular ranges include the current command. Order overlapping entries
deliberately, and end with a range covering every possible velocity so there
is a fallback.

For `holonomic: false`, `linear_min` and `linear_max` are signed x velocity,
with negative values representing reverse motion. For `holonomic: true`, they
are nonnegative resultant-speed magnitudes; the match may also be restricted
with `direction_start_angle` and `direction_end_angle`.

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

## Observation-source filtering

Point-cloud sources project only points between `min_height` and `max_height`
and beyond `min_range`. With `use_global_height: true`, height filtering uses
an existing global `height` field instead of transformed `z`.

Point-cloud sources also support `point_cloud_transport`; `transport_type`
defaults to `raw` and may select a configured compressed format such as
`zstd`, `zlib`, or `draco`.

Range sources synthesize arc points at `obstacles_angle` spacing, which defaults
to one degree. Polygon sources sample their boundaries at `sampling_distance`,
default `0.1`.

```yaml
observation_sources: [depth, sonar]
depth:
  type: pointcloud
  topic: camera/points
  transport_type: zstd
  min_height: 0.05
  max_height: 0.5
  min_range: 0.2
sonar:
  type: range
  topic: sonar
  obstacles_angle: 0.0174533
```

## Costmap sources

Collision Monitor and Collision Detector can consume `nav2_msgs/Costmap` via a
`CostmapSource`. Use it cautiously: costmap age, resolution, unknown-space
policy, and the robot's safety stopping distance all affect the result.

For an already configured costmap source, `cost_threshold` defaults to `253`,
so inscribed and lethal cells become obstacle points. Cost `255` is handled
separately by `treat_unknown_as_obstacle`, which defaults to `true`. Disable it
only when large unknown regions must not trigger the monitor.

```yaml
costmap:
  type: costmap
  topic: local_costmap/costmap
  cost_threshold: 254
  treat_unknown_as_obstacle: false
```
