# Plugins, projects, and migration

## QGIS 4 plugin metadata

### Declare a compatibility range

For QGIS 4 migration, plugin compatibility is determined by
`qgisMinimumVersion` and optional `qgisMaximumVersion`. Without a maximum,
support is assumed only through the end of the minimum version's major line.
To preserve QGIS 3.22 compatibility and join the QGIS 4 Ready list, declare:

```ini
[general]
qgisMinimumVersion=3.22
qgisMaximumVersion=4.99
```

The Ready list includes a plugin when either bound is at least 4.0.

### Remove `supportsQt6`

For QGIS 4 migration, `supportsQt6=True` has been removed from core and is no
longer recognized. It cannot advertise QGIS 4 compatibility; remove it and use
the QGIS version range.

### Complete the Qt 6 migration first

Before widening the compatibility range, replace Qt 5-only APIs and direct
`PyQt5` imports with Qt 6 equivalents, preferably imported through
`qgis.PyQt`, and test on QGIS 4.

Repository uploads run `pyqgis4-checker`. Its Qt6 Check tab identifies affected
files and lines, but findings do not block upload or approval. Treat the
report as migration guidance, not proof of runtime compatibility.

## Settings and project security

### Target isolated QGIS 4 settings

Since 4.2, QGIS 4 stores settings separately from QGIS 3. On first startup it
performs a one-time, lossless copy of the loaded QGIS 3 user profile.
Subsequent changes do not synchronize. Installation, profile-management, and
enterprise-deployment scripts must target the QGIS 4 location.

### Grant granular project trust

Since 4.0, projects store separate trust decisions for macros, expression
functions, actions, and attribute-form initialization code. A trust dialog can
preview code. Global policy can allow or deny execution by project or path.

## Application UI and theming

### Deliver themes from plugins

Since 4.0, plugins can ship themes and custom application styles. Installing a
plugin can therefore alter the application theme without a matching core
theme.

### Create menus and toolbars

Since 4.0, users can create their own menus and toolbars instead of only
customizing built-in UI.

Since 4.2, a Processing algorithm can be assigned to a user-defined menu or
toolbar. Triggering the action opens the algorithm parameter and execution
dialog.

## Project metadata and translation

### Localize metadata

Since 4.0, key project and layer metadata participates in project translation.
Translated values can feed layout labels, map decorations, and other metadata
consumers.
