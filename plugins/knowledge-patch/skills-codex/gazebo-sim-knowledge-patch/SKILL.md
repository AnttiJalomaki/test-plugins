---
name: gazebo-sim-knowledge-patch
description: Gazebo Sim
version: 10.1.1
license: MIT
metadata:
  author: Nevaberry
---


# Gazebo Sim Compatibility Guide

Use this skill when maintaining Gazebo Sim applications, plugins, systems,
worlds, build definitions, or integrations that may depend on recent Gazebo
behavior. Start with the migration notes for build or API failures, then open
the task-specific reference for complete details.

## How to use this skill

1. Determine the installed Gazebo Sim and Gazebo GUI versions from the
   project's package manifest, build configuration, or runtime environment.
2. For a migration, read the breaking-change checklist before changing code.
3. Use the index to open the reference matching the subsystem being changed.
4. Apply only behavior relevant to the installed version and its configured
   transport, physics engine, GUI, and systems.
5. Verify configuration-dependent behavior in a minimal world or focused
   system test.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration, build, and GUI](references/migration-build-gui.md) | Qt 6 porting, removed APIs and macros, CLI packaging, Bazel, GUI behavior, relocation |
| [Physics and environment](references/physics-environment.md) | Queries, inertia, forces, aerodynamics, wind, friction, gravity, constraints |
| [Systems, control, and data](references/systems-control-data.md) | Controllers, sensors, pose data, entity metadata, odometry, component tracking |
| [Runtime, integration, and testing](references/runtime-integration-testing.md) | Process lifecycle, transports, ROS, services, networking, reset, Python, WebSocket |

## Breaking migration checklist

### Port custom GUI plugins to Qt 6

- Build custom GUI plugins against the Qt 6 Gazebo GUI generation.
- Remove version numbers from Qt QML imports.
- Review `FileDialog` and `TreeView` usage for Qt 6 behavior.
- Prefix C++ objects exposed to QML with `_` at the QML call site; do not
  rename the corresponding C++ object solely for this change.
- Treat `gz::gui::App()` as nullable.
- Do not manually call `QCoreApplication::processEvents()` from Qt 6 plugins.

```qml
import QtQuick.Dialogs

_MyClass.FunctionFoo()
```

### Replace removed configuration macros

Use runtime path functions instead of including `config.hh` for installation
paths:

| Removed macro | Runtime function |
| --- | --- |
| `GZ_SIM_GUI_CONFIG_PATH` | `gz::sim::getGUIConfigPath()` |
| `GZ_SIM_SYSTEM_CONFIG_PATH` | `gz::sim::getSystemConfigPath()` |
| `GZ_SIM_SERVER_CONFIG_PATH` | `gz::sim::getServerConfigPath()` |
| `GZ_SIM_PLUGIN_INSTALL_DIR` | `gz::sim::getPluginInstallDir()` |
| `GZ_SIM_GUI_PLUGIN_INSTALL_DIR` | `gz::sim::getGUIPluginInstallDir()` |
| `GZ_SIM_WORLD_INSTALL_DIR` | `gz::sim::getWorldInstallDir()` |

### Update component registration

Component factory registration now needs a C-string type name and a
registration-object ID. Store the ID and pass it when unregistering; the
parameterless `Unregister()` path is gone.

```cpp
Register(const char *_type, ComponentDescriptorBase *_compDesc,
         RegistrationObjectId _regObjId);
Unregister(RegistrationObjectId _regObjId);
```

### Update names and packaging assumptions

- Replace `systemTimeISO` calls with `systemTimeIso`.
- Treat `gz` commands as standalone applications when packaging or debugging
  them; do not assume Ruby-based CLI library loading.
- Use unversioned Gazebo package names in dependencies.
- Use Bzlmod rather than legacy workspace-based Bazel setup for Jetty-era
  Gazebo packages.
- Do not assume an installed `gz-sim-main` remains at its original build or
  installation path.

## High-value physics features

### Query geometry and sensors

- Physics supports ray-intersection queries.
- A link axis-aligned bounding box can be derived from its collision geometry.
- The public C++ `Link` API exposes accessors for sensors attached to a link.
- Frustums can be visualized while inspecting a simulation.

### Configure inertia and surface behavior

Automatic inertia can use an explicit mass. Gazebo derives density and
inertial parameters when `inertial/@auto` is enabled:

```xml
<inertial auto="true">
  <mass>5.0</mass>
</inertial>
```

For spatially varying wheel slip, configure `LookupWheelSlip` with an 8-bit
RGB lookup image whose colors map to material-friction values. Remember that
the slip-map path follows Gazebo Common resource resolution.

### Account for changed dynamics

- Set the Physics fixed-constraint enforcement parameter when fixed
  constraints must be actively enforced.
- Configure `JointController` braking explicitly for force mode when braking
  is not desired.
- Reversible-airfoil models can use the corresponding `LiftDrag` support.
- Airspeed in wind is calculated with the wind triangle.
- IMU output responds to runtime gravity changes.

## High-value systems and control features

### Select a controller

- Use `DriveToPoseController` for drive-to-pose behavior.
- Change `JointPositionController` PID parameters dynamically when runtime
  tuning is required.
- `JointController` can address nested joints.
- Use `MecanumDrive` when the model needs mecanum control with odometry and TF
  output.

### Publish and consume simulation data

- Pose publication can include entities below top-level models.
- Pose publication suppresses empty poses; subscribers must not rely on empty
  placeholder entries.
- Wind can be published on Gazebo and ROS topics.
- `ParticleEmitter` can read its publication topic from SDF.
- Use `EntitySemantics` to assign entity categories and tags.

## Runtime and integration quick reference

### Choose transport and interfaces

Select Zenoh for a process by setting:

```sh
export GZ_TRANSPORT_IMPLEMENTATION=zenoh
```

Gazebo's ROS integration supports the community standard simulation
interface. Occupancy-grid exploration can be started through the scan-image
workflow with:

```sh
gz topic -t /start_exploration -m gz.msgs.Boolean -p 'data: true'
```

### Handle lifecycle and results

- Expect graceful shutdown on `SIGTERM` from a service manager or container.
- Check the actual result returned by UserCommands services.
- Use the public reset API for simulation reset.
- Implement `ISystemReset` in reset-sensitive test fixtures.
- An existing server is detected during startup.
- Invalid SDF must be handled according to the corrected server behavior, not
  the affected regression.

## Diagnostic routing

If a Qt plugin fails to load or QML names stop resolving, start with
[Migration, build, and GUI](references/migration-build-gui.md). Check the Qt
generation, unversioned imports, underscored QML object names, and nullable
application access.

If a world behaves differently under force, wind, gravity, or automatic
inertia, use [Physics and environment](references/physics-environment.md).
Check controller braking, wind-triangle airspeed, runtime gravity propagation,
and whether mass or density drives automatic inertia.

If entities, poses, controllers, or sensors appear missing, use
[Systems, control, and data](references/systems-control-data.md). Distinguish
pose suppression from missing data, check nested-model and nested-joint
support, and verify component tracking state.

If startup, shutdown, reset, Python execution, networking, service results, or
WebSocket schemas differ, use
[Runtime, integration, and testing](references/runtime-integration-testing.md).
Confirm the selected transport, server ownership, secondary-network role,
and reset contract before adding workarounds.
