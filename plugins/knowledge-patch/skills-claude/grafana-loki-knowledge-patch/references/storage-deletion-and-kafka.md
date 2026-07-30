# Storage, deletion, and Kafka

## Thanos object-store clients

Loki 3.4.0 moves object-store access to the shared Thanos client and adds Swift
support through `thanos.io/objstore`. Related storage controls include:

- an Alibaba Cloud OSS request timeout;
- age-based suppression of cache writeback for fetched chunks;
- in-memory-only TSDB index creation.

Thanos object storage accepts dashes in `storage_prefix` as of 3.6.0.

## S3, GCS, Swift, and MinIO compatibility

- The S3 chunk delimiter is configurable in 3.5.0 for MinIO on Windows.
- GCS can target a custom endpoint in 3.5.0.
- The Operator can configure a TLS CA for Swift in 3.5.0.
- The S3 client preserves a region already supplied by the configuration
  chain in 3.7.0.
- Loki 3.7.2 adds a SHA-256 checksum to `PutObject` calls for S3 Object Lock
  buckets.
- Loki 3.7.4 fixes index filenames when the legacy S3 client uses
  `chunk_delimiter`.

## Chart storage validation

In 3.7.0, chunk bucket names are not required when storage uses an S3 URL,
MinIO, or local disk. Ruler bucket names are optional when ruler storage is
local. Do not add placeholder buckets merely to satisfy older chart
validation.

## SQLite delete requests

Delete requests can use SQLite storage as of 3.5.0. For this backend, Loki uses
each request's stored completion time to reduce the requests considered during
query-time filtering.

## Horizontally scalable deletion

The experimental horizontally scalable compactor in 3.6.0 delegates queued
deletion work to workers. It is intended to scale large deletion jobs and
backlogs. Index compaction and retention remain in the main singleton
Compactor; scaling deletion workers does not distribute those duties.

## Object-backed deletion markers

The compactor can store chunk-deletion markers in object storage rather than
local disk in 3.7.0. Loki 3.7.4 also repairs delete requests made through the
Thanos object-store client when its backend is the filesystem.

## Kafka ingestion and block building

- Kafka-backed ingestion supports tenant-specific topics in 3.5.0.
- Components can consume Kafka records and maintain multiple Kafka clients in
  3.6.0.
- The Helm chart exposes `block_builder` configuration for deploying this path
  in 3.6.0.

Keep tenant topic selection, client multiplicity, and block-builder deployment
configuration aligned; configuring one does not implicitly deploy the others.

## Index gateway

Index-gateway clients can use shuffle sharding in 3.7.0. Select shard sizing
with tenant isolation and index-gateway capacity in mind.

## Caches

The Helm chart supports external Memcached and an L2 chunks cache in 3.6.0.
Separately, storage configuration can suppress cache writeback for fetched
chunks based on age in 3.4.0.
