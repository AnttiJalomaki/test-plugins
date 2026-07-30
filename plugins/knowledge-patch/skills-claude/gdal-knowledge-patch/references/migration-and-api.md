# Migration and API compatibility

Use this reference for the task areas below. Batch labels identify when each behavior entered the covered compatibility history.

## Breaking migrations

### Canonical opaque-type forward declarations

*Batch: 3.11-migration*

The new `gcore/gdal_fwd.h` header normalizes forward declarations for GDAL's public opaque types. Downstream code that redeclares those types can now conflict with GDAL, especially in DEBUG builds with stricter aliases, and should use GDAL's declarations instead.

### Driver extent override hooks

*Batch: 3.11-migration*

The public `OGRLayer::GetExtent()` overloads are no longer virtual, and their `bForce` parameter is now `bool`. Drivers should override the protected `IGetExtent(int, OGREnvelope*, bool)` hook; likewise, `GetExtent3D(int, OGREnvelope3D*, bool)` is now the checked public entry point and drivers can override `IGetExtent3D()`.

### Spatial-filter API now reports errors

*Batch: 3.11-migration*

`OGRLayer::SetSpatialFilter()` and `SetSpatialFilterRect()` are no longer virtual, now return `OGRErr` instead of `void`, and accept `const OGRGeometry*`. Callers should check the result, while drivers should implement the protected `ISetSpatialFilter(int, const OGRGeometry*)` hook.

### Half-precision raster band types

*Batch: 3.11-migration*

Drivers may return bands of type `GDT_Float16` or `GDT_CFloat16`, so GDAL API consumers must handle both values in data-type dispatch. Code without native Float16 support can request conversion by passing `GDT_Float32` as the `RasterIO()` buffer type.

### CMake minor-version pinning

*Batch: 3.11-migration*

Projects that support only GDAL 3.11 must express that minor-version constraint as a range in `find_package()`:

```cmake
find_package(GDAL 3.11...<3.12 REQUIRED)
```

### Partial coordinate-transform failures

*Batch: 3.11-migration*

The time-aware `OGRCoordinateTransformation::Transform()` and `TransformWithErrorCodes()` overloads now return `FALSE` when any point fails, rather than only when no point can be transformed. The same rule applies to `GDALTransformerFunc` implementations; inspect `pabSuccess[]` or `panErrorCodes[]` to distinguish successful points from failed ones after an aggregate failure.

### Unified CLI subcommand moves and output defaults

*Batch: 3.12-migration*

`gdal vector geom buffer`, `explode-collections`, `make-valid`, `segmentize`, `simplify`, and `swap-xy` move directly under `gdal vector`; the old paths remain only for 3.12 and are removed in 3.13. `gdal vector geom set-type` is renamed and moved to `gdal vector set-geom-type`. Command-line progress now goes to standard output unless `--quiet`/`-q` is used, and `gdal raster info`, `gdal vector info`, and `gdal vsi list` default to text output at the CLI while retaining JSON defaults through the API.

### Const-correct vector driver and C++ APIs

*Batch: 3.12-migration*

Out-of-tree drivers must update overrides because `GDALDataset::GetLayer()`, `GetLayerCount()`, and `TestCapability()`, plus `OGRLayer::GetName()`, `GetGeomType()`, `GetLayerDefn()`, `GetFIDColumn()`, `GetGeometryColumn()`, `GetSpatialRef()`, and `TestCapability()`, are now const methods. `GetLayer()`, `GetLayerDefn()`, and `GetSpatialRef()` return pointers to const objects; C++ callers also receive a `const OGRFeatureDefn*` from `OGRFeature::GetDefnRef()`. Store these results in const pointers; when only reference-count mutation is required, the migration guidance recommends casting away constness.

### Raster attribute table type and mutation changes

*Batch: 3.12-migration*

`GDALRATFieldType` adds `GFT_Boolean`, `GFT_DateTime`, and `GFT_WKBGeometry`, so code switching on `GDALRATGetTypeOfCol()` must handle them. `GDALRasterAttributeTable::SetValue()` methods now return `CPLErr` instead of `void`; callers should check failures and subclasses in out-of-tree drivers must update their overrides.

### Restricted raw VRT bands

*Batch: 3.12-migration*

`VRTRawRasterBand` raw-file capabilities are restricted by default for security. VRT workflows that depend on unrestricted raw-file access must account for the `vrtrawrasterband_restricted_access` policy instead of assuming the previous default behavior.

### `GDALGeoTransform` parameters in raster driver overrides

*Batch: 3.12-migration*

The virtual `GDALDataset::GetGeoTransform()` and `SetGeoTransform()` methods now take `GDALGeoTransform&` and `const GDALGeoTransform&`, respectively, rather than pointers to six doubles. Out-of-tree raster drivers must update their overrides; `GDALGeoTransform` is a thin wrapper around `std::array<double, 6>`.

### Geometry point mutation now reports errors

*Batch: 3.13-migration*

`OGR_G_SetPointCount`, `OGR_G_SetPoint`, `OGR_G_SetPoint_2D`, `OGR_G_SetPointM`, `OGR_G_SetPointZM`, `OGR_G_AddPoint`, `OGR_G_AddPoint_2D`, `OGR_G_AddPointM`, `OGR_G_AddPointZM`, `OGR_G_SetPoints`, and `OGR_G_SetPointsZM` now return `OGRErr` instead of `void`. C callers should check the result of these mutations.

### CPL macro and pi exposure changes

*Batch: 3.13-migration*

The `MIN`, `MAX`, and `ABS` macros from `port/cpl_port.h` are renamed to `CPL_MIN`, `CPL_MAX`, and `CPL_ABS`. GDAL also no longer exports `M_PI`; code relying on it must define `_USE_MATH_DEFINES` before including `math.h`.

```c
#define _USE_MATH_DEFINES
#include <math.h>
```

### Out-of-tree driver signature updates

*Batch: 3.13-migration*

`GDALDataset::Close()` overrides must now accept `(GDALProgressFunc pfnProgress, void *pProgressData)`; both arguments may be null. The option-list parameters of `GDALDataset::AddBand()`, `AdviseRead()`, `BeginAsyncReader()`, and `CopyLayer()`, `GDALDriver::pfnCreate` and `pfnCreateCopy`, and `GDALRasterBand::AdviseRead()` and `GetVirtualMemAuto()` are now `CSLConstList` instead of `char **`.

### Const metadata string lists

*Batch: 3.13-migration*

`GDALMajorObject::SetMetadata()` now accepts `CSLConstList`, while `GDALMajorObject::GetMetadata()` and `GDALGetMetadata()` now return it. C++ callers that stored returned metadata in `char **` must use `CSLConstList`; that declaration remains compatible with earlier GDAL versions.

### Unified CLI input and output option names

*Batch: 3.13-migration*

Several unified `gdal` arguments are renamed from the `--src`/`--dst` pattern to `--input`/`--output`. The old names remain accepted by the command line and the C, C++, and Python APIs.

### RasterIO resampling uses the output buffer type

*Batch: 3.13-migration*

RasterIO resampling and VRT operations now run in the output buffer type by default; set the new `GDALRasterIOExtraArg::bOperateInBufType` field to false to opt out. Consequently, non-nearest resampling from a Byte band into a Float32 buffer now generally yields non-integer values.

## Public C, C++, and raster APIs

### Raster SDK additions

*Batch: 3.11.0*

New SDK facilities include `gdal::CXXTypeTraits<T>`, `gdal::GDALDataTypeTraits<T>`, `gdal_minmax_element.hpp`, `gdal::VectorX`, `GDALRasterComputeMinMaxLocation()`/`GDALRasterBand::ComputeMinMaxLocation()`, `GDALDataset::GeolocationToPixelLine()`, `GDALRasterBand::InterpolateAtGeolocation()`, `GDALTranspose2D()`, `GDALGroup::GetMDArrayFullNamesRecursive()`, `GDALIsValueInRangeOf()`, and `GDALRasterBand::SetNoDataValueAsString()`.

### OGR APIs and Arrow time values

*Batch: 3.11.0*

`OGRFieldDefn::SetGenerated()`/`IsGenerated()` marks generated fields, `OSRGetAuthorityListFromDatabase()` lists CRS authorities from PROJ, and `OGR_GT_GetSingle()` is available through SWIG. `OGRLayer::GetArrowStream()` adds `DATETIME_AS_STRING=YES/NO`; `ogr2ogr` uses it to preserve source time zones and can now transfer dataset relationships when the target supports them.

### IIIF Image API 3.0

*Batch: 3.11.1*

The WMS driver adds a mini-driver for International Image Interoperability Framework Image API 3.0.

### Raster band algebra API

*Batch: 3.12.0*

The C, C++, and Python APIs support arithmetic and comparison directly on raster bands, type conversion with `AsType()`, and algebra functions including `abs()`, `sqrt()`, logarithms, `min()`, `max()`, `mean()`, and `IfThenElse()`.

### Dataset extent, overview, and window APIs

*Batch: 3.12.0*

New APIs include `GDALDataset::GetLayerIndex()`, `GetExtent()`, `GetExtentWGS84LongLat()`, and `AddOverviews()`, plus `GDALRasterBand::IterateWindows()` and `SplitRasterIO()`. `GDALGetGDALPath()` exposes GDAL's installation path, and `GDALRescaleGeoTransform()` rescales a geotransform.

### Geolocation, geometry, schema, and celestial-body APIs

*Batch: 3.12.0*

The geolocation transformer adds `GEOLOC_NORMALIZE_LONGITUDE_MINUS_180_PLUS_180` to force longitude normalization. OGR adds envelope-to-geometry creation and constrained Delaunay triangulation, vector datasets expose `GetSpatialRef()`, schema overrides accept `*` layer matching and `srcType`/`srcSubType` matching, and CRS APIs can report celestial-body names.

### Progressive dataset closure

*Batch: 3.13.0*

The new `GDALCloseEx()` API and `GDALDataset::Close()` progress callback support observable long-running closes; `GDALDataset::GetCloseReportsProgress()` reports whether a dataset provides that progress.

### Newly public headers

*Batch: 3.13.0*

The installed headers now include `gdal_mem.h`, which exposes the `MEMCreate()` C API, plus `gdal_thread_pool.h` and `ogr_refcountedptr.h`.

### Vector geometry and SQL APIs

*Batch: 3.13.0*

The GeoPackage and SQLite dialects add `ST_Hilbert()`, and geometry APIs add polygon-based concave hull generation plus invalidity-reason retrieval in C, C++, and SWIG. `ExportToKML()` now fails rather than emitting coordinates with invalid latitudes.

## Driver and algorithm extension APIs

### Driver capability and algorithm metadata

*Batch: 3.12.0*

Drivers can advertise maximum string length and the new append, upsert, close-time visibility, reopen-after-write, and read-after-delete capabilities. Algorithm consumers can retrieve typed default arguments through the new C/SWIG getters, while algorithm implementers gain dedicated geometry-type, append-layer, overwrite-layer, absolute-path, stdout, hidden, and deprecation helpers.

### Driver exclusion and algorithm metadata

*Batch: 3.13.0*

An allowed-driver entry prefixed with `-` now excludes that driver in `GDALOpenEx()`. Algorithm front ends can inspect pipeline-step availability, direct and aggregate argument dependencies, mutual-dependency groups, duplicate-value allowance, and maximum character counts.

### Arrow and directory-oriented vector capabilities

*Batch: 3.13.0*

Arrow field creation and batch writing support string-view values, and the C API gains `OGR_L_GetAttributeFilter()`. A new driver capability identifies directories that may contain multiple vector layers and is advertised by Shapefile, MapInfo, CSV, FlatGeobuf, and MiraMonVector.

## Language and binding APIs

### Python color interpretation during translation

*Batch: 3.10.1*

The Python `gdal.Translate()` binding adds a `colorInterpretation` argument; the similar argument in `gdal.TileIndex()` also receives a correctness fix.

### Python raster arrays and accepted inputs

*Batch: 3.11.0*

Python adds `Dataset.ReadAsMaskedArray()`, `mask_resample_alg` on `ReadAsArray()` methods, and the `-epo`/`-eco` translation controls; `gdal.VectorTranslate()` gains `relatedFieldNameMatch`. `osr.SpatialReference()` accepts a CRS definition, `Driver.Create()` accepts NumPy types, `Driver.Rename()`/`CopyFiles()` accept `os.PathLike`, and `GDAL_PYTHON_BINDINGS_WITHOUT_NUMPY` accepts `YES/1/ON/TRUE` or `NO/0/OFF/FALSE`.

### Range-domain validation and binding errors

*Batch: 3.11.1*

Python's `ogr.CreateRangeFieldDomain()` and `ogr.CreateRangeFieldDomainDateTime()` correctly handle `None` bounds. The OpenFileGDB writer rejects range domains missing either bound, and SWIG `AddFieldDomain()` surfaces failures as errors or exceptions.

### C# spatial-reference matching

*Batch: 3.11.1*

The C# bindings add `SpatialReference.FindMatches`.

### Zero-stride Python array writes

*Batch: 3.11.4*

Python `Dataset.WriteArray()` and `Band.WriteArray()` correctly write arrays containing a zero stride.

### Java dataset closure

*Batch: 3.11.4*

Closing a dataset obtained with `Band.GetDataset().Close()` no longer causes a double free.

### Python algorithm namespace

*Batch: 3.12.0*

Python exposes the algorithm registry through a dynamically generated `gdal.alg` module:

```python
gdal.alg.raster.convert(input="in.tif", output="out.tif")
```

### Python raster iteration and Boolean arrays

*Batch: 3.12.0*

Python adds `Band.BlockWindows()`, permits a band as `Driver.CreateCopy()` input, maps NumPy Boolean types to GDAL types, and avoids promoting Boolean arrays to `float64` when writing. Configuration-option values are coerced to strings.

### SWIG feature-definition ownership

*Batch: 3.12.1*

`Feature.GetDefnRef` now increments the returned `FeatureDefn` reference count.

### Binding behavior

*Batch: 3.13.0*

Python `Dataset.AdviseRead()` and `Band.AdviseRead()` accept keywords, with dataset calls defaulting to all bands; algorithm functions accept visible and hidden argument aliases, and `Feature.SetField()` accepts NumPy values. Java exposes full and partial `/vsicurl/` cache clearing, while SWIG adds the missing relationship capability constants.

### Python open-option parsing

*Batch: 3.13.1*

Python methods such as `gdal.VectorTranslate()` recognize the list form `options=["-oo", "FOO=BAR"]`.
