# Vector Formats and Databases

OGR formats, geometry behavior, schemas, SQL, databases, services, Arrow, and Parquet.

## Additional vector format capabilities

*Batch 3.13.0*

ILI2 supports INTERLIS 2.4, MEM datasets can create, update, and delete relationships, and MVT reads expose tile-coordinate fields. Shapefile reads `.shp.xml` long field names and aliases, and MapInfo exposes coordinate-system bounds through `BOUNDS` metadata.

## Arrow validity-buffer compliance

*Batch 3.12.4*

`CompactValidityBuffer()` now produces ArrowArray-compliant output when `null_count == 0`.

## C geometry growth through `OGR_G_SetPoint()`

*Batch 3.13.2*

`OGR_G_SetPoint()` can again grow a geometry when the supplied point index is greater than or equal to its current point count.

## Complex OSM multipolygons

*Batch 3.12.2*

The OSM driver again reads complex multipolygons correctly, fixing a regression introduced in 3.11.5.

## CSV and binary DXF creation controls

*Batch 3.12.0*

CSV creation supports pipe separators and a `HEADER=YES/NO` layer creation option, and its feature IDs are 64-bit. DXF can read AutoCAD Binary DXF and translate it directly to ASCII DXF, and its reader/writer supports true color, transparency, and additional HATCH styling properties.

## CSV directories with projection sidecars

*Batch 3.11.4*

The CSV driver can open a directory containing both `.csv` and `.prj` files.

## CSV fields containing double quotes

*Batch 3.10.2*

The CSV driver now correctly parses a double quote within a field value.

## CSV geometry-option routing

*Batch 3.13.1*

For CSV output, `GEOMETRY=AS_WKT` supplied as a layer creation option is also set as a dataset creation option.

## DuckDB 1.5 through ADBC

*Batch 3.12.3*

The ADBC driver is compatible with DuckDB 1.5.

## DXF encoding selection

*Batch 3.11.5*

The DXF driver now honors its `ENCODING` open option.

## ESRIJSON field types and identification

*Batch 3.11.5*

The ESRIJSON driver recognizes `esriFieldTypeDateOnly`, `esriFieldTypeTimeOnly`, `esriFieldTypeBigInteger`, `esriFieldTypeGUID`, and `esriFieldTypeGlobalID`, and its format detection recognizes more ESRIJSON variants.

## Explicit Arrow and Parquet flushing

*Batch 3.11.4*

Arrow and Parquet datasets implement `Close()`, and their destructors call it so deleting a dataset properly flushes pending output.

## Filename-safe MiraMon layer names

*Batch 3.12.4*

The MiraMonVector driver's `CreateLayer()` launders layer names for filename compatibility.

## Full-range 64-bit integers in CSV

*Batch 3.10.3*

The CSV driver now parses 64-bit integer values above `2^53` without losing their integer interpretation.

## GeoJSON detection when geometry comes first

*Batch 3.10.3*

The GeoJSON driver now detects features whose object starts with a `geometry` member, including input beginning `{"geometry":{"type":...,"coordinates":...`.

## GeoJSON export controls

*Batch 3.12.1*

`OGR_G_ExportToJson()` adds the `ALLOW_MEASURE`, `ALLOW_CURVE`, and `COORDINATE_ORDER` options.

## GeoJSON media types

*Batch 3.13.1*

The GeoJSON writer recognizes `application/geo+json` as well as `application/vnd.geo+json`.

## GeoPackage indexed iteration without schema preloading

*Batch 3.10.3*

After `SetNextByIndex()`, `GetNextFeature()` works without an explicit preceding call to `GetLayerDefn()`.

## GML, GeoPackage, MEM, and PGDump schema capabilities

*Batch 3.12.0*

GML exposes `SKIP_CORRUPTED_FEATURES` and `SKIP_RESOLVE_ELEMS` open options and reads CityGML 3 Shell geometry. GeoPackage implements field-domain update and deletion, MEM can create a layer from an `OGRFeatureDefn` and declares Boolean, Int16, Float32, JSON, and UUID subtypes, and PGDump adds `SKIP_CONFLICTS`.

## Large SQLite compression results

*Batch 3.13.2*

The SQLite driver's `ogr_inflate` and `ogr_deflate` functions use the 64-bit blob-result API, avoiding size truncation for large results.

## MapInfo pen-width units

*Batch 3.11.5*

MapInfo `.tab` styling now distinguishes pixel (`px`) from point (`pt`) pen widths and supports fractional point widths.

## MiraMon and ADBC BigQuery

*Batch 3.12.0*

The new MiraMon raster driver is read-only. The ADBC vector driver can use an installed ADBC BigQuery driver and defers layer loading when no SQL open option is supplied.

## Missing DuckDB databases through ADBC

*Batch 3.11.5*

The ADBC driver reports an error when asked to open a DuckDB database that does not exist.

## MSSQLSpatial metadata tables in `dbo`

*Batch 3.10.3*

The MSSQLSpatial driver now creates the metadata tables associated with the `dbo` schema correctly.

## Multiple zoom-zero MVT tiles

*Batch 3.10.2*

The MVT driver can generate a tileset with more than one tile at zoom level 0.

## New data-source drivers

*Batch 3.11.0*

The read-only OGR ADBC driver can access DuckDB or Parquet datasets when libduckdb is installed, while LIBERTIFF provides a native thread-safe read-only GeoTIFF reader. Read-only RCM and AIVector drivers are also new.

## OAPIF collection item counts

*Batch 3.11.1*

The OAPIF driver recognizes the `itemCount` element in a Collection description.

## ODS field names with short first rows

*Batch 3.12.2*

The ODS driver now preserves field names from the title row when the first data row has fewer columns.

## Parquet Hive filtering and GeoArrow interoperability

*Batch 3.12.2*

Filters now apply correctly to Hive-partitioned Parquet datasets. GeoArrow-encoded files without GeoParquet metadata can also be opened without conflicting with the `geoarrow.pyarrow` Python module.

## Parquet ignored-field name collisions

*Batch 3.13.1*

`SetIgnoredFields()` works when a top-level Parquet column and a nested column have the same name.

## Parquet `LargeList` fields

*Batch 3.12.4*

The Parquet driver adds support for the `LargeList` type.

## Progressive dataset closure

*Batch 3.13.0*

The new `GDALCloseEx()` API and `GDALDataset::Close()` progress callback support observable long-running closes; `GDALDataset::GetCloseReportsProgress()` reports whether a dataset provides that progress.

## Removed drivers and writers

*Batch 3.11.0*

Removed raster drivers are BLX, BT, CTable2, ELAS, FIT, GSAG, GSBG, JP2Lura, OZI OZF2/OZFX3, Rasterlite v1, R object `.rda`, RDB, SDTS, SGI, XPM, and DIPex; removed vector drivers are Geoconcept Export, OGDI, SDTS, SVG, Tiger, and UK .NTF. Write support was removed from Interlis 1/2, ADRG, PAux, MFF, MFF2/HKV, LAN, NTv2, BYN, USGSDEM, and ISIS2.

## Restored legacy drivers and ABI change

*Batch 3.13.0*

The OGR Tiger and UK .NTF drivers are restored after their 3.11 removal, although both remain candidates for future removal. The shared library major version is bumped, so binary dependents must be rebuilt or use matching GDAL libraries.

## Scientific and hydrographic raster formats

*Batch 3.12.0*

`DTED_ASSUME_COMPLIANT` opts out of the driver's DTED value conversion below `-16000`. PDS4 supports `Int64` and `UInt64` rasters plus hexadecimal constant values; S102 reads Edition 3.0, S104 and S111 read Edition 2.0, and the S10x drivers decode custom coordinate reference systems.

## South-up and newer STAC data in GTI

*Batch 3.12.1*

GTI accepts south-up tiles and automatically warps them north-up. For STAC GeoParquet, it recognizes `stac_extensions` as a marker and supports a top-level `bands` object plus the EO 2.0 extension; URL rewriting is limited to STAC collection catalogs.

## Spatial filters in SQL and GeoPackage counts

*Batch 3.12.3*

OGRSQL honors a spatial filter passed through `ExecuteSQL()` when the result is an aggregation record. GeoPackage `GetFeatureCount()` also reports the filtered count correctly immediately after an insertion within a transaction.

## SQLite `REGEXP` null handling

*Batch 3.11.4*

The SQLite driver's `REGEXP` behavior for null values now matches the official extension.

## Temporal, GeoParquet, and service-driver controls

*Batch 3.13.0*

Arrow and Parquet support the Timestamp With Offset extension type and a `TIMESTAMP_WITH_OFFSET` layer creation option, while GeoParquet adds `COVERING_BBOX_NAME`. ESRIJSON adds `HTTP_METHOD=AUTO|GET|POST`; PostGIS uses full-geometry spatial intersection and offers `SPATIAL_FILTER_INTERSECTION=LOCAL|SERVER`.

## Unqualified NAS property updates

*Batch 3.12.2*

The NAS driver now updates properties expressed without namespace qualification, as well as their qualified equivalents.

## Vector `CreateCopy()` for GeoPackage

*Batch 3.10.1*

The GPKG driver's `CreateCopy()` operation now works with vector datasets.

## WFS feature counts with client-side filters

*Batch 3.10.3*

`GetFeatureCount()` no longer crashes on WFS layers that use client-side filters.

## WFS spatial-filter forwarding

*Batch 3.11.5*

The WFS driver forwards a spatial filter to the server even when it cannot understand the XSD schema.

## WFS-T typed-value formatting

*Batch 3.12.4*

WFS-T now formats `xs:dateTime`, `xs:date`, and `xs:boolean` fields correctly.

## Zero-padded MVT input

*Batch 3.11.5*

The MVT driver can read files containing zero-byte padding.

## Zero-sized DXF `INSERT` arrays

*Batch 3.10.2*

The DXF driver interprets an `INSERT` block whose row count or column count is zero as having a count of one.
