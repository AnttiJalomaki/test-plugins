---
name: gdal-knowledge-patch
description: GDAL
version: 3.13.2
license: MIT
metadata:
  author: Nevaberry
---


# GDAL Knowledge Patch

Use this skill when maintaining GDAL applications, extensions, drivers, build
systems, command lines, or data pipelines whose behavior depends on recent
GDAL APIs and utilities.

## Working method

1. Establish the actual GDAL version from the build manifest, package metadata,
   `gdal-config --version`, or `gdal --version`.
2. Identify whether the task concerns an API/ABI migration, CLI behavior,
   raster processing, VSI/cloud access, a format driver, multidimensional data,
   a language binding, or the build system.
3. Read the matching topic file from the index below. Consult more than one file
   for cross-cutting work such as cloud-hosted COG pipelines.
4. Apply notes only when their batch is relevant to the installed version.
5. Prefer the project manifest, source, tests, and observed runtime behavior if
   they disagree with compatibility guidance.
6. Test error paths as well as success paths: several APIs now report partial or
   close-time failures that older callers ignored.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-core-api.md](references/migrations-and-core-api.md) | Source migrations, C/C++ signatures, ABI, data types, errors, capabilities |
| [cli-and-pipelines.md](references/cli-and-pipelines.md) | Unified commands, pipelines, utilities, options, output contracts |
| [raster-processing.md](references/raster-processing.md) | RasterIO, warping, resampling, nodata, masks, statistics, VRT |
| [virtual-filesystems-and-cloud.md](references/virtual-filesystems-and-cloud.md) | VSI, HTTP, credentials, redirects, cache, cloud storage |
| [raster-formats.md](references/raster-formats.md) | Raster drivers, codecs, creation, metadata, validation |
| [vector-formats-and-databases.md](references/vector-formats-and-databases.md) | OGR formats, geometry, schema, SQL, services, Arrow, Parquet |
| [multidimensional-and-georeferencing.md](references/multidimensional-and-georeferencing.md) | Arrays, CRS, transforms, geolocation, dates, scientific data |
| [bindings-build-and-packaging.md](references/bindings-build-and-packaging.md) | CMake, dependencies, headers, Python, C#, Java, SWIG |

## Migration first: C and C++

### Public declarations and driver hooks

- Include `gcore/gdal_fwd.h` instead of redeclaring GDAL opaque types.
- Override protected `IGetExtent()` and `IGetExtent3D()` hooks; the checked
  public `GetExtent()` entry points are not virtual.
- Override `ISetSpatialFilter(int, const OGRGeometry*)`. Public
  `SetSpatialFilter()` and `SetSpatialFilterRect()` return `OGRErr`; callers
  must check it.
- Handle `GDT_Float16` and `GDT_CFloat16` in raster type dispatch. Request a
  `GDT_Float32` `RasterIO()` buffer when native half precision is unsuitable.
- Treat an aggregate coordinate-transform failure as potentially partial and
  inspect `pabSuccess[]` or `panErrorCodes[]` per point.

### Const and geotransform migration

- Out-of-tree vector drivers must make the affected dataset and layer methods
  const and accept the resulting pointers to const layer, feature-definition,
  and spatial-reference objects.
- Handle `GFT_Boolean`, `GFT_DateTime`, and `GFT_WKBGeometry` in raster
  attribute-table switches. Check the `CPLErr` returned by `SetValue()`.
- Update raster-driver `GetGeoTransform()` and `SetGeoTransform()` overrides to
  use `GDALGeoTransform` references rather than `double*` parameters.
- Raw-file `VRTRawRasterBand` access is restricted by default. Treat
  `vrtrawrasterband_restricted_access` and `GDAL_VRT_ENABLE_RAWRASTERBAND` as
  explicit security policy, not incidental configuration.

### Current signature and semantic changes

- Every C point mutation such as `OGR_G_SetPoint()`, `OGR_G_AddPoint()`, and
  their 2D/M/ZM variants returns `OGRErr`; check it. `OGR_G_SetPoint()` can grow
  a geometry again when its index is at or beyond the current point count.
- Replace `MIN`, `MAX`, and `ABS` from `cpl_port.h` with `CPL_MIN`, `CPL_MAX`,
  and `CPL_ABS`. GDAL no longer supplies `M_PI`; arrange the platform math
  definition before including `math.h`.
- Update `GDALDataset::Close()` overrides to accept a progress function and
  progress data. Use `GDALCloseEx()` when close progress must be observed.
- Use `CSLConstList` for the changed option-list and metadata signatures,
  including values returned by `GetMetadata()`.
- Update custom `VSIVirtualHandle::Read()` and `Write()` overrides to the
  single-count `size_t` signatures.
- RasterIO resampling now operates in the output buffer type by default. Set
  `GDALRasterIOExtraArg::bOperateInBufType` to false only when legacy working
  type behavior is required.
- `GDT_UInt8` is the canonical unsigned eight-bit type; `GDT_Byte` is its alias.
- Rebuild binary dependents when moving across a shared-library major-version
  bump, and never mix headers and shared objects from incompatible installs.

## CLI and pipeline quick reference

### Unified command family

- Prefer the `gdal raster`, `gdal vector`, `gdal mdim`, `gdal vsi`,
  `gdal dataset`, and `gdal driver` hierarchy for new automation.
- In 3.12, geometry operations moved directly under `gdal vector`, and
  `geom set-type` became `set-geom-type`; the compatibility paths disappear in
  3.13.
- CLI `raster info`, `vector info`, and `vsi list` default to text, while the API
  keeps JSON defaults. Request a format explicitly in machine-facing scripts.
- Progress is written to standard output unless `--quiet` or `-q` is used.
- Unified argument names favor `--input` and `--output`; the earlier
  `--src`/`--dst` aliases remain accepted.

### Pipelines and algorithms

- Pipelines can combine raster and vector stages, nest pipelines, branch with
  `tee`, materialize intermediate datasets, and invoke an `external` step.
- `_` selects a non-first output dataset from the preceding multi-output stage.
- Named materialization infers its format; an anonymous COG can flow directly
  into `tile`.
- Pipeline inputs may be supplied outside the pipeline string. Nested pipeline
  inputs work with raster comparison, information, tiling, and calculation.
- Python exposes registered algorithms as generated `gdal.alg.*` functions;
  pass `progress=` when callback reporting is needed.
- Validate generated argument lists: algorithms reject malformed lists and
  constrained `NaN` values, and cancellation reports `CE_Failure`.

### Failure-sensitive automation

- `ogr2ogr` fails by default when destination-field creation fails; opt into
  `-skip` only deliberately. It also returns nonzero for VRT processing errors.
- `gdalwarp` fails if `INIT_DEST=NO_DATA` has no nodata value. Earlier warning
  and zero-initialization behavior must not be assumed.
- `gdal raster mosaic --target-aligned-pixels` requires `--resolution`.
- Text `gdal vector info` requires `--features` to emit features.
- Explicitly request `--features`, JSON/text output, quiet operation, and output
  layer names in scripts instead of relying on changing defaults.

## Raster correctness quick reference

- For non-nearest Byte-to-Float32 resampling, expect fractional output because
  processing uses the destination buffer type.
- Lanczos no longer discards results merely because fewer than half of source
  pixels are valid; masked and nodata-edge output can differ.
- NaN-to-signed-integer conversion yields zero in the optimized and scalar
  paths. NaN nodata is handled by bilinear, cubic, cubic-spline, and Lanczos.
- Integer nodata exclusion in min/max and histogram calculations now requires
  an exact integer match.
- `RESET_DEST_PIXELS=YES` can reset an existing warp destination fully to its
  nodata value or zero.
- Check per-source failures in GTI reads, partial transform status, close-time
  flush errors, and progress cancellation.
- VRT derived functions include expression, reclassification, aggregate,
  positional, quantile, area, and rounding operations; consult the raster file
  for nodata and overview details before relying on their output.

## Cloud and VSI quick reference

- `/vsis3/` supports IAM Identity Center, `credential_process`, directory
  buckets, and path-specific request behavior. Authentication changes invalidate
  cached `/vsigs/` and `/vsiaz/` state.
- Use path-specific `VSICURL_QUERY_STRING`, URL `header.<key>=<value>`, and the
  connection-limit settings for controlled HTTP access.
- `AWS_S3_ENDPOINT` may include its scheme. Cloud paths normalize `/./` and
  `/../` unless `GDAL_HTTP_PATH_VERBATIM` is set.
- Set `CACHE=ON/OFF` through `VSIFOpenEx2L()` when post-close caching matters.
- Do not expect authorization to follow an S3-like redirect. Configure redirect
  authorization policy per path where needed.
- `CPL_NULL_VALUE` explicitly masks an environment-provided configuration value.
- Empty files participate in multithreaded `VSISync()` cloud transfers.

## Driver and data-model highlights

- Treat driver availability as version-specific: several legacy drivers or
  writers were removed, some raster drivers were restored in patch releases,
  and Tiger plus UK .NTF later returned as removal candidates.
- The unified `MEM` driver replaces the deprecated OGR `Memory` name.
- FileGDB creation and update use OpenFileGDB. GeoPackage creation defaults to
  version 1.4.
- LIBERTIFF is the thread-safe read-only GeoTIFF path. JP2GROK uses the
  AGPLv3-licensed Grok toolkit and has versioned minimum dependency requirements.
- COG supports band/tile interleave, complex types, random-write creation, and
  BigTIFF intermediates. Format and mask details remain version-sensitive.
- Zarr v3 supports consolidated metadata, sharding, additional codecs and data
  types, mapped multiscales, and multidimensional overviews.
- Parquet supports editable layers, native geometry types, partition metadata,
  list variants, timestamp offsets, and GeoArrow interoperability.
- Arrow and Parquet implement explicit `Close()`; close or destroy writers so
  pending output is flushed and failures can surface.

## Binding reminders

- Python accepts CRS definitions in `osr.SpatialReference()`, NumPy types in
  `Driver.Create()`, path-like values for rename/copy, and list-form open
  options in helpers such as `VectorTranslate()`.
- Raster Python APIs include masked arrays, mask resampling, block-window
  iteration, Boolean mapping, and band inputs to `CreateCopy()`.
- `Feature.GetDefnRef()` now increments the returned definition's reference
  count; account for ownership in long-running binding code.
- Free-threaded Python builds are supported for Python 3.13 and later.
- Close datasets intentionally in Python, Java, and Arrow/Parquet workflows;
  do not use object destruction as the only correctness boundary.

## Verification checklist

- Confirm command and option availability against the installed binary.
- Compile out-of-tree drivers with warnings enabled and exercise virtual hooks.
- Run raster tests across nodata, NaN, masks, overviews, strides, edge windows,
  and large dimensions.
- Validate pipeline exit status, standard streams, output format, and layer names.
- Exercise cloud authentication refresh, redirects, listing, empty files, and
  cache invalidation.
- Reopen created datasets and explicitly close streaming writers.
- Test geometry and coordinate operations with partial failures, 3D/curve/M
  values, polar regions, and invalid input.
