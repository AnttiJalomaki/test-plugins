# Migrations and Core API

C and C++ source migrations, ABI changes, core data types, errors, and capability contracts.

## Canonical opaque-type forward declarations

*Batch 3.11-migration*

The new `gcore/gdal_fwd.h` header normalizes forward declarations for GDAL's public opaque types. Downstream code that redeclares those types can now conflict with GDAL, especially in DEBUG builds with stricter aliases, and should use GDAL's declarations instead.

## CMake minor-version pinning

*Batch 3.11-migration*

Projects that support only GDAL 3.11 must express that minor-version constraint as a range in `find_package()`:

```cmake
find_package(GDAL 3.11...<3.12 REQUIRED)
```

## Const metadata string lists

*Batch 3.13-migration*

`GDALMajorObject::SetMetadata()` now accepts `CSLConstList`, while `GDALMajorObject::GetMetadata()` and `GDALGetMetadata()` now return it. C++ callers that stored returned metadata in `char **` must use `CSLConstList`; that declaration remains compatible with earlier GDAL versions.

## Const-correct vector driver and C++ APIs

*Batch 3.12-migration*

Out-of-tree drivers must update overrides because `GDALDataset::GetLayer()`, `GetLayerCount()`, and `TestCapability()`, plus `OGRLayer::GetName()`, `GetGeomType()`, `GetLayerDefn()`, `GetFIDColumn()`, `GetGeometryColumn()`, `GetSpatialRef()`, and `TestCapability()`, are now const methods. `GetLayer()`, `GetLayerDefn()`, and `GetSpatialRef()` return pointers to const objects; C++ callers also receive a `const OGRFeatureDefn*` from `OGRFeature::GetDefnRef()`. Store these results in const pointers; when only reference-count mutation is required, the migration guidance recommends casting away constness.

## CPL macro and pi exposure changes

*Batch 3.13-migration*

The `MIN`, `MAX`, and `ABS` macros from `port/cpl_port.h` are renamed to `CPL_MIN`, `CPL_MAX`, and `CPL_ABS`. GDAL also no longer exports `M_PI`; code relying on it must define `_USE_MATH_DEFINES` before including `math.h`.

```c
#define _USE_MATH_DEFINES
#include <math.h>
```

## Dataset- and layer-option diagnostics

*Batch 3.13.1*

Dataset creation warns when an unknown dataset creation option matches a layer creation option, and layer creation issues the converse warning.

## Driver extent override hooks

*Batch 3.11-migration*

The public `OGRLayer::GetExtent()` overloads are no longer virtual, and their `bForce` parameter is now `bool`. Drivers should override the protected `IGetExtent(int, OGREnvelope*, bool)` hook; likewise, `GetExtent3D(int, OGREnvelope3D*, bool)` is now the checked public entry point and drivers can override `IGetExtent3D()`.

## `GDALGeoTransform` parameters in raster driver overrides

*Batch 3.12-migration*

The virtual `GDALDataset::GetGeoTransform()` and `SetGeoTransform()` methods now take `GDALGeoTransform&` and `const GDALGeoTransform&`, respectively, rather than pointers to six doubles. Out-of-tree raster drivers must update their overrides; `GDALGeoTransform` is a thin wrapper around `std::array<double, 6>`.

## Geometry point mutation now reports errors

*Batch 3.13-migration*

`OGR_G_SetPointCount`, `OGR_G_SetPoint`, `OGR_G_SetPoint_2D`, `OGR_G_SetPointM`, `OGR_G_SetPointZM`, `OGR_G_AddPoint`, `OGR_G_AddPoint_2D`, `OGR_G_AddPointM`, `OGR_G_AddPointZM`, `OGR_G_SetPoints`, and `OGR_G_SetPointsZM` now return `OGRErr` instead of `void`. C callers should check the result of these mutations.

## Half-precision raster band types

*Batch 3.11-migration*

Drivers may return bands of type `GDT_Float16` or `GDT_CFloat16`, so GDAL API consumers must handle both values in data-type dispatch. Code without native Float16 support can request conversion by passing `GDT_Float32` as the `RasterIO()` buffer type.

## NaN conversion to signed integers

*Batch 3.13.1*

The SSE2 `GDALCopyWords()` path now converts floating-point NaN values to zero for signed 8-, 16-, and 32-bit integer destinations, matching the scalar path.

## Out-of-tree driver signature updates

*Batch 3.13-migration*

`GDALDataset::Close()` overrides must now accept `(GDALProgressFunc pfnProgress, void *pProgressData)`; both arguments may be null. The option-list parameters of `GDALDataset::AddBand()`, `AdviseRead()`, `BeginAsyncReader()`, and `CopyLayer()`, `GDALDriver::pfnCreate` and `pfnCreateCopy`, and `GDALRasterBand::AdviseRead()` and `GetVirtualMemAuto()` are now `CSLConstList` instead of `char **`.

## Partial coordinate-transform failures

*Batch 3.11-migration*

The time-aware `OGRCoordinateTransformation::Transform()` and `TransformWithErrorCodes()` overloads now return `FALSE` when any point fails, rather than only when no point can be transformed. The same rule applies to `GDALTransformerFunc` implementations; inspect `pabSuccess[]` or `panErrorCodes[]` to distinguish successful points from failed ones after an aggregate failure.

## Raster attribute table type and mutation changes

*Batch 3.12-migration*

`GDALRATFieldType` adds `GFT_Boolean`, `GFT_DateTime`, and `GFT_WKBGeometry`, so code switching on `GDALRATGetTypeOfCol()` must handle them. `GDALRasterAttributeTable::SetValue()` methods now return `CPLErr` instead of `void`; callers should check failures and subclasses in out-of-tree drivers must update their overrides.

## RasterIO resampling uses the output buffer type

*Batch 3.13-migration*

RasterIO resampling and VRT operations now run in the output buffer type by default; set the new `GDALRasterIOExtraArg::bOperateInBufType` field to false to opt out. Consequently, non-nearest resampling from a Byte band into a Float32 buffer now generally yields non-integer values.

## Restricted raw VRT bands

*Batch 3.12-migration*

`VRTRawRasterBand` raw-file capabilities are restricted by default for security. VRT workflows that depend on unrestricted raw-file access must account for the `vrtrawrasterband_restricted_access` policy instead of assuming the previous default behavior.

## Spatial-filter API now reports errors

*Batch 3.11-migration*

`OGRLayer::SetSpatialFilter()` and `SetSpatialFilterRect()` are no longer virtual, now return `OGRErr` instead of `void`, and accept `const OGRGeometry*`. Callers should check the result, while drivers should implement the protected `ISetSpatialFilter(int, const OGRGeometry*)` hook.

## Unified CLI input and output option names

*Batch 3.13-migration*

Several unified `gdal` arguments are renamed from the `--src`/`--dst` pattern to `--input`/`--output`. The old names remain accepted by the command line and the C, C++, and Python APIs.

## Unified CLI subcommand moves and output defaults

*Batch 3.12-migration*

`gdal vector geom buffer`, `explode-collections`, `make-valid`, `segmentize`, `simplify`, and `swap-xy` move directly under `gdal vector`; the old paths remain only for 3.12 and are removed in 3.13. `gdal vector geom set-type` is renamed and moved to `gdal vector set-geom-type`. Command-line progress now goes to standard output unless `--quiet`/`-q` is used, and `gdal raster info`, `gdal vector info`, and `gdal vsi list` default to text output at the CLI while retaining JSON defaults through the API.
