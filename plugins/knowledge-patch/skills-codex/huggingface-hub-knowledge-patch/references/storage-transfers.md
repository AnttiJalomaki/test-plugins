# Storage, cache, and transfer operations

## Central cache paths are immutable

The central cache stores file content once under `blobs` and presents revision
trees under `snapshots/{commit}` using links where the filesystem supports
them. One physical blob can therefore back paths in multiple snapshots.

Never modify a path returned from the central cache. Editing it can corrupt
shared content or invalidate more than one snapshot. Copy the file to a
working directory before modifying it.

## `local_dir` behavior

Passing `local_dir` materializes the selected files in that directory and
writes resume metadata under:

```text
.cache/huggingface
```

Exclude this metadata from packages and publications. A local directory is
convenient for editable material, but it provides less cross-project
deduplication than the central cache.

## Clean cache layers deliberately

Use supported cache commands to inspect and remove Hub content:

```bash
hf cache ls
hf cache rm
hf cache prune
```

Avoid deleting cache internals while active work may hold or create cache
state.

The Xet chunk cache is separate from Hub snapshots and repository refs:

- removing snapshots does not necessarily remove Xet chunks;
- clearing chunks does not update repository refs; and
- logging out leaves previously downloaded private bytes on disk.

Choose the cleanup target based on whether the goal is snapshot removal, Xet
chunk removal, credential removal, or deletion of private local content.

## Resumable large-folder uploads

`upload_large_folder` stores hashing, pre-upload, and commit progress in the
source folder's `.cache/huggingface`. A rerun can reuse finished work when it
targets the same folder and repository.

For reliable resumption:

- keep the progress metadata;
- use the same source folder and repository;
- do not modify source files while the operation runs; and
- rerun after interruption instead of discarding state.

The operation is not atomic. It can produce multiple repository commits, so
some work may be visible before the full folder finishes.

For an all-or-nothing release, upload to a staging branch or repository,
validate the complete result, and promote it only after validation.

## Process-local deferred uploads

Upload calls with `run_as_future=True` return futures and preserve queue order
for a given client. They are background tasks in the current process, not
durable jobs submitted to a remote worker.

The caller must:

- keep the process alive until the work finishes;
- retrieve each future's result so failures are observed; and
- avoid treating queue acceptance as successful upload completion.

Scheduled commit helpers have the same shutdown concern. Stop them explicitly
and verify their last commit before the job exits.

## Server-side copies in commits

`create_commit` accepts supported `CommitOperationCopy` operations alongside
add and delete operations. Eligible copies occur server-side, so the client
does not have to download and re-upload the bytes.

A copy still creates a repository commit. It is subject to the client's
supported source, destination, revision, and repository contexts; do not
assume an arbitrary cross-context copy is valid.

## Xet and legacy Git LFS

Xet-backed repositories expose a compatibility bridge for legacy Git LFS
clients. This permits interoperability but does not make Xet and LFS identical
in storage layout or performance behavior.

A generic Git clone can report success while materializing only repository
metadata or large-file pointers. Before consuming the checkout:

1. inspect its filter and pointer state;
2. verify that required large-file bytes were materialized; and
3. use a Hub download API when a normal clone does not resolve the content.
