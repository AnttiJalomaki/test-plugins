# Bindings, Build, and Packaging

Build-system controls, dependency compatibility, installed headers, and language-binding behavior.

## Binding behavior

*Batch 3.13.0*

Python `Dataset.AdviseRead()` and `Band.AdviseRead()` accept keywords, with dataset calls defaulting to all bands; algorithm functions accept visible and hidden argument aliases, and `Feature.SetField()` accepts NumPy values. Java exposes full and partial `/vsicurl/` cache clearing, while SWIG adds the missing relationship capability constants.

## Binding-level virtual filesystem support

*Batch 3.11.0*

SWIG bindings add `Driver.CreateVector()`, and C# adds `VSIGetMemFileBuffer`. Python adds `osgeo.gdal.VSIFile` and `osgeo.gdal_fsspec`, whose import registers GDAL VSI handlers as fsspec `AbstractFileSystem` implementations.

## Build compatibility

*Batch 3.12.4*

Builds configured with `-DGDAL_ENABLE_ALGORITHMS=OFF` now succeed. The PDF driver builds against Poppler 26.04.00, and HDF5 builds tolerate libhdf5 2.1 headers that redefine `_POSIX_C_SOURCE`.

## Build compatibility and JP2Grok dependency

*Batch 3.13.1*

GDAL builds against Poppler 26.06.00, and the JP2Grok driver now requires Grok 20.3.2 or newer.

## Build compatibility with newer dependencies

*Batch 3.12.2*

Arrow and Parquet builds work with libarrow 23.0 when precompiled headers are enabled, the PDF driver supports Poppler 26.01.0, and LIBKML builds with Boost 1.90, Clang 21, and C++23.

## Build configuration and dependency requirements

*Batch 3.13.2*

With `BUILD_PYTHON_BINDINGS=OFF`, CMake no longer searches for Python. Builds support CMake 4.4, Poppler 26.08 development versions, and SWIG 4.5 development versions; the JP2Grok driver now requires `libjp2grok` 20.3.5 or newer.

## Build-time feature gates and public raster headers

*Batch 3.12.0*

CMake adds `GDAL_ENABLE_ALGORITHMS` to omit algorithms beneath `gdal`, plus `GDAL_VRT_ENABLE_RAWRASTERBAND` to compile out raw VRT bands; the latter also exists as a runtime configuration option. Raster implementation types are now available through installed public C++ headers such as `gdal_dataset.h`, `gdal_rasterband.h`, `gdal_geotransform.h`, and `gdal_raster_cpp.h`.

## Building with Poppler 25.02

*Batch 3.10.2*

GDAL 3.10.2 fixes compilation against Poppler 25.02.00.

## C# spatial-reference matching

*Batch 3.11.1*

The C# bindings add `SpatialReference.FindMatches`.

## CMake and packaged-build controls

*Batch 3.11.0*

The CMake package now exports GDAL library targets, exports `GDAL_DEBUG` publicly for debug builds, and offers `USE_PRECOMPILED_HEADERS` with a default of `OFF`. Ubuntu `ubuntu-full` amd64 images can optionally add the Oracle, ECW, and MrSID drivers, which remain disabled by default.

## Debian Python installation with an external Python

*Batch 3.10.2*

On Debian, the Python bindings' install target now works with a Python version not provided by Debian.

## Driver capability and algorithm metadata

*Batch 3.12.0*

Drivers can advertise maximum string length and the new append, upsert, close-time visibility, reopen-after-write, and read-after-delete capabilities. Algorithm consumers can retrieve typed default arguments through the new C/SWIG getters, while algorithm implementers gain dedicated geometry-type, append-layer, overwrite-layer, absolute-path, stdout, hidden, and deprecation helpers.

## Driver exclusion and algorithm metadata

*Batch 3.13.0*

An allowed-driver entry prefixed with `-` now excludes that driver in `GDALOpenEx()`. Algorithm front ends can inspect pipeline-step availability, direct and aggregate argument dependencies, mutual-dependency groups, duplicate-value allowance, and maximum character counts.

## Embedded resources and VRT expression dependencies

*Batch 3.11.0*

CMake adds `EMBED_RESOURCE_FILES` and `USE_ONLY_EMBEDDED_RESOURCE_FILES` for compiling resource files into libgdal. `muparser` is strongly recommended as a build and runtime dependency for C++ VRT expressions; header-only `exprtk` may be added alongside it for advanced expressions, at an approximately 8 MB library-size cost.

## Java dataset closure

*Batch 3.11.4*

Closing a dataset obtained with `Band.GetDataset().Close()` no longer causes a double free.

## MongoDB C++ driver 4 compatibility

*Batch 3.11.4*

The MongoDB driver builds against `mongo-cpp-driver` version 4 and later.

## Newly public headers

*Batch 3.13.0*

The installed headers now include `gdal_mem.h`, which exposes the `MEMCreate()` C API, plus `gdal_thread_pool.h` and `ogr_refcountedptr.h`.

## OGR APIs and Arrow time values

*Batch 3.11.0*

`OGRFieldDefn::SetGenerated()`/`IsGenerated()` marks generated fields, `OSRGetAuthorityListFromDatabase()` lists CRS authorities from PROJ, and `OGR_GT_GetSingle()` is available through SWIG. `OGRLayer::GetArrowStream()` adds `DATETIME_AS_STRING=YES/NO`; `ogr2ogr` uses it to preserve source time zones and can now transfer dataset relationships when the target supports them.

## Poppler 25.10 build compatibility

*Batch 3.11.5*

GDAL now builds with Poppler 25.10 while retaining compatibility with older Poppler versions.

## Poppler and parallel HDF5 builds

*Batch 3.12.3*

The PDF driver builds with Poppler 26.02.0, and the build system properly supports parallel HDF5.

## Python color interpretation during translation

*Batch 3.10.1*

The Python `gdal.Translate()` binding adds a `colorInterpretation` argument; the similar argument in `gdal.TileIndex()` also receives a correctness fix.

## Python no-GIL builds and algorithm progress

*Batch 3.12.1*

The Python bindings support Python 3.13 and later free-standing/no-GIL builds. Dynamically generated `gdal.alg.*()` methods also accept a `progress` keyword argument.

## Python open-option parsing

*Batch 3.13.1*

Python methods such as `gdal.VectorTranslate()` recognize the list form `options=["-oo", "FOO=BAR"]`.

## Python packaging with recent setuptools

*Batch 3.13.2*

When setuptools 77 or newer is used, the bindings declare Python 3.9 as their minimum because those setuptools versions do not support Python 3.8.

## Python raster arrays and accepted inputs

*Batch 3.11.0*

Python adds `Dataset.ReadAsMaskedArray()`, `mask_resample_alg` on `ReadAsArray()` methods, and the `-epo`/`-eco` translation controls; `gdal.VectorTranslate()` gains `relatedFieldNameMatch`. `osr.SpatialReference()` accepts a CRS definition, `Driver.Create()` accepts NumPy types, `Driver.Rename()`/`CopyFiles()` accept `os.PathLike`, and `GDAL_PYTHON_BINDINGS_WITHOUT_NUMPY` accepts `YES/1/ON/TRUE` or `NO/0/OFF/FALSE`.

## Python raster iteration and Boolean arrays

*Batch 3.12.0*

Python adds `Band.BlockWindows()`, permits a band as `Driver.CreateCopy()` input, maps NumPy Boolean types to GDAL types, and avoids promoting Boolean arrays to `float64` when writing. Configuration-option values are coerced to strings.

## Range-domain validation and binding errors

*Batch 3.11.1*

Python's `ogr.CreateRangeFieldDomain()` and `ogr.CreateRangeFieldDomainDateTime()` correctly handle `None` bounds. The OpenFileGDB writer rejects range domains missing either bound, and SWIG `AddFieldDomain()` surfaces failures as errors or exceptions.

## SWIG feature-definition ownership

*Batch 3.12.1*

`Feature.GetDefnRef` now increments the returned `FeatureDefn` reference count.

## Vector geometry and SQL APIs

*Batch 3.13.0*

The GeoPackage and SQLite dialects add `ST_Hilbert()`, and geometry APIs add polygon-based concave hull generation plus invalidity-reason retrieval in C, C++, and SWIG. `ExportToKML()` now fails rather than emitting coordinates with invalid latitudes.

## Windows builds with `WIN32_LEAN_AND_MEAN`

*Batch 3.10.3*

GDAL now builds when `WIN32_LEAN_AND_MEAN` is defined.

## Zarr v3 sharding, metadata, and georeferencing

*Batch 3.13.0*

Zarr v3 gains read, update, and creation of consolidated metadata; read/write support for `sharding_indexed`, `crc32c`, and variable-length UTF-8; and NumPy datetime/timedelta extension types. Multiscales map to GDAL overviews, `spatial` and `proj` conventions can be read or written with `GEOREFERENCING_CONVENTION=SPATIAL_PROJ`, and multidimensional overview building supports arrays with more than two dimensions.

## Zero-stride Python array writes

*Batch 3.11.4*

Python `Dataset.WriteArray()` and `Band.WriteArray()` correctly write arrays containing a zero stride.
