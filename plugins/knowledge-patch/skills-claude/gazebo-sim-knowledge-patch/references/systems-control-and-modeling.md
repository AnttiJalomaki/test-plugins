# Systems, Control, and Modeling

## Joint control

- `JointController` can disable braking in force mode. Use this option where
  the force controller must coast or where automatic braking conflicts with
  the intended dynamics (since 9.3.0).
- `JointPositionController` accepts dynamic PID parameters, enabling runtime
  tuning rather than fixed startup-only gains (since 9.3.0).
- `JointController` supports joints in nested models (since 10.0.0).

## Drive systems

- `DriveToPoseController` is a system plugin for driving to a target pose
  (since 9.2.0).
- `MecanumDrive` supplies mecanum drive behavior with odometry and TF output
  (since 10.0.0).

## Wheel slip from textures

`LookupWheelSlip` uses an 8-bit RGB lookup map to translate surface texture
colors into material-friction values, so friction can vary spatially across a
surface (jetty-highlights).

Slip-map paths are resolved through `common::findFile`; account for that
resource-resolution behavior when configuring the map (since 10.0.0).

## Entity semantics and cleanup

- `EntitySemantics` assigns categories and tags to simulation entities (since
  10.0.0).
- Removing a model removes its detachable joints as well. Do not retain logic
  that expects those joints to remain after model deletion (since 9.1.0).

## Battery convention

The battery plugin accepts a parameter that adjusts the sign of current. Set it
to the charging/discharging convention expected by downstream integrations
(since 9.1.0).

## Simulation reset

- Simulation reset is exposed through a public callable API (since 10.0.0).
- The test fixture supports `ISystemReset`, allowing reset-sensitive system
  behavior to be exercised in tests (since 10.0.0).

## Particle emitters

`ParticleEmitter` can read its topic from SDF, allowing the publisher topic to
be selected per model or world configuration (since 10.1.0).
