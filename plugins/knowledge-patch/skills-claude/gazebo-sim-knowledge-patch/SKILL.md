---
name: gazebo-sim-knowledge-patch
description: Gazebo Sim
version: 10.1.1
license: MIT
metadata:
  author: Nevaberry
---


# Gazebo Sim Knowledge Patch

Use this skill when upgrading, integrating, configuring, or extending Gazebo Sim,
especially when work touches custom GUI plugins, system plugins, physics,
controllers, packaging, transport, or server lifecycle behavior. Check the
project's actual Gazebo release and apply only guidance available to that release.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration, build, and packaging](references/migration-build-and-packaging.md) | Qt6 plugin migration, removed macros, component registration, standalone applications, package names, Bazel, relocatable executables |
| [Physics, sensors, and environment](references/physics-sensors-and-environment.md) | Ray queries, bounds, inertia, pose publication, cameras, wind, airfoils, gravity, fixed constraints, environment visualization |
| [Systems, control, and modeling](references/systems-control-and-modeling.md) | Joint controllers, drive systems, wheel slip, entity semantics, detachable joints, reset behavior, particle emitters |
| [Server, runtime, and APIs](references/server-runtime-and-apis.md) | Process shutdown, server startup, Python execution, static registries, command results, ECM tracking, WebSockets, networking, API spelling |
| [GUI, transport, and ROS](references/gui-transport-and-ros.md) | Component Inspector, Zenoh selection, standard simulation interface, occupancy maps, wind topics, Windows Quick Start |

## Migration quick reference

### Port custom GUI plugins to Qt6

- Build custom plugins against the Qt6-based Gazebo GUI.
- Remove version numbers from Qt QML imports.
- Review Qt6 replacements for types such as `FileDialog` and `TreeView`.
- Prefix C++ objects exposed to QML with an underscore at the QML call site:

```qml
_MyClass.FunctionFoo()
```

- Do not rename the corresponding C++ object merely to match the QML prefix.
- Treat `gz::gui::App()` as nullable and check the returned pointer before use.
- Avoid manually calling `QCoreApplication::processEvents()` from the plugin.

### Replace removed install-path macros

Include and call the runtime path APIs instead of relying on `config.hh` macros:

| Removed macro | Runtime API |
| --- | --- |
| `GZ_SIM_GUI_CONFIG_PATH` | `gz::sim::getGUIConfigPath()` |
| `GZ_SIM_SYSTEM_CONFIG_PATH` | `gz::sim::getSystemConfigPath()` |
| `GZ_SIM_SERVER_CONFIG_PATH` | `gz::sim::getServerConfigPath()` |
| `GZ_SIM_PLUGIN_INSTALL_DIR` | `gz::sim::getPluginInstallDir()` |
| `GZ_SIM_GUI_PLUGIN_INSTALL_DIR` | `gz::sim::getGUIPluginInstallDir()` |
| `GZ_SIM_WORLD_INSTALL_DIR` | `gz::sim::getWorldInstallDir()` |

### Update component registration ownership

The component factory no longer accepts a `std::string` registration type or a
parameterless unregister operation. Use a C-string type name and associate both
registration and unregistration with the same `RegistrationObjectId`:

```cpp
Register(const char *_type, ComponentDescriptorBase *_compDesc,
         RegistrationObjectId _regObjId);
Unregister(RegistrationObjectId _regObjId);
```

Keep the registration-object ID alive and available for cleanup. This makes
unregistration explicit when multiple modules contribute registrations.

### Adjust packages, commands, and build files

- Depend on unversioned Gazebo package names.
- Package `gz` commands as standalone applications; do not assume the older
  Ruby library-loading path when locating or debugging a command.
- Use the standalone `gz model` executable for model operations.
- Prefer Bzlmod declarations for Gazebo Bazel consumers. Confirm that the
  selected release supports every requested target because early Bazel support
  covered the core library before it expanded to systems.
- Do not infer GUI or physics Bazel support from core-library support.
- Treat `gz-sim-main` as relocatable when the executable's runtime location
  differs from its original installation layout.

### Apply API and behavior corrections

- Rename calls from `systemTimeISO` to `systemTimeIso`.
- Consume the actual status returned by UserCommands services instead
  of assuming transport success means command success.
- Expect pose publication to omit empty pose entries.
- Do not encode the bad-SDF server regression as intended behavior; use the
  corrected invalid-SDF handling.

## High-value capability quick reference

### Query geometry and sensors

- Use the Physics system for supported ray-intersection queries.
- Obtain a link's axis-aligned bounding box from its collision geometry.
- Use the public C++ `Link` sensor accessors to access sensors associated with
  a link.
- Logical cameras can detect models nested below top-level models.
- Enable frustum visualization when camera-volume diagnostics are useful.

### Configure inertia and surface interaction

Mass can drive automatic inertia calculation:

```xml
<inertial auto="true">
  <mass>5.0</mass>
</inertial>
```

Gazebo derives density and inertial parameters from the mass. For mesh inertia,
the calculator is available even when the simulation originates from an SDF
string, and mesh-optimization parameters can tune that calculation.

Use `LookupWheelSlip` when surface texture should select material friction. Its
8-bit RGB lookup map maps colors to friction values, and its slip-map resource
is resolved with `common::findFile`.

### Select controllers and drive systems

- Use `DriveToPoseController` for drive-to-pose behavior.
- In `JointController` force mode, disable braking when unconditional braking
  conflicts with the control design.
- Change `JointPositionController` PID parameters dynamically when runtime
  tuning is required.
- Address nested joints directly through `JointController`.
- Use `MecanumDrive` when mecanum motion with odometry and TF output is needed.

### Model environment effects

- Configure the battery plugin's current-sign parameter to match the
  integration's charging and discharging convention.
- Use reversible-airfoil support in `LiftDrag` for aerodynamic configurations
  whose operating direction reverses.
- Expect wind-aware airspeed to follow the wind triangle.
- Publish wind data to Gazebo and ROS topics when external consumers need it.
- Expect IMU output to react to runtime gravity changes, including changes from
  the GUI or gravity-setting command.
- Use `EnvironmentPreload` visualization for static environments.

### Use entity and simulation lifecycle APIs

- Assign categories and tags with the `EntitySemantics` system.
- Enforce fixed constraints with the Physics system parameter when the model
  requires strict fixed-joint behavior.
- Clear Entity Component Manager addition/removal tracking through its public
  cleanup APIs when a consumer has finished processing a change set.
- Invoke simulation reset through the public API.
- Implement `ISystemReset` in reset-sensitive systems and exercise it through
  the reset-aware test fixture.
- Removing a model also cleans up its detachable joints.

### Operate distributed and hosted simulations

- Service managers and container runtimes can stop Gazebo Sim with `SIGTERM`
  and receive graceful shutdown behavior.
- Startup detects an already-running server; handle that condition rather than
  assuming every invocation creates a new server.
- Python systems and Python `TestFixture` calls use corrected GIL-release
  behavior; avoid workarounds based on the older behavior.
- Network secondaries create entities correctly in distributed simulations.
- The WebSocket server is owned by Gazebo Sim rather than `gz-launch`, and its
  protocol schema exposes top-level enums.

### Choose integration paths

Select Zenoh for a process by setting the transport implementation before
launch:

```sh
export GZ_TRANSPORT_IMPLEMENTATION=zenoh
```

Use the ROS community standard simulation interface when robot code must remain
portable between compatible simulators. To begin occupancy-grid exploration
from `/scan_image`, publish the start command:

```sh
gz topic -t /start_exploration -m gz.msgs.Boolean -p 'data: true'
```

## Verification checklist

Before considering a migration or feature complete:

1. Confirm the installed Gazebo release and package naming scheme.
2. Build every custom GUI plugin with Qt6 and exercise its QML dialogs and
   views.
3. Search for removed path macros, `systemTimeISO`, old component-factory
   overloads, and parameterless `Unregister()` calls.
4. Verify that Bazel targets used by the project are supported, especially GUI
   and physics targets.
5. Test startup with an existing server, shutdown through `SIGTERM`, reset, and
   invalid SDF input.
6. Validate controller behavior with braking, PID updates, nested joints, wind,
   and gravity changes that match the application's operating envelope.
7. Exercise transport and ROS integrations with the selected process
   environment and verify command-level service results.
8. Open the topic reference for any touched subsystem; the quick reference
   intentionally favors migration-sensitive and commonly used behavior.
