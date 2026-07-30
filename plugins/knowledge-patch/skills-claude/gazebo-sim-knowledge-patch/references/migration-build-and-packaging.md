# Migration, Build, and Packaging

## Custom GUI plugin migration

### Qt6 QML and plugin code

Gazebo GUI's move to `gz-gui10` uses Qt6, so custom Qt5 GUI plugins must be
ported. C++ objects exposed to QML are referenced with a leading underscore in
QML, without a matching rename on the C++ side (jetty-migration):

```qml
_MyClass.FunctionFoo()
```

Omit versions from Qt6 QML imports—for example, change
`import QtQuick.Dialogs 1.0` to `import QtQuick.Dialogs`. Review Qt6 migration
requirements for affected types, including `FileDialog` and `TreeView`.

`gz::gui::App()` may return a null `qGuiApp` pointer, so validate it before use.
Qt6 plugins should not manually call `QCoreApplication::processEvents()`.

## Removed configuration macros

The `config.hh` path and install-directory macros were removed. Resolve these
locations at runtime instead (jetty-migration):

| Removed macro | Replacement |
| --- | --- |
| `GZ_SIM_GUI_CONFIG_PATH` | `gz::sim::getGUIConfigPath()` |
| `GZ_SIM_SYSTEM_CONFIG_PATH` | `gz::sim::getSystemConfigPath()` |
| `GZ_SIM_SERVER_CONFIG_PATH` | `gz::sim::getServerConfigPath()` |
| `GZ_SIM_PLUGIN_INSTALL_DIR` | `gz::sim::getPluginInstallDir()` |
| `GZ_SIM_GUI_PLUGIN_INSTALL_DIR` | `gz::sim::getGUIPluginInstallDir()` |
| `GZ_SIM_WORLD_INSTALL_DIR` | `gz::sim::getWorldInstallDir()` |

## Component factory registration

The `std::string` overloads of `gz::sim::components::Factory::Register` and the
parameterless `Unregister()` were removed. Registration takes a C-string type
name and explicit registration-object ID; unregistration requires that ID too
(jetty-migration):

```cpp
Register(const char *_type, ComponentDescriptorBase *_compDesc,
         RegistrationObjectId _regObjId);
Unregister(RegistrationObjectId _regObjId);
```

## Commands and executable layout

- `gz model` is available as a standalone model executable (since 9.2.0).
- The `gz` tool moved away from Ruby-based CLI library loading to standalone
  applications. Package and debug command implementations as applications
  rather than dynamically loaded Ruby CLI libraries (jetty-highlights).
- `gz-sim-main` is relocatable, so it can run correctly when its runtime
  location differs from the original installation layout (since 10.1.0).

## Package naming

Gazebo's major version was removed from package names. Update package
dependencies to use unversioned names (jetty-highlights).

## Bazel support and migration

- Initial Bazel support covers the core Gazebo Sim library, not the GUI,
  physics, or systems (since 9.2.0).
- System targets became supported after the initial core-only work (since
  9.3.0). This does not imply GUI or physics coverage.
- Gazebo packages moved from legacy workspace-based Bazel configuration to
  Bzlmod. Jetty and Ionic library versions and required third-party packages
  are available through the Bazel Central Registry (jetty-highlights).
