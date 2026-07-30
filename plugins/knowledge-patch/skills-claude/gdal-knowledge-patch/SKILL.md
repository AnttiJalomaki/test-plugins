---
name: gdal-knowledge-patch
description: GDAL
version: 3.13.2
license: MIT
metadata:
  author: Nevaberry
---


# GDAL Knowledge Patch

Use this skill when upgrading, building, extending, or troubleshooting GDAL,
its unified command family, raster and vector drivers, multidimensional data,
cloud virtual filesystems, or language bindings.

## How to use this skill

1. Determine the GDAL version used by the project from its package manifest,
   build configuration, container image, or linked library.
2. Read the migration checklist before changing downstream C or C++ code,
   out-of-tree drivers, custom VSI handlers, or command invocations.
3. Open the task-specific reference from the index; reference entries retain
   exact batch labels for version-sensitive behavior.
4. Prefer the installed headers, driver metadata, command help, and project
   tests when local behavior differs from this guidance.
5. Treat a newer project version as potentially beyond this patch and verify
   changed APIs and defaults directly.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migration-and-api.md](references/migration-and-api.md) | Breaking C/C++ migrations, public APIs, driver extension points, bindings |
| [references/commands-and-pipelines.md](references/commands-and-pipelines.md) | Unified commands, pipelines, algorithms, utility behavior and exit contracts |
| [references/raster-processing-and-formats.md](references/raster-processing-and-formats.md) | Warping, reprojection, VRT, nodata, overviews, raster formats |
| [references/vector-and-database-workflows.md](references/vector-and-database-workflows.md) | Geometry, schemas, vector formats, SQL, databases, network services |
| [references/multidimensional-and-scientific.md](references/multidimensional-and-scientific.md) | Multidimensional arrays, Zarr, GTI/STAC, scientific and product formats |
| [references/cloud-vsi-build-and-bindings.md](references/cloud-vsi-build-and-bindings.md) | Cloud authentication, HTTP/VSI, builds, dependencies, language bindings |

## Migration quick reference

### C and C++ callers

- Use GDAL's canonical opaque-type declarations from `gdal_fwd.h`; remove
  downstream redeclarations that can conflict under strict debug aliases.
- Check `OGRErr` from spatial-filter setters and geometry point mutation APIs.
- Store metadata returned by `GetMetadata()` as `CSLConstList`, not `char **`.
- Handle `GDT_Float16`, `GDT_CFloat16`, and canonical `GDT_UInt8` in data-type
  dispatch. Request `GDT_Float32` conversion when native half precision is not
  supported by the caller.
- Expect aggregate coordinate transformation failure when any point fails;
  inspect the per-point success or error arrays for partial results.
- Account for RasterIO resampling operating in the output buffer type. Set
  `GDALRasterIOExtraArg::bOperateInBufType` to false only when older working-
  type behavior is deliberately required.
- Replace `MIN`, `MAX`, and `ABS` uses supplied by GDAL with `CPL_MIN`,
  `CPL_MAX`, and `CPL_ABS`. Arrange for the platform math header to expose pi
  instead of relying on GDAL to export `M_PI`.

### Out-of-tree driver authors

- Override `IGetExtent()` and `IGetExtent3D()` rather than the checked public
  extent methods.
- Implement `ISetSpatialFilter()` instead of overriding the public spatial-
  filter entry points.
- Update const-correct dataset and layer overrides and retain const pointers
  returned by layer, feature-definition, and spatial-reference accessors.
- Update geotransform overrides to use `GDALGeoTransform` references.
- Update raster attribute table switches for Boolean, DateTime, and WKB
  geometry fields, and return/check errors from `SetValue()` mutations.
- Update `GDALDataset::Close()` overrides for progress arguments and option-
  list parameters for `CSLConstList`.
- Update custom `VSIVirtualHandle::Read()` and `Write()` overrides to the
  single-count signatures.
- Rebuild binary dependents when crossing a shared-library major-version bump.

### Command and configuration migrations

- Move geometry subcommands directly under `gdal vector`; use
  `set-geom-type` for the renamed geometry-type operation.
- Prefer `--input` and `--output` in unified commands. Legacy `--src` and
  `--dst` aliases remain accepted where compatibility is required.
- Do not assume info and VSI-list commands choose JSON at the terminal; ask
  explicitly for machine-readable output.
- Do not parse progress output as data. Use quiet mode for scripts whose
  standard output is a data channel.
- Audit raw VRT use. Raw-file access is restricted by default and can also be
  compiled out.
- Replace `MRF_BYPASSCACHING` with `MRF_ENABLE_CACHING`.
- For GDAL-only minor compatibility, express a bounded range in
  `find_package()` instead of relying on a minimum version alone.

## Unified command and pipeline quick reference

- The `gdal` front end covers raster, vector, multidimensional, dataset, VSI,
  and driver operations; prefer it for new automation while retaining legacy
  utilities where their exact contracts matter.
- Raster pipelines can begin with mosaic or stack, use analysis and editing
  stages, materialize intermediate datasets, and terminate in tiling.
- Vector pipelines support selection, SQL, concatenation, filtering, limits,
  geometry operations, and writing; selected-layer reads can feed SQL stages.
- General pipelines can mix raster and vector stages, nest another pipeline,
  branch with `tee`, execute an external step, and select a non-first output
  from a preceding multi-output stage with `_`.
- Treat pipeline inputs and outputs as datasets, not merely filenames. Several
  commands accept externally supplied or anonymous pipeline datasets.
- Use explicit output layers when composing contour, polygonize, select, or
  other stages that can expose more than one layer.
- In Python, use the generated `gdal.alg` namespace for algorithm calls and
  pass `progress=` when cancellation or reporting is needed.
- Validate algorithm arguments before execution: malformed lists, constrained
  `NaN` values, invalid numeric options, and incompatible dependencies are
  rejected rather than coerced.
- Check process status in automation. Utilities now surface failures for VRT
  processing, failed footprint simplification, and other previously silent
  error paths.

## Raster quick reference

### Warping and reprojection

- Inspect per-band data types before choosing a warp working type, especially
  for signed bytes and half precision.
- Expect mode resampling to use source-pixel coverage, RMS overviews to use
  corrected normalization, and Lanczos output near masks or nodata to differ
  from older special-case behavior.
- For `INIT_DEST=NO_DATA`, provide a destination nodata value. Use
  `RESET_DEST_PIXELS` when an existing destination must be reset completely.
- Treat NaN nodata explicitly for bilinear, cubic, cubic-spline, and Lanczos
  resampling; exact integer nodata exclusion also affects statistics.
- Use bounds transformation for target extents and account for global,
  non-square-pixel, polar, south-up, and rotated-source cases.
- Honor interruption from progress callbacks in both single- and
  multithreaded work.

### VRT, masks, and overviews

- VRT derived bands support expressions, reclassification, aggregate pixel
  functions, constants, transposition, block selection, and implicit
  overviews.
- Embedded source datasets and processed-VRT output-band declarations can
  avoid temporary named datasets.
- Keep source alignment in mind: neighboring unaligned VRT sources may disable
  multithreading to preserve deterministic nearest-neighbor output.
- Preserve mask semantics when selecting a mask as a regular band, building
  pansharpened overviews, or reading masked GTI overviews.
- Use overview source and creation options explicitly when importing or
  rebuilding overviews; COG cleanup has layout-aware behavior.

### Formats

- Confirm driver availability at runtime. Several legacy drivers were removed,
  some were later restored, and read/write capabilities continue to evolve.
- Distinguish GTiff, COG, and LIBERTIFF: their creation models, thread safety,
  compression support, nodata parsing, and alpha handling are not identical.
- Treat JPEG XL, JPEG 2000, HEIF/GeoHEIF, AVIF, WEBP, Lerc, and Grok support as
  dependency-sensitive.
- Validate creation and open options against the dataset-versus-layer scope;
  diagnostics may identify an option supplied at the wrong level.

## Vector quick reference

- Check spatial-filter and geometry mutation results; do not assume failure is
  silent or that polygon-building returns a single polygon.
- Preserve curve, Z, M, field domains, relationships, schema, and metadata only
  when the destination driver advertises the required capability.
- Expect conversion to fail when destination field creation fails unless skip
  behavior is explicitly requested.
- Use Arrow paths with awareness of validity-buffer contracts, ignored-field
  collisions, timestamp offsets, list types, string views, and selected fields.
- For Parquet and GeoParquet, account for Hive filters, partition metadata,
  geometry encodings, bounding-box column names, and large-list fields.
- Verify server-side versus local spatial intersection for service/database
  drivers, and check whether filters and counts are pushed down or evaluated
  client-side.
- Treat format identification as significant: GeoJSON, ESRIJSON, TileDB, WFS,
  STAC, and subdataset strings have tightened recognition rules.

## Multidimensional and cloud quick reference

- Discover arrays explicitly when drivers default to a limited listing; use
  raw-block information or classic-dataset bridges when the workflow needs it.
- Preserve slicing steps, indexing variables, strides, coordinate metadata,
  and geotransforms when working with multidimensional views.
- For Zarr, distinguish v2 codecs from v3 sharding, consolidated metadata,
  multiscales, georeferencing conventions, and Kerchunk reference stores.
- For GTI and STAC, validate SRS behavior, relative paths, asset metadata,
  south-up handling, URL rewriting, interleave, masks, and overview refresh.
- Scope HTTP and cloud options to the intended path. Authentication changes can
  invalidate cached directory state, and redirects may deliberately drop
  credentials.
- Use the appropriate cloud VSI handler for provider URLs and verify directory
  listing, empty-file synchronization, timestamps, retries, and size discovery.
- Keep permitted-filename restrictions in mind when using `/vsicurl?` header
  files.

## Diagnostic checklist

1. Record the exact GDAL library and command versions used by the failing path.
2. Identify the driver selected by `GDALOpenEx()` or the command's detailed
   identification output; exclude an unwanted driver explicitly when needed.
3. Capture open, dataset-creation, and layer-creation options separately.
4. Reproduce with explicit CRS, nodata, output type, resampling, and thread
   settings before assuming a driver defect.
5. Check whether input is a named file, VSI URL, subdataset, anonymous dataset,
   or pipeline output; these paths can exercise different behavior.
6. Close writable datasets explicitly so Arrow, Parquet, COG, and other
   deferred writers flush output and report close-time failures.
7. Read the relevant reference entry for the exact batch attribution and any
   dependency or platform condition.
