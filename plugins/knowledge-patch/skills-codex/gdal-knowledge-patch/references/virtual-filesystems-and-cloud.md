# Virtual Filesystems and Cloud

VSI handlers, HTTP behavior, authentication, caching, redirects, paths, and cloud storage.

## AWS IAM Identity Center support in `/vsis3/`

*Batch 3.10.1*

The S3 virtual filesystem now supports AWS Single Sign-On (AWS IAM Identity Center), enabling `/vsis3/` access through that authentication system.

## AWS SSO and cloud-directory reads

*Batch 3.11.4*

AWS SSO authentication now uses the correct cache-file location and region parameter. `/vsis3/` directory reads honor path-specific options, and `/vsiaz/` `ReadDir()` works with `AZURE_NO_SIGN_REQUEST=YES`.

## Azure access-token options

*Batch 3.13.2*

The Azure/ADLS handler's option metadata now includes `AZURE_STORAGE_ACCESS_TOKEN` and `AZURE_STORAGE_SAS_TOKEN`.

## Cloud and configuration controls

*Batch 3.13.0*

`CPL_NULL_VALUE` passed to `CPLSetConfigOption()` explicitly masks an environment variable with a null value. `/vsicurl/` redirect-authorization policy can be path-specific, multi-range reads retry HTTP 429 and 5xx responses, and `/vsigs/` recognizes Google Cloud Run for GCE authentication.

## Cloud and `/vsicurl/` request controls

*Batch 3.11.0*

`VSICURL_QUERY_STRING` is a path-specific option, `/vsicurl?` URLs accept `header.<key>=<value>`, and `GDAL_HTTP_MAX_CACHED_CONNECTIONS` plus `GDAL_HTTP_MAX_TOTAL_CONNECTIONS` bound connection caching. `CPL_VSIL_CURL_CACHE_SIZE` and `CPL_VSIL_CURL_CHUNK_SIZE` accept memory units, `/vsicurl_streaming/` follows HTTP 303 redirects, and `AWS_S3_ENDPOINT` may include an `http://` or `https://` prefix.

## Cloud filesystem cache invalidation after authentication changes

*Batch 3.10.3*

When authentication parameters change, `/vsigs/` and `/vsiaz/` invalidate cached file and directory state instead of retaining results obtained with the previous credentials.

## Cloud-backed tiled formats

*Batch 3.12.0*

STACTA recognizes `gs://`, `az://`, and `azure://` URL templates, reads WEBP and JPEG XL tiles, and can retry failed `/vsicurl/` access through the matching cloud VSI handler. TileDB adds `/vsiaz/` support.

## Custom VSI handler compatibility

*Batch 3.13.0*

`VSIVirtualHandle::Read()` and `Write()` now take one `size_t` count instead of separate size and member counts, requiring custom handlers to update overrides. Handler installation can take a `shared_ptr`, and virtual handles add little-endian `ReadLSB()` and `WriteLSB()` helpers.

## Database and service driver capabilities

*Batch 3.11.0*

OpenFileGDB adds `CREATE_MULTIPATCH=YES` and accepts ZIP archives whose contents are directly at the top level; SQLite supports `SAVEPOINT` and runs `PRELUDE_STATEMENTS` after initialization and SpatiaLite loading. NGW adds HTTP timeouts/retries, filtered deletes, coded domains, COG and TMS web-map sources, and field alteration; PostgreSQL adds escaping for table names containing an opening parenthesis in `TABLES`, while MySQL removes its deprecated reconnect option for MySQL 8.0.34 and later.

## EC2 credentials for new `/vsis3/` connections

*Batch 3.11.5*

New `/vsis3/` connections using EC2 credentials no longer incorrectly go through the WebIdentity authentication path.

## Empty files in multithreaded cloud sync

*Batch 3.13.2*

`VSISync()` now includes empty files when multithreaded synchronization is used in either direction with cloud storage.

## GTI SQL sources and GeoTIFF metadata tags

*Batch 3.12.0*

GTI accepts a SQL request instead of only a layer or table name for selecting tile features, and STAC GeoParquet `s3://` references are translated to `/vsis3/`. GeoTIFF reads and writes the `GDAL_METADATA` TIFF tag, including supported `json:*` metadata domains.

## HTTP headers for GeoJSON-like drivers

*Batch 3.10.1*

GeoJSON-like drivers combine `GDAL_HTTP_HEADERS` with the driver-generated `Accept` header, so custom headers no longer displace the driver's content-negotiation header.

## HTTP retries for SSL connection timeouts

*Batch 3.10.3*

The CPL HTTP layer now retries errors reported as `SSL connection timeout`.

## HTTP-aware cloud filesystem operations

*Batch 3.11.1*

`/vsigs/` supports `UnlinkBatch()` and `GetFileMetadata()` when an OAuth2 bearer token is supplied through `GDAL_HTTP_HEADERS`. `/vsiswift/` forwards HTTP options for listing, while `/vsiwebhdfs/` forwards them for listing, deletion, and directory creation.

## ISIS3 PVL arrays and repeated keywords

*Batch 3.12.2*

ISIS3 PVL-to-JSON and JSON-to-PVL conversion now handles arrays whose values carry units and metadata containing repeated keywords.

## Lexical path normalization

*Batch 3.12.3*

The portability API adds `CPLLexicallyNormalize()` for normalizing file paths.

## New CPL and VSI helpers

*Batch 3.11.0*

New public helpers include `CPLIsInteractive()`, `CPLIsDebugEnabled()`, `VSIGlob()`, `VSIMove()`, `CPLGetKnownConfigOptions()`, `CPLErrorOnce()`, and `CPLDebugOnce()`, along with safe path-manipulation functions. C++ callers also gain `CPLTurnFailureIntoWarningBackuper`, `CPLErrorAccumulator`, and `CPLQuietWarningsErrorHandler`.

## Portability-layer debug and URL handling

*Batch 3.10.1*

`CPLDebug` accepts `YES`, `TRUE`, and `1`. `CPLGetPath()` and `CPLGetDirname()` now handle `/vsicurl?` and URL-encoded paths, while `CPLFormFilename()` strips a relative `../...` path when it is joined to an absolute path.

## Redirects and file sizes in curl-backed VSI

*Batch 3.12.2*

`/vsicurl_streaming/` now sets file sizes correctly, and `/vsicurl/` handles HTTP 302 responses to `HEAD` requests. Authentication sent to an original URL is no longer propagated to an S3-like redirect.

## Repeatable VSI handle closure

*Batch 3.11.2*

Unix, Win32, sparse-file, and archive VSI handles may have `Close()` called more than once, and their destructors also close them.

## S3 authentication, directory buckets, and cloud paths

*Batch 3.12.0*

`/vsis3/` supports S3 directory buckets and AWS `credential_process` authentication. Cloud VSI paths squash `/./` and `/../` by default; set `GDAL_HTTP_PATH_VERBATIM` to preserve them, and use the new `CACHE=ON/OFF` option of `VSIFOpenEx2L()` to control post-close caching for `/vsicurl/`-style files.

## Single-file RAR reads

*Batch 3.11.4*

`/vsirar/` no longer returns a negative read result when opening an archive made from a single file through a path such as `/vsirar/the.rar`.

## TileDB identification on S3

*Batch 3.13.1*

The TileDB driver identifies `/vsis3/` paths only when they have a `.tdb` extension or no extension, avoiding claims on other suffixed objects.

## Top-level ISIS3 metadata from GeoTIFF

*Batch 3.12.2*

For the `json:ISIS3` metadata domain, `GetMetadataItem(<top-level-key>, json:ISIS3)` now returns the requested subset rather than the complete JSON object.

## `/vsicurl/` header files and Nginx listings

*Batch 3.13.2*

The `header_file` value in a `/vsicurl?` URL is now restricted to permitted filenames. `/vsicurl/` also reports correct file sizes from Nginx directory listings generated with `autoindex_exact_size off`.

## `/vsicurl/` size discovery without `Content-Length`

*Batch 3.12.4*

When an initial `HEAD` response advertises `Accept-ranges: bytes` but omits `Content-Length`, `/vsicurl/` retries with a limited-range `GET`.
