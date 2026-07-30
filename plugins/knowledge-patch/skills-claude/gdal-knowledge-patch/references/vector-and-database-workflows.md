# Vector and database workflows

Use this reference for the task areas below. Batch labels identify when each behavior entered the covered compatibility history.

## Geometry, CRS, and spatial filtering

### Reprojection in Arrow streams from warped layers

*Batch: 3.10.1*

`OGRWarpedLayer` no longer takes its Arrow stream directly from the source layer, because doing so skipped reprojection. Arrow-stream reads from a warped layer therefore retain the layer's reprojection behavior.

### FlatGeobuf output without a spatial index

*Batch: 3.10.1*

With `SPATIAL_INDEX=NO`, the FlatGeobuf writer accepts a dataset with no features and handles empty geometries as null geometries.

### GML and AIXM geometry handling

*Batch: 3.10.1*

The GML driver supports AIXM `ElevatedCurve` and honors `SWAP_COORDINATES=YES` even when a geometry has no spatial reference system.

### Geodesic lengths for open line strings

*Batch: 3.10.2*

`GeodesicLength()` works on non-closed line strings again; a regression in 3.10.0 had limited it to closed line strings.

### GeoJSON detection when geometry comes first

*Batch: 3.10.3*

The GeoJSON driver now detects features whose object starts with a `geometry` member, including input beginning `{"geometry":{"type":...,"coordinates":...`.

### Multipolygon results from edge-built polygons

*Batch: 3.11.5*

`OGRBuildPolygonFromEdges()` returns a multipolygon when the edges require one, including for affected DXF `HATCH` geometries. Callers must therefore be prepared for a multipolygon result.

### Three-dimensional geometry validation

*Batch: 3.12.1*

`gdal vector make-valid` no longer skips 3D geometries.

### Geometry-check result fields

*Batch: 3.12.1*

`gdal vector check-geometry` adds the `--include-field` option for including a field in its results.

### GML 3D geometry discovery

*Batch: 3.12.1*

The GML reader accepts 3D geometries whose `srsName` is three-dimensional even without `srsDimension='3'`. When several geometry elements exist and the last one is consistently selected, that geometry column now receives a name.

### Degenerate line geometries in MIF

*Batch: 3.12.2*

The MITAB `.mif` reader now accepts line strings and multi-line strings containing one point or no points.

### Complex OSM multipolygons

*Batch: 3.12.2*

The OSM driver again reads complex multipolygons correctly, fixing a regression introduced in 3.11.5.

### Closed polygons after polar reprojection

*Batch: 3.12.4*

`OGRGeometryFactory::transformWithOptions()` closes polygons produced by its polar-reprojection path, including when used with GEOS 3.15.

### CSV geometry-option routing

*Batch: 3.13.1*

For CSV output, `GEOMETRY=AS_WKT` supplied as a layer creation option is also set as a dataset creation option.

### C geometry growth through `OGR_G_SetPoint()`

*Batch: 3.13.2*

`OGR_G_SetPoint()` can again grow a geometry when the supplied point index is greater than or equal to its current point count.

## Databases and network services

### OCI time-zone timestamp creation

*Batch: 3.10.1*

The OCI driver adds a `TIMESTAMP_WITH_TIME_ZONE` layer creation option, with corresponding `ogr2ogr` handling.

### SQLite 3.49.1 with double-quoted strings disabled

*Batch: 3.10.3*

The SQLite SQL dialect and the GeoPackage driver now work with SQLite 3.49.1 built or configured with `SQLITE_DQS=0`.

### MSSQLSpatial metadata tables in `dbo`

*Batch: 3.10.3*

The MSSQLSpatial driver now creates the metadata tables associated with the `dbo` schema correctly.

### WFS feature counts with client-side filters

*Batch: 3.10.3*

`GetFeatureCount()` no longer crashes on WFS layers that use client-side filters.

### Database and service driver capabilities

*Batch: 3.11.0*

OpenFileGDB adds `CREATE_MULTIPATCH=YES` and accepts ZIP archives whose contents are directly at the top level; SQLite supports `SAVEPOINT` and runs `PRELUDE_STATEMENTS` after initialization and SpatiaLite loading. NGW adds HTTP timeouts/retries, filtered deletes, coded domains, COG and TMS web-map sources, and field alteration; PostgreSQL adds escaping for table names containing an opening parenthesis in `TABLES`, while MySQL removes its deprecated reconnect option for MySQL 8.0.34 and later.

### OAPIF collection item counts

*Batch: 3.11.1*

The OAPIF driver recognizes the `itemCount` element in a Collection description.

### PostgreSQL string truncation restored

*Batch: 3.11.3*

The PostgreSQL driver again truncates strings as intended, restoring behavior that was broken in 3.11.1.

### SQLite `REGEXP` null handling

*Batch: 3.11.4*

The SQLite driver's `REGEXP` behavior for null values now matches the official extension.

### Missing DuckDB databases through ADBC

*Batch: 3.11.5*

The ADBC driver reports an error when asked to open a DuckDB database that does not exist.

### WFS spatial-filter forwarding

*Batch: 3.11.5*

The WFS driver forwards a spatial filter to the server even when it cannot understand the XSD schema.

### MiraMon and ADBC BigQuery

*Batch: 3.12.0*

The new MiraMon raster driver is read-only. The ADBC vector driver can use an installed ADBC BigQuery driver and defers layer loading when no SQL open option is supplied.

### Unqualified NAS property updates

*Batch: 3.12.2*

The NAS driver now updates properties expressed without namespace qualification, as well as their qualified equivalents.

### Fake GeoServer CRS values in WFS

*Batch: 3.12.2*

The WFS driver now skips the synthetic GeoServer CRS identifier `EPSG:404000`.

### Spatial filters in SQL and GeoPackage counts

*Batch: 3.12.3*

OGRSQL honors a spatial filter passed through `ExecuteSQL()` when the result is an aggregation record. GeoPackage `GetFeatureCount()` also reports the filtered count correctly immediately after an insertion within a transaction.

### DuckDB 1.5 through ADBC

*Batch: 3.12.3*

The ADBC driver is compatible with DuckDB 1.5.

### WFS-T typed-value formatting

*Batch: 3.12.4*

WFS-T now formats `xs:dateTime`, `xs:date`, and `xs:boolean` fields correctly.

### Large SQLite compression results

*Batch: 3.13.2*

The SQLite driver's `ogr_inflate` and `ogr_deflate` functions use the 64-bit blob-result API, avoiding size truncation for large results.

## Vector drivers, schemas, and fields

### Vector `CreateCopy()` for GeoPackage

*Batch: 3.10.1*

The GPKG driver's `CreateCopy()` operation now works with vector datasets.

### More general OGR VRT source regions

*Batch: 3.10.1*

An OGR VRT `SrcRegion` accepts any geometry type, as does `SetSpatialFilter()`, and `SrcRegion.clip` is applied correctly at the `OGRVRTLayer` level.

### CSV fields containing double quotes

*Batch: 3.10.2*

The CSV driver now correctly parses a double quote within a field value.

### Zero-sized DXF `INSERT` arrays

*Batch: 3.10.2*

The DXF driver interprets an `INSERT` block whose row count or column count is zero as having a count of one.

### ISO-compliant GML center-point circles

*Batch: 3.10.2*

`gml:CircleByCenterPoint()` now returns a five-point `CIRCULARSTRING`, complying with ISO/IEC 13249-3:2011.

### Multiple zoom-zero MVT tiles

*Batch: 3.10.2*

The MVT driver can generate a tileset with more than one tile at zoom level 0.

### Full-range 64-bit integers in CSV

*Batch: 3.10.3*

The CSV driver now parses 64-bit integer values above `2^53` without losing their integer interpretation.

### Repeated GMLAS string-list elements

*Batch: 3.10.3*

The GMLAS driver now reads every value when a `StringList` field is represented by a repeated element.

### GeoPackage indexed iteration without schema preloading

*Batch: 3.10.3*

After `SetNextByIndex()`, `GetNextFeature()` works without an explicit preceding call to `GetLayerDefn()`.

### Vector schema, defaults, and format controls

*Batch: 3.11.0*

CSV, GML, and SQLite accept the `OGR_SCHEMA` open option, GeoJSON adds `FOREIGN_MEMBERS=AUTO/ALL/NONE/STAC`, and newly created GeoPackages default to version 1.4. DXF creation can set `$INSUNITS` and `$MEASUREMENT` and handles MultiPoint output and WIPEOUT input; Shapefile conversion writes DateTime as ISO 8601 text, GMLAS can resolve CityGML 2.0 without `schemaLocation`, and TopoJSON reads a top-level `crs`.

### Empty GML curves

*Batch: 3.11.1*

The GML geometry parser interprets `<gml:Curve><gml:segments/></gml:Curve>` as `LINESTRING EMPTY`.

### LIBKML creation types

*Batch: 3.11.1*

The LIBKML driver advertises Date, Time, DateTime, and Integer64 fields during creation, mapping them to strings, and maps boolean fields correctly.

### Larger vector concatenations

*Batch: 3.11.4*

`gdal vector concat` accepts more than 1,000 input files.

### Explicit Arrow and Parquet flushing

*Batch: 3.11.4*

Arrow and Parquet datasets implement `Close()`, and their destructors call it so deleting a dataset properly flushes pending output.

### CSV directories with projection sidecars

*Batch: 3.11.4*

The CSV driver can open a directory containing both `.csv` and `.prj` files.

### JGD2024 in GML

*Batch: 3.11.4*

The GML driver recognizes the JGD2024 coordinate reference system used by recent Japanese Fundamental Geospatial Data.

### DXF encoding selection

*Batch: 3.11.5*

The DXF driver now honors its `ENCODING` open option.

### ESRIJSON field types and identification

*Batch: 3.11.5*

The ESRIJSON driver recognizes `esriFieldTypeDateOnly`, `esriFieldTypeTimeOnly`, `esriFieldTypeBigInteger`, `esriFieldTypeGUID`, and `esriFieldTypeGlobalID`, and its format detection recognizes more ESRIJSON variants.

### GML time instants

*Batch: 3.11.5*

The GML and GMLAS drivers support `gml:TimeInstantType`.

### MapInfo pen-width units

*Batch: 3.11.5*

MapInfo `.tab` styling now distinguishes pixel (`px`) from point (`pt`) pen widths and supports fractional point widths.

### Zero-padded MVT input

*Batch: 3.11.5*

The MVT driver can read files containing zero-byte padding.

### Vector update and inspection workflows

*Batch: 3.12.0*

`gdal vector convert` can update an existing output and accept output open options, and `gdal vector write`/`convert` add `--upsert`; `gdal vector sql --update` performs in-place changes. Conversion no longer assumes Shapefile when the output has no `.shp` extension, and text-mode `gdal vector info` needs `--features` to emit features and accepts `--limit`.

### JSON-FG and Parquet evolution

*Batch: 3.12.0*

JSON-FG is updated to specification 0.3.0 with read/write support for curve and measured geometries. Parquet gains editable-layer update support, reads and writes the Parquet `GEOMETRY` type with libarrow 21 or later, and adds a `COMPRESSION_LEVEL` layer creation option.

### CSV and binary DXF creation controls

*Batch: 3.12.0*

CSV creation supports pipe separators and a `HEADER=YES/NO` layer creation option, and its feature IDs are 64-bit. DXF can read AutoCAD Binary DXF and translate it directly to ASCII DXF, and its reader/writer supports true color, transparency, and additional HATCH styling properties.

### GML, GeoPackage, MEM, and PGDump schema capabilities

*Batch: 3.12.0*

GML exposes `SKIP_CORRUPTED_FEATURES` and `SKIP_RESOLVE_ELEMS` open options and reads CityGML 3 Shell geometry. GeoPackage implements field-domain update and deletion, MEM can create a layer from an `OGRFeatureDefn` and declares Boolean, Int16, Float32, JSON, and UUID subtypes, and PGDump adds `SKIP_CONFLICTS`.

### Arrow-backed field selection

*Batch: 3.12.1*

`ogr2ogr` and `VectorTranslate` correctly apply `selectFields` through the Arrow code path.

### VRT source schema and deterministic reads

*Batch: 3.12.1*

The VRT XML schema permits a `name` attribute on source types. Nearest-neighbor reads use the generic raster-band coordinate rounding, and multithreading is disabled for neighboring sources that are not aligned to an integer output coordinate so pixel output remains deterministic.

### GeoJSON export controls

*Batch: 3.12.1*

`OGR_G_ExportToJson()` adds the `ALLOW_MEASURE`, `ALLOW_CURVE`, and `COORDINATE_ORDER` options.

### Parquet list-field handling

*Batch: 3.12.1*

The Parquet driver adds the `LISTS_AS_STRING_JSON=YES/NO` open option. `SetIgnoredFields()` also works for fields whose type is a list of structures.

### LIBKML field-name collisions

*Batch: 3.12.2*

When a LIBKML simple field has the same name as a core attribute, the driver appends `2` to the simple field's name.

### ODS field names with short first rows

*Batch: 3.12.2*

The ODS driver now preserves field names from the title row when the first data row has fewer columns.

### Parquet Hive filtering and GeoArrow interoperability

*Batch: 3.12.2*

Filters now apply correctly to Hive-partitioned Parquet datasets. GeoArrow-encoded files without GeoParquet metadata can also be opened without conflicting with the `geoarrow.pyarrow` Python module.

### Valid bundled JSON schemas

*Batch: 3.12.4*

JSON Schema validation errors in the bundled `gdalinfo` and `ogrinfo` schemas are fixed.

### Arrow validity-buffer compliance

*Batch: 3.12.4*

`CompactValidityBuffer()` now produces ArrowArray-compliant output when `null_count == 0`.

### Filename-safe MiraMon layer names

*Batch: 3.12.4*

The MiraMonVector driver's `CreateLayer()` launders layer names for filename compatibility.

### Parquet `LargeList` fields

*Batch: 3.12.4*

The Parquet driver adds support for the `LargeList` type.

### Vector workflow preservation and inspection

*Batch: 3.13.0*

Unified vector algorithms propagate dataset field domains, relationships, and metadata. Vector filtering adds `--update-extent`, vector info adds `--fid`, and pipelines can use `--no-create-empty-layers`.

### Partitioned vector datasets

*Batch: 3.13.0*

`gdal vector partition` can partition by geometry type, makes `--field` optional, and automatically creates the Parquet `_metadata` index. The same index can be built independently with `gdal driver parquet create-metadata-file`.

### Stricter vector conversion failures

*Batch: 3.13.0*

`ogr2ogr` now fails by default when destination field creation fails unless `-skip` is supplied. It and `gdal vector convert` also warn when the output driver cannot preserve curve, Z, or M geometries.

### Temporal, GeoParquet, and service-driver controls

*Batch: 3.13.0*

Arrow and Parquet support the Timestamp With Offset extension type and a `TIMESTAMP_WITH_OFFSET` layer creation option, while GeoParquet adds `COVERING_BBOX_NAME`. ESRIJSON adds `HTTP_METHOD=AUTO|GET|POST`; PostGIS uses full-geometry spatial intersection and offers `SPATIAL_FILTER_INTERSECTION=LOCAL|SERVER`.

### Additional vector format capabilities

*Batch: 3.13.0*

ILI2 supports INTERLIS 2.4, MEM datasets can create, update, and delete relationships, and MVT reads expose tile-coordinate fields. Shapefile reads `.shp.xml` long field names and aliases, and MapInfo exposes coordinate-system bounds through `BOUNDS` metadata.

### GeoJSON media types

*Batch: 3.13.1*

The GeoJSON writer recognizes `application/geo+json` as well as `application/vnd.geo+json`.

### Parquet ignored-field name collisions

*Batch: 3.13.1*

`SetIgnoredFields()` works when a top-level Parquet column and a nested column have the same name.
