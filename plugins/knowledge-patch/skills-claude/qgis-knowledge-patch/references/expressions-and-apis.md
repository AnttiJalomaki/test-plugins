# Expressions and APIs

## String, CRS, and interpolation expressions

### Repeat and reverse strings

Since 3.44, `repeat` and `reverse` accept strings:

```qgis
repeat('ab', 3)
reverse('abc')
```

### Create and identify CRSs

Since 3.44, `crs_from_text` creates a CRS from an authority code, WKT, or PROJ
definition. `crs_to_authid` returns an `authority:id` value:

```qgis
crs_to_authid(crs_from_text('EPSG:4326'))
```

### Scale cubic Bézier values and join non-NULL text

Since 4.2, `scale_cubic_bezier` performs cubic Bézier interpolation and
supports conversion from MapBox `cubic-bezier` style values.
`concat_ws(separator, ...)` ignores NULL arguments:

```qgis
concat_ws('-', 'a', NULL, 'b')
```

The result is `a-b`.

## Magnetic and time-zone expressions

### Calculate magnetic-model values

Since 4.0, expressions include `magnetic_declination`,
`magnetic_inclination`, `magnetic_declination_rate_of_change`, and
`magnetic_inclination_rate_of_change`. They return angles or annual rates at a
point.

### Create, inspect, and change time zones

Since 4.0, `timezone_from_id`, `timezone_id`, and `get_timezone` create or
inspect IANA time zones. `convert_timezone` changes a datetime to the
equivalent time in another zone. `set_timezone` replaces the zone without
changing the date or time components.

## Geometry and data-frame APIs

### Access GEOS-specific behavior

Since 3.42, PyQGIS exposes `QgsGeos` directly, so Python code can use
GEOS-specific functions that are unavailable through the base
`QgsGeometryEngine`.

### Preserve dimensions in NumPy output

Since 3.42, `QgsGeometry.as_numpy()` preserves dimensionality. Geometries with
Z and/or M values return XYZ, XYM, or XYZM coordinates instead of always
returning XY.

### Calculate 3D area and collinearity

Since 4.0, `QgsGeometry.area3D()` returns surface area for polygons,
polyhedral surfaces, TINs, and collections; it returns zero for points and
lines. `QgsGeometryUtilsBase::pointsAreCollinear` handles both 2D and 3D
points, and `QgsGeometryUtilsBase::points3DAreCollinear` explicitly handles
3D points.

### Convert layers to GeoPandas

Since 4.0, `QgsVectorLayer.as_geopandas()` converts a layer and its attributes
to a GeoPandas dataframe when GeoPandas is installed.

## Application and plugin APIs

### Control GPS tools

Since 3.44, `iface.gpsTools()` exposes `QgsAppGpsTools`. Plugins can create a
feature from the current track:

```python
iface.gpsTools().createFeatureFromGpsTrack()
```

They can also set the track line symbol and update its geometry:

```python
iface.gpsTools().setGpsTrackLineSymbol(line_symbol)
```

### Extend 3D map tools

Since 4.0, plugins can subclass `Qgs3DMapTool`, apply the cross-section tool's
four clipping planes, and call `Qgs3DMapCanvas.castRay()` to obtain and manage
3D hits through `QgsRay3D`.

### Use SFCGAL without repeated conversion

Since 4.0, `QgsSfcgalEngine` provides native SFCGAL integration and
`QgsSfcgalGeometry` reduces conversion overhead around SFCGAL operations.
