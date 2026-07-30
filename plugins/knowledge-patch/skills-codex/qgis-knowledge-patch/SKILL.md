---
name: qgis-knowledge-patch
description: QGIS
version: 4.2
license: MIT
metadata:
  author: Nevaberry
---



# QGIS Knowledge Patch

## When to load this skill

Load this skill for QGIS desktop, QGIS Server, Processing, PyQGIS, plugin, or
provider work involving:

- QGIS 4 plugin compatibility, Qt 6 migration, profile locations, or project
  trust;
- labeling, symbology, layouts, elevation profiles, 3D scenes, meshes, or
  point clouds;
- Processing algorithms whose parameters, outputs, dependency gates, or
  replacements affect a workflow;
- PostgreSQL, WMS, WFS, OAPIF, STAC, SensorThings, authentication, or server
  deployment behavior.

Inspect the project's QGIS version and any GDAL, PDAL, SFCGAL, GEOS, or wrench
requirements before applying version-dependent guidance. Prefer project
metadata, provider capabilities, runtime checks, and focused tests over
assumptions.

## Working method

1. Identify whether the task targets desktop, server, Processing, a provider,
   or a plugin, and whether it runs through the GUI, C++, or PyQGIS.
2. Read project and plugin metadata, then check the actual QGIS and dependency
   versions.
3. Separate persistent project/profile state from per-layer, per-layout, and
   per-connection settings.
4. Treat defaults and replacement algorithms as migration-sensitive.
5. Verify output geometry, dimensions, CRS, fields, NoData, and temporary-layer
   behavior with representative data.

## Reference index

| Reference | Topics |
| --- | --- |
| [plugins-api-and-user-interface.md](references/plugins-api-and-user-interface.md) | QGIS 4 plugin metadata, Qt 6, settings, trust, PyQGIS APIs, forms, menus, themes |
| [cartography-labeling-layouts-and-profiles.md](references/cartography-labeling-layouts-and-profiles.md) | Labels, symbols, layouts, charts, legends, expressions, elevation profiles |
| [three-d-mesh-and-point-cloud.md](references/three-d-mesh-and-point-cloud.md) | 3D scenes, materials, tiled scenes, mesh editing, VPC, COPC, point-cloud tools |
| [processing-geometry-and-raster.md](references/processing-geometry-and-raster.md) | Processing, geometry validation, digitizing, raster operations, networks, metadata |
| [data-sources-databases-and-server.md](references/data-sources-databases-and-server.md) | PostgreSQL, WMS/WFS/OAPIF, STAC, authentication, Browser, QGIS Server |

## Breaking changes and migration priorities

### Advertise QGIS 4 compatibility with version bounds

Plugin compatibility is determined by `qgisMinimumVersion` and the optional
`qgisMaximumVersion`. If the maximum is absent, compatibility is assumed only
through the end of the minimum version's major line.

To retain QGIS 3.22 support while advertising QGIS 4 compatibility:

```ini
[general]
qgisMinimumVersion=3.22
qgisMaximumVersion=4.99
```

A Ready-list entry requires at least one bound to be 4.0 or later.
`supportsQt6=True` is obsolete and ignored; remove it.

Before widening the range, replace Qt 5-only APIs and direct `PyQt5` imports,
prefer `qgis.PyQt`, run the plugin on QGIS 4, and inspect
`pyqgis4-checker` results. The checker reports affected files and lines but
does not block repository upload or approval.

### Target QGIS 4 settings explicitly

QGIS 4 settings are isolated from QGIS 3. First startup makes a one-time,
lossless copy of the loaded QGIS 3 profile, but later changes do not
synchronize. Installation, profile-management, and enterprise scripts must
write the QGIS 4 location.

### Update changed server and layout behavior

The default QGIS Server OAPIF root is `/ogcapi`, replacing `/wfs3`. Set
`QGIS_SERVER_API_WFS3_ROOT_PATH` when a deployment must retain another path.

Layout legends use All Project Layers, Visible Layers, or Manual Layer
Selection instead of the former Auto update checkbox. New legends default to
Visible Layers; a global layout option restores the previous default. Visible
Layers follows layer-tree visibility and structure but does not filter by map
extent.

### Replace deprecated hub-distance algorithms

The C++ Hub Distance algorithm replaces Distance to Nearest Hub (Points) and
Distance to Nearest Hub (Line to Hub), and can produce both optional outputs.
Migrate models and scripts away from the two deprecated algorithms.

## High-use guidance

### Keep plugin execution behind project trust

Projects carry separate trust decisions for macros, expression functions,
actions, and attribute-form initialization code. Use the trust preview and
global project/path policy; do not assume that embedded Python can execute.

Plugins can ship application themes and styles and can expose Processing
algorithms through user-defined menus and toolbars. Treat these as application
integration surfaces, not merely project-layer behavior.

### Preserve geometry dimensions

`QgsGeometry.as_numpy()` preserves Z and M coordinates, returning XYZ, XYM, or
XYZM where appropriate. `QgsGeometry.area3D()` calculates polygonal surface
area and returns zero for points and lines. When GeoPandas is installed,
`QgsVectorLayer.as_geopandas()` transfers geometry and attributes to a
GeoPandas dataframe.

Use `QgsGeos` when GEOS-specific methods are required in PyQGIS. For native
SFCGAL work, use `QgsSfcgalEngine` or the conversion-reducing
`QgsSfcgalGeometry` wrapper and remember that Approximate Medial Axis operates
on a 2D projection.

### Make dependency-gated outputs explicit

- Request COG with `-of COG`; a `.tif` suffix cannot distinguish COG from
  GTiff. COG export controls require GDAL 3.13 or later.
- GDAL Data Identification also requires GDAL 3.13 or later.
- GeoPackage field-domain updates and deletion require GDAL 3.12 or later.
- M3C2 requires a build with PDAL later than 2.10.
- point-cloud TIN maximum-edge filtering requires PDAL 2.6+ and wrench
  1.2.2+.
- Approximate Medial Axis endpoint extension requires SFCGAL 2.3.

### Distinguish temporary output from display name

Processing outputs may remain temporary while using a chosen layer name. The
memory-chip icon, rather than the name, identifies temporary results. Batch
Processing also accepts temporary outputs.

### Preserve provider and server intent

WFS connections can select `default`, `SimpleFeatures`, or `ComplexFeatures`
through `featureMode` and can use POST instead of GET. OAPIF connections can
choose advertised formats instead of assuming GeoJSON. WMS connections retain
their selected advertised image format.

OAuth2 connections refresh tokens while in use; cleanup and layer removal stop
refresh for unused connections. Extra token-endpoint values may be attached as
HTTP request headers. Planetary Computer authentication is stored in layer
data-source URIs and works across STAC, GDAL, and point-cloud consumers.

### Validate scale and temporal semantics

Project scale may be calculated at the map top, bottom, middle, horizontal
average, or equator. The setting affects displayed and API scale, scale-based
visibility, layouts, Processing renders, and server renders, but not map-unit
symbol sizes.

Temporal raster pixels can accumulate across frames. WMS-T groups can derive
time dimensions recursively from children, while disabling propagation on a
group prevents child dimensions from reaching its parent.

### Check legend and PDF layer-tree behavior

Per-layer automatic layout-legend inclusion is enabled by default. Geospatial
PDF export can preserve project groups, order, names, nesting, visibility, and
group layers when the layout map has no locked layers. It exports visible and
invisible project layers, has all-or-none attribute export, and does not
support mutually exclusive groups.

## Verification checklist

- Confirm QGIS, Qt, Python, GDAL, PDAL, SFCGAL, GEOS, and provider versions
  relevant to the task.
- Check plugin version bounds, imports, checker findings, QGIS 4 runtime
  behavior, and project trust.
- Exercise 2D and 3D views separately; verify CRS, Z/M dimensions, camera,
  terrain, and scene-export limitations.
- Test Processing outputs for geometry type, fields, removed/error reports,
  NoData, CRS, names, and temporary state.
- Verify label collisions, curved-text limitations, legend mode, atlas
  filtering, and layer-tree PDF structure.
- Test database permissions, schema restrictions, stored-query location,
  request method, advertised feature/image format, and server environment
  variables.
