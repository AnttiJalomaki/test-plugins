# Cloud, VSI, builds, and bindings

Use this reference for the task areas below. Batch labels identify when each behavior entered the covered compatibility history.

## Cloud, HTTP, and virtual filesystems

### AWS IAM Identity Center support in `/vsis3/`

*Batch: 3.10.1*

The S3 virtual filesystem now supports AWS Single Sign-On (AWS IAM Identity Center), enabling `/vsis3/` access through that authentication system.

### Portability-layer debug and URL handling

*Batch: 3.10.1*

`CPLDebug` accepts `YES`, `TRUE`, and `1`. `CPLGetPath()` and `CPLGetDirname()` now handle `/vsicurl?` and URL-encoded paths, while `CPLFormFilename()` strips a relative `../...` path when it is joined to an absolute path.

### HTTP headers for GeoJSON-like drivers

*Batch: 3.10.1*

GeoJSON-like drivers combine `GDAL_HTTP_HEADERS` with the driver-generated `Accept` header, so custom headers no longer displace the driver's content-negotiation header.

### HTTP retries for SSL connection timeouts

*Batch: 3.10.3*

The CPL HTTP layer now retries errors reported as `SSL connection timeout`.

### Cloud filesystem cache invalidation after authentication changes

*Batch: 3.10.3*

When authentication parameters change, `/vsigs/` and `/vsiaz/` invalidate cached file and directory state instead of retaining results obtained with the previous credentials.

### Cloud and `/vsicurl/` request controls

*Batch: 3.11.0*

`VSICURL_QUERY_STRING` is a path-specific option, `/vsicurl?` URLs accept `header.<key>=<value>`, and `GDAL_HTTP_MAX_CACHED_CONNECTIONS` plus `GDAL_HTTP_MAX_TOTAL_CONNECTIONS` bound connection caching. `CPL_VSIL_CURL_CACHE_SIZE` and `CPL_VSIL_CURL_CHUNK_SIZE` accept memory units, `/vsicurl_streaming/` follows HTTP 303 redirects, and `AWS_S3_ENDPOINT` may include an `http://` or `https://` prefix.

### New CPL and VSI helpers

*Batch: 3.11.0*

New public helpers include `CPLIsInteractive()`, `CPLIsDebugEnabled()`, `VSIGlob()`, `VSIMove()`, `CPLGetKnownConfigOptions()`, `CPLErrorOnce()`, and `CPLDebugOnce()`, along with safe path-manipulation functions. C++ callers also gain `CPLTurnFailureIntoWarningBackuper`, `CPLErrorAccumulator`, and `CPLQuietWarningsErrorHandler`.

### HTTP-aware cloud filesystem operations

*Batch: 3.11.1*

`/vsigs/` supports `UnlinkBatch()` and `GetFileMetadata()` when an OAuth2 bearer token is supplied through `GDAL_HTTP_HEADERS`. `/vsiswift/` forwards HTTP options for listing, while `/vsiwebhdfs/` forwards them for listing, deletion, and directory creation.

### Repeatable VSI handle closure

*Batch: 3.11.2*

Unix, Win32, sparse-file, and archive VSI handles may have `Close()` called more than once, and their destructors also close them.

### AWS SSO and cloud-directory reads

*Batch: 3.11.4*

AWS SSO authentication now uses the correct cache-file location and region parameter. `/vsis3/` directory reads honor path-specific options, and `/vsiaz/` `ReadDir()` works with `AZURE_NO_SIGN_REQUEST=YES`.

### Single-file RAR reads

*Batch: 3.11.4*

`/vsirar/` no longer returns a negative read result when opening an archive made from a single file through a path such as `/vsirar/the.rar`.

### MongoDB C++ driver 4 compatibility

*Batch: 3.11.4*

The MongoDB driver builds against `mongo-cpp-driver` version 4 and later.

### EC2 credentials for new `/vsis3/` connections

*Batch: 3.11.5*

New `/vsis3/` connections using EC2 credentials no longer incorrectly go through the WebIdentity authentication path.

### S3 authentication, directory buckets, and cloud paths

*Batch: 3.12.0*

`/vsis3/` supports S3 directory buckets and AWS `credential_process` authentication. Cloud VSI paths squash `/./` and `/../` by default; set `GDAL_HTTP_PATH_VERBATIM` to preserve them, and use the new `CACHE=ON/OFF` option of `VSIFOpenEx2L()` to control post-close caching for `/vsicurl/`-style files.

### Cloud-backed tiled formats

*Batch: 3.12.0*

STACTA recognizes `gs://`, `az://`, and `azure://` URL templates, reads WEBP and JPEG XL tiles, and can retry failed `/vsicurl/` access through the matching cloud VSI handler. TileDB adds `/vsiaz/` support.

### Redirects and file sizes in curl-backed VSI

*Batch: 3.12.2*

`/vsicurl_streaming/` now sets file sizes correctly, and `/vsicurl/` handles HTTP 302 responses to `HEAD` requests. Authentication sent to an original URL is no longer propagated to an S3-like redirect.

### Lexical path normalization

*Batch: 3.12.3*

The portability API adds `CPLLexicallyNormalize()` for normalizing file paths.

### `/vsicurl/` size discovery without `Content-Length`

*Batch: 3.12.4*

When an initial `HEAD` response advertises `Accept-ranges: bytes` but omits `Content-Length`, `/vsicurl/` retries with a limited-range `GET`.

### Custom VSI handler compatibility

*Batch: 3.13.0*

`VSIVirtualHandle::Read()` and `Write()` now take one `size_t` count instead of separate size and member counts, requiring custom handlers to update overrides. Handler installation can take a `shared_ptr`, and virtual handles add little-endian `ReadLSB()` and `WriteLSB()` helpers.

### Cloud and configuration controls

*Batch: 3.13.0*

`CPL_NULL_VALUE` passed to `CPLSetConfigOption()` explicitly masks an environment variable with a null value. `/vsicurl/` redirect-authorization policy can be path-specific, multi-range reads retry HTTP 429 and 5xx responses, and `/vsigs/` recognizes Google Cloud Run for GCE authentication.

### Single-file VSI timestamps

*Batch: 3.13.1*

`gdal vsi list` displays the last-modification timestamp when its target is a single file.

### TileDB identification on S3

*Batch: 3.13.1*

The TileDB driver identifies `/vsis3/` paths only when they have a `.tdb` extension or no extension, avoiding claims on other suffixed objects.

### Azure access-token options

*Batch: 3.13.2*

The Azure/ADLS handler's option metadata now includes `AZURE_STORAGE_ACCESS_TOKEN` and `AZURE_STORAGE_SAS_TOKEN`.

### `/vsicurl/` header files and Nginx listings

*Batch: 3.13.2*

The `header_file` value in a `/vsicurl?` URL is now restricted to permitted filenames. `/vsicurl/` also reports correct file sizes from Nginx directory listings generated with `autoindex_exact_size off`.

### Empty files in multithreaded cloud sync

*Batch: 3.13.2*

`VSISync()` now includes empty files when multithreaded synchronization is used in either direction with cloud storage.

### MRF cache configuration rename

*Batch: 3.13.2*

The MRF caching configuration replaces `MRF_BYPASSCACHING` with `MRF_ENABLE_CACHING`; deployments setting the former variable must migrate to the latter.

## Builds, packaging, and dependencies

### Building with Poppler 25.02

*Batch: 3.10.2*

GDAL 3.10.2 fixes compilation against Poppler 25.02.00.

### Debian Python installation with an external Python

*Batch: 3.10.2*

On Debian, the Python bindings' install target now works with a Python version not provided by Debian.

### Windows builds with `WIN32_LEAN_AND_MEAN`

*Batch: 3.10.3*

GDAL now builds when `WIN32_LEAN_AND_MEAN` is defined.

### CMake and packaged-build controls

*Batch: 3.11.0*

The CMake package now exports GDAL library targets, exports `GDAL_DEBUG` publicly for debug builds, and offers `USE_PRECOMPILED_HEADERS` with a default of `OFF`. Ubuntu `ubuntu-full` amd64 images can optionally add the Oracle, ECW, and MrSID drivers, which remain disabled by default.

### Poppler 25.10 build compatibility

*Batch: 3.11.5*

GDAL now builds with Poppler 25.10 while retaining compatibility with older Poppler versions.

### Build-time feature gates and public raster headers

*Batch: 3.12.0*

CMake adds `GDAL_ENABLE_ALGORITHMS` to omit algorithms beneath `gdal`, plus `GDAL_VRT_ENABLE_RAWRASTERBAND` to compile out raw VRT bands; the latter also exists as a runtime configuration option. Raster implementation types are now available through installed public C++ headers such as `gdal_dataset.h`, `gdal_rasterband.h`, `gdal_geotransform.h`, and `gdal_raster_cpp.h`.

### Python no-GIL builds and algorithm progress

*Batch: 3.12.1*

The Python bindings support Python 3.13 and later free-standing/no-GIL builds. Dynamically generated `gdal.alg.*()` methods also accept a `progress` keyword argument.

### Build compatibility with newer dependencies

*Batch: 3.12.2*

Arrow and Parquet builds work with libarrow 23.0 when precompiled headers are enabled, the PDF driver supports Poppler 26.01.0, and LIBKML builds with Boost 1.90, Clang 21, and C++23.

### Poppler and parallel HDF5 builds

*Batch: 3.12.3*

The PDF driver builds with Poppler 26.02.0, and the build system properly supports parallel HDF5.

### Build compatibility

*Batch: 3.12.4*

Builds configured with `-DGDAL_ENABLE_ALGORITHMS=OFF` now succeed. The PDF driver builds against Poppler 26.04.00, and HDF5 builds tolerate libhdf5 2.1 headers that redefine `_POSIX_C_SOURCE`.

### Build compatibility and JP2Grok dependency

*Batch: 3.13.1*

GDAL builds against Poppler 26.06.00, and the JP2Grok driver now requires Grok 20.3.2 or newer.

### Build configuration and dependency requirements

*Batch: 3.13.2*

With `BUILD_PYTHON_BINDINGS=OFF`, CMake no longer searches for Python. Builds support CMake 4.4, Poppler 26.08 development versions, and SWIG 4.5 development versions; the JP2Grok driver now requires `libjp2grok` 20.3.5 or newer.

### Python packaging with recent setuptools

*Batch: 3.13.2*

When setuptools 77 or newer is used, the bindings declare Python 3.9 as their minimum because those setuptools versions do not support Python 3.8.

## Language bindings

### Binding-level virtual filesystem support

*Batch: 3.11.0*

SWIG bindings add `Driver.CreateVector()`, and C# adds `VSIGetMemFileBuffer`. Python adds `osgeo.gdal.VSIFile` and `osgeo.gdal_fsspec`, whose import registers GDAL VSI handlers as fsspec `AbstractFileSystem` implementations.
