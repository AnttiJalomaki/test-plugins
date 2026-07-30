# Storage, Deletion, and Compaction

## Shared Thanos object-store clients

Loki moved object-store access to the shared Thanos client in 3.4.0 and added
Swift support through `thanos.io/objstore`. The storage surface also gained:

- a request timeout for Alibaba Cloud OSS;
- age-based suppression of cache writeback for chunks fetched from storage;
  and
- an in-memory-only mode for TSDB index creation.

When migrating a backend, validate credentials, endpoints, timeouts, prefix
handling, cache policy, and existing-object reads through the selected client.

Thanos object storage accepts dashes in `storage_prefix` as of 3.6.0.

## Object-store value changes and compatibility

The Helm value is `object_store.storage_prefix`, not the former
`object_store.prefix`, as of 3.5.0.

The S3 chunk delimiter is configurable for MinIO on Windows, and GCS can use a
custom endpoint (3.5.0).

The S3 client preserves a region already supplied by its configuration chain
as of 3.7.0. Starting in 3.7.2, `PutObject` includes a SHA-256 checksum for
Object Lock buckets. Loki 3.7.4 fixes index filenames when the legacy S3 client
uses `chunk_delimiter`.

## SQLite delete-request storage

Delete requests can use SQLite storage as of 3.5.0. With this backend, Loki
uses each request's stored completion time to reduce the set considered for
query-time filtering. Preserve completion timestamps when operating or
migrating the database.

## Horizontally scalable deletion workers

The experimental horizontally scalable compactor introduced in 3.6.0
delegates queued deletion work to workers. This scales large deletion jobs and
backlogs horizontally. It does not distribute every compactor responsibility:
index compaction and retention remain in the main singleton Compactor.

Size worker capacity for deletion throughput while continuing to protect the
singleton's availability for compaction and retention.

## Object-backed chunk-deletion markers

The compactor can keep chunk-deletion markers in object storage instead of
local disk as of 3.7.0. Configure lifecycle, permissions, and durability for
these marker objects as part of the deletion path.

Loki 3.7.4 repairs delete requests made through the Thanos object-store client
when its filesystem backend is selected. Include that maintenance fix when
testing this combination.

## Index-gateway shuffle sharding

Index-gateway clients can use shuffle sharding as of 3.7.0. Configure and test
the shard behavior from the clients rather than assuming every tenant reaches
every index gateway.

## Chart-generated storage configuration

Since 3.6.0, the Loki chart exposes the complete storage configuration. It can
bypass generated S3, GCS, and Azure settings, and it supports separate ruler
storage. Use either chart-generated settings or the bypass path deliberately;
inspect the final Loki configuration for duplicate or conflicting blocks.

The chart no longer requires chunk bucket names when using an S3 URL, MinIO,
or local disk (3.7.0). Ruler bucket names are optional with local ruler
storage. These relaxed validations do not remove backend-specific credential,
path, or permission requirements.
