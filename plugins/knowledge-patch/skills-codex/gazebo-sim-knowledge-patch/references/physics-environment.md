# Physics and Environment

## Geometry queries and cleanup

- The Physics system supports ray-intersection queries (9.1.0).
- A link's axis-aligned bounding box can be obtained from its collision
  geometry (9.2.0).
- Removing a model also removes its detachable joints (9.1.0); do not add
  cleanup logic based on the earlier orphaned-joint behavior.

## Calculate inertia from meshes or mass

- `MeshInertialCalculator` is registered when a simulation is loaded from an
  SDF string (9.2.0), so that loading path supports mesh-inertia calculation.
- `MeshInertialCalculator` accepts mesh-optimization parameters (9.2.0).
- With `inertial/@auto` enabled, an SDF object may specify mass instead of
  density (`jetty-highlights`). Gazebo derives density and inertial
  parameters.

```xml
<inertial auto="true">
  <mass>5.0</mass>
</inertial>
```

## Configure joints and constraints

- `JointController` can disable braking in force mode (9.3.0). Configure this
  when a force-controlled joint must operate without automatic braking.
- The Physics system has a parameter for enforcing fixed constraints
  (10.0.0).

## Model aerodynamics and wind

- `LiftDrag` supports reversible airfoils (9.3.0).
- Airspeed under wind is calculated with the wind triangle (9.3.0). Expect
  wind-aware results rather than a calculation that ignores that geometry.
- Wind information can be published to Gazebo and ROS topics (9.2.0).

## Configure battery-current convention

The battery plugin accepts a parameter that adjusts the current sign (9.1.0).
Set it to match the sign convention expected by the consuming integration.

## Vary wheel slip by surface texture

`LookupWheelSlip` maps colors from an 8-bit RGB lookup image to
material-friction values (`jetty-highlights`), allowing friction to vary over
a surface. Its slip-map lookup uses `common::findFile` (10.0.0), so configure
the resource path according to Gazebo Common file resolution.

## React to runtime environment changes

- The IMU system reacts to gravity changes (10.1.0). Gravity updates made
  through both the GUI and the gravity-setting command propagate correctly.
- `EnvironmentPreload` can visualize static environments (10.1.0).
