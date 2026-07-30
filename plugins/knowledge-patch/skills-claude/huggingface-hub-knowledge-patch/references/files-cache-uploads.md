# Files, caches, and upload workflows

Use this reference when choosing a download destination, modifying downloaded
files, cleaning storage, uploading large directories, queuing background work,
copying repository files, or mixing Git LFS and Xet tooling.

## Central-cache paths are immutable inputs

The central Hub cache stores file content once under `blobs`. Revision trees
under `snapshots/{commit}` expose that content through links where the platform
supports them.

A returned snapshot path may therefore share storage with other snapshots.
Editing it can corrupt shared content or make several snapshots inconsistent.

Treat the returned path as read-only:

1. Download or resolve the cached file.
2. Copy it to a working directory.
3. Modify the copy.
4. Upload the intended result as a new repository change.

Do not use link count or apparent directory separation as proof that a cached
file is safe to mutate.

## `local_dir` materialization

Passing `local_dir` materializes selected files in that destination rather than
using a central snapshot path as the working tree. Transfer and resume metadata
is written beneath:

```text
.cache/huggingface
```

Keep that metadata when an interrupted transfer should be resumed. Exclude it
from packages, container contexts, publications, and repository commits unless
there is a specific operational reason to retain it there.

`local_dir` offers a convenient explicit destination, but expect less
cross-project deduplication than the central cache.

## Clean cache layers deliberately

Use supported commands to inspect and remove Hub cache content:

```bash
hf cache ls
hf cache rm
hf cache prune
```

Avoid deleting internal directories during active downloads or uploads. The
command surface understands Hub cache records and references better than an
ad hoc recursive delete.

The Xet chunk cache is separate from repository snapshots:

- removing Hub snapshots does not necessarily remove Xet chunks;
- clearing Xet chunks does not update repository refs;
- cleaning one layer is not evidence that the other meets a storage or data
  retention target.

Logging out also leaves previously downloaded private content on disk. Treat
credential removal and cached-data cleanup as independent operations.

## Resumable large-folder uploads

`upload_large_folder` persists progress for hashing, pre-upload, and commits in
the source folder's `.cache/huggingface`. A rerun against the same folder and
repository can reuse completed work.

For reliable resumption:

- keep the progress metadata available between attempts;
- retry with the same source folder and target repository;
- do not modify source files while the operation is running;
- preserve logs and surface failures rather than assuming every queued item
  completed.

Deleting the metadata discards reusable progress. Moving to a materially
different source tree can also invalidate assumptions behind that progress.

## Large-folder uploads are not one transaction

The operation may create multiple commits. A failure can therefore leave a
valid but incomplete sequence of remote changes.

For an all-or-nothing release:

1. Upload to a staging branch or staging repository.
2. Wait for all work to finish.
3. Validate file inventory and content.
4. Promote the validated state through an explicit repository workflow.

Do not present the target as complete merely because the upload call began or
because some commits are visible.

## Deferred uploads are process-local

Upload calls with `run_as_future=True` return futures. A client preserves queue
order for its own deferred calls, but these are background tasks in the current
process, not durable jobs managed remotely.

The caller must:

- keep the process alive;
- retain each returned future;
- retrieve each result so failures are observed;
- wait for required work before reporting success or exiting.

Process termination can abandon unfinished work. Queue order does not provide
durability and does not make a group of commits atomic.

Scheduled commit helpers have a similar shutdown boundary. Stop the helper and
verify its final commit before a job exits.

## Copy files without re-uploading

Supported clients accept `CommitOperationCopy` alongside add and delete
operations passed to `create_commit`.

An eligible copy:

- happens server-side without downloading and re-uploading file bytes;
- still produces a repository commit;
- is constrained by supported source and destination paths, revisions, and
  repository contexts.

Check those constraints before building a large copy plan. Include the copy in
the same commit operation set when it must be coordinated atomically with
supported adds or deletes.

## Xet and legacy Git LFS interoperability

Xet-backed repositories retain a compatibility bridge for legacy Git LFS
clients. This permits interoperability, but Xet and LFS do not have identical
storage layouts or performance behavior.

A successful generic Git clone may materialize repository metadata or pointer
files without fetching the large-file bytes.

After a clone:

- inspect whether LFS filters ran;
- check suspect files for pointer content;
- verify required large objects are actually materialized;
- use a Hub download API when the workflow needs dependable file
  materialization rather than generic Git metadata.

Do not treat clone success alone as proof that large model or dataset files are
present.

## Operational checklist

- Copy central-cache paths before editing.
- Keep `.cache/huggingface` only where resumable local work needs it.
- Exclude transfer metadata from published artifacts.
- Use `hf cache` commands and account for Hub and Xet layers separately.
- Preserve large-folder progress and freeze source files during upload.
- Stage and validate workflows that require all-or-nothing publication.
- Await every process-local future and stop scheduled helpers cleanly.
- Prefer `CommitOperationCopy` for eligible server-side copies.
- Verify large bytes after generic Git or LFS operations on Xet-backed
  repositories.
