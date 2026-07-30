# Physics, Sensors, and Environment

## Geometry queries and visualization

- The Physics system supports ray-intersection queries (since 9.1.0).
- A link's axis-aligned bounding box can be obtained from its collision
  geometry (since 9.2.0).
- Frustums can be visualized, which is useful when checking camera coverage
  (since 9.1.0).
- The public C++ `Link` API provides sensor accessors for sensors associated
  with a link (since 9.1.0).

## Cameras and pose publication

- The pose publisher is not limited to top-level models; it can publish poses
  below that scope (since 9.1.0).
- The `LogicalCamera` system detects nested models (since 9.2.0).
- Empty poses are suppressed rather than sent to pose subscribers (since
  10.1.0).

## Mesh and automatic inertia

`MeshInertialCalculator` is registered when a simulation is loaded from an SDF
string, so mesh-inertia calculation works through that load path. It also
accepts mesh-optimization parameters (since 9.2.0).

With `inertial/@auto` enabled, specify mass instead of density when that is the
known quantity. Gazebo derives density and inertial parameters
(jetty-highlights):

```xml
<inertial auto="true">
  <mass>5.0</mass>
</inertial>
```

## Aerodynamics, wind, and gravity

- `LiftDrag` supports reversible airfoils (since 9.3.0).
- Airspeed under wind influence uses the wind triangle. Validate expected
  airspeed against that model instead of treating wind as a scalar correction
  (since 9.3.0).
- The IMU system reacts to runtime gravity changes. Gravity updates made in the
  GUI or through the gravity-setting command propagate correctly (since
  10.1.0).

## Constraints and static environments

- The Physics system has a parameter for enforcing fixed constraints (since
  10.0.0).
- `EnvironmentPreload` can visualize static environments (since 10.1.0).
