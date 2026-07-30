# Lifecycle, snapshots, reindexing, and repositories

Use this reference when changing ILM or data-stream lifecycle, moving data,
managing transforms, or validating repository behavior.

## Reindexing and migration

### Data-stream migration controls

In 9.0.0, Elasticsearch adds REST and action support to create an index from a
source index and to query, cancel, and throttle data-stream migration
reindexing with `requests_per_second`. `_create_from` removes index blocks by
default; control that with `remove_index_block`. Migration ignores closed
source indices and filters deprecated destination settings.

### Remote reindex controls

In 9.3.0, remote reindex accepts a convenience API-key parameter.

In 9.4.0, remote reindexing gains a blocklist setting.

## ILM and index controls

### Searchable-snapshot replication

In 9.0.0, the ILM `searchable_snapshot` action accepts `replicate_for`.

### Skipping ILM per index

In 9.1.0, `index.lifecycle.skip` tells ILM not to process a particular index:

```http
PUT my-index/_settings
{
  "index.lifecycle.skip": true
}
```

### Time-series unfollow ordering

In 9.1.0, ILM injects an unfollow action before downsampling when necessary. A
follower index waits until the leader's time-series end time has passed before
unfollowing.

### Removing an index block

In 9.1.0, the remove-block API deletes a specified block:

```http
DELETE /my-index/_block/write
```

### Lifecycle response fields

In 9.2.0, ILM explain responses add `age_in_millis`, and the read-only action
sets `indexing_complete` to `true`. `_resolve/index` can filter by index mode
and includes the mode in its response.

### Downsampling controls

In 9.3.0, data-stream lifecycle and ILM can choose among downsampling methods,
and the Downsample API adds another sampling method. Force merge moves from the
downsampling request into the ILM action, where it can be disabled.

## Transforms and machine learning

### Transform migration and deletion

In 9.0.0, transforms gain upgrade mode, automatic migration of
`max_page_search_size`, and `extended_stats`. With `delete_dest_index=true`,
deleting a transform whose destination is an alias deletes the alias's write
index.

### Preview and shutdown controls

In 9.3.0, transforms add a preview-index request, and the stop-datafeed API
accepts `close_job`.

## Snapshot and archive compatibility

### N-2 archives

In 9.0.0, archive and searchable-snapshot indices are allowed from N-2
versions, including supported 7.x segment cases used as archives in 8.x or
9.x.

### Snapshot filtering

In 9.1.0, the get snapshots API accepts a `state` query parameter.

## S3 repositories

### Metadata service support

In 9.0.0, `repository-s3` supports IMDSv2.

### AWS SDK v2 migration

In 8.19.0, `repository-s3` moves from AWS SDK v1 to v2. The SDKs differ in
behavior and configuration, so test production repository settings before
upgrading.

### Conditional writes and connection lifetime

In 9.2.0, S3 repositories use conditional writes to prevent accidental object
overwrites and possible repository corruption, including on fully
S3-compatible storage. The S3 connection maximum idle time is configurable.

### API timeouts

In 9.3.0, S3 repositories gain an API-call timeout setting.
