# Migration, Build, and GUI

## Port custom GUI plugins

- Gazebo GUI's Jetty generation, `gz-gui10`, moves from Qt 5 to Qt 6
  (`jetty-migration`). Port custom Qt 5 GUI plugins rather than relying on Qt 5
  compatibility.
- C++ objects exposed to QML are referenced with a leading underscore
  (`jetty-migration`). Change `_MyClass.FunctionFoo()` at the QML call site;
  the C++ object itself does not need a matching rename.
- Qt 6 QML imports omit version numbers (`jetty-migration`). For example,
  replace `import QtQuick.Dialogs 1.0` with `import QtQuick.Dialogs`. Migrate
  `FileDialog` and `TreeView` uses for their Qt 6 APIs and behavior.
- `gz::gui::App()` may return a null `qGuiApp` pointer
  (`jetty-migration`). Validate the result before dereferencing it.
- Qt 6 plugins should not manually invoke
  `QCoreApplication::processEvents()` (`jetty-migration`).

```qml
import QtQuick.Dialogs

_MyClass.FunctionFoo()
```

## Replace removed C++ APIs and macros

### Resolve installation paths at runtime

The `config.hh` path and its install-directory macros are removed
(`jetty-migration`). Replace them with runtime functions:

| Removed macro | Replacement |
| --- | --- |
| `GZ_SIM_GUI_CONFIG_PATH` | `gz::sim::getGUIConfigPath()` |
| `GZ_SIM_SYSTEM_CONFIG_PATH` | `gz::sim::getSystemConfigPath()` |
| `GZ_SIM_SERVER_CONFIG_PATH` | `gz::sim::getServerConfigPath()` |
| `GZ_SIM_PLUGIN_INSTALL_DIR` | `gz::sim::getPluginInstallDir()` |
| `GZ_SIM_GUI_PLUGIN_INSTALL_DIR` | `gz::sim::getGUIPluginInstallDir()` |
| `GZ_SIM_WORLD_INSTALL_DIR` | `gz::sim::getWorldInstallDir()` |

### Register components with an owner ID

The `std::string` overloads of
`gz::sim::components::Factory::Register` are removed, as is parameterless
`Unregister()` (`jetty-migration`). Registration takes a C-string type name
and an explicit `RegistrationObjectId`; unregistration takes the same kind of
ID.

```cpp
Register(const char *_type, ComponentDescriptorBase *_compDesc,
         RegistrationObjectId _regObjId);
Unregister(RegistrationObjectId _regObjId);
```

### Correct the time API spelling

Replace `systemTimeISO` with `systemTimeIso` (10.0.0).

## Update command and package layouts

- The `gz` tool uses standalone applications instead of Ruby-based CLI
  library loading (`jetty-highlights`). Package and debug command
  implementations as standalone applications.
- Gazebo package names no longer carry major version numbers
  (`jetty-highlights`). Update dependency declarations to the unversioned
  package names.
- A standalone `gz model` executable is available (9.2.0).
- `gz-sim-main` is relocatable (10.1.0). Installations can run it from a
  location different from the original layout.

## Choose the supported Bazel shape

- The first Bazel build covers the core Gazebo Sim library but excludes GUI,
  physics, and systems targets (9.2.0).
- Bazel support later extends to system targets (9.3.0).
- Gazebo packages migrate from legacy workspace-based Bazel configuration to
  Bzlmod (`jetty-highlights`). Jetty and Ionic library versions, together with
  required third-party packages, are available through the Bazel Central
  Registry.

## GUI behavior to account for

- Pose attributes can again be plotted from the Component Inspector (9.2.0).
- Gazebo Sim can visualize frustums (9.1.0).
- The Quick Start dialog is disabled on Windows (10.1.0).
