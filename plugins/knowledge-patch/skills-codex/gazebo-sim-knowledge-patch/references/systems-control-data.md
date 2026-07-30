# Systems, Control, and Data

## Load statically registered plugins

System plugins can load from the static plugin registry (9.2.0), making
statically registered systems available through the normal Gazebo Sim system
loading path.

## Control motion and joints

- `DriveToPoseController` is available as a system plugin (9.2.0).
- `JointPositionController` accepts dynamic PID parameters (9.3.0), allowing
  PID configuration to change at runtime.
- `JointController` supports nested joints (10.0.0).
- `MecanumDrive` supplies mecanum-drive behavior with odometry and TF output
  (10.0.0).

## Publish pose, wind, and particle data

- The pose publisher can publish poses beyond top-level models (9.1.0).
- Empty poses are suppressed (10.1.0), so subscribers should not wait for
  empty placeholder entries.
- Wind can be published to Gazebo and ROS topics (9.2.0).
- `ParticleEmitter` can take its topic from SDF (10.1.0).

## Access sensors and detect nested models

- The public C++ `Link` API provides accessors for sensors associated with a
  link (9.1.0).
- `LogicalCamera` detects nested models (9.2.0).

## Attach semantics to entities

Use the `EntitySemantics` system to assign categories and tags to simulation
entities (10.0.0).

## Manage Entity Component Manager tracking

The Entity Component Manager exposes APIs that clear its internal tracking of
entity additions and removals (10.0.0). Use those APIs when a consumer has
finished processing the tracked changes.
