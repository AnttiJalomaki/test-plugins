# Ingestion and storage

Use this guide to reason about the separate state held by ingestion caches,
docstores, local storage contexts, and external vector stores.

## Treat persistence as a component map

Persisting an index's storage context saves its local stores:

```python
from llama_index.core import StorageContext, load_index_from_storage

index.storage_context.persist(persist_dir="./storage")
storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

That directory is not necessarily a complete index backup. With an external
vector store, it may contain only auxiliary state while the vectors remain
remote. The inverse is also incomplete: a remote vector collection can omit
node text, mappings, or document state required by the index.

Reloading local state does not verify either of these external properties:

- The remote collection still exists.
- Its vectors use an embedding model compatible with the application.

When several indexes share one persistence directory, loading can also require
the expected index ID. Record and test the complete set of local and remote
components needed for recovery.

## Understand ingestion cache identity

Cache reuse is keyed by each node-plus-transformation combination. A custom
transformation's behavior can change without changing its stable serialized
settings or hashes; in that case, a later run can silently reuse stale output.
Make cache invalidation part of custom-transformation changes.

Pipeline cache state can be saved and restored:

```python
pipeline.persist("./pipeline-state")
pipeline.load("./pipeline-state")
```

This cache state is separate from docstore state. The docstore is responsible
for tracking the identities and hashes used to classify documents as duplicate,
updated, or deleted.

## Select an execution path

Use the asynchronous pipeline entry point in async applications:

```python
nodes = await pipeline.arun(documents=documents, num_workers=4)
```

`num_workers` selects process-based parallelism for suitable synchronous
transformations. It is not automatically faster or valid. Provider clients and
non-picklable transformations can fail across process boundaries or add enough
overhead to make parallel execution slower.

## Make document identity stable

When a docstore and vector store are both attached, duplicate and update
detection uses hashes associated with stable `document.doc_id` or
`node.ref_doc_id` values. Preserve those identities across ingestion runs so
updates target the intended source document.

## Reserve deletion strategies for complete inventories

Some `DocstoreStrategy` modes delete documents that are absent from the current
run. Those modes assume that the run is an authoritative inventory of all
documents managed by the pipeline.

A partial crawl violates that assumption. If it is run with an absence-based
deletion mode, unrelated documents can be classified as removed. Use a strategy
compatible with partial input, or supply the complete authoritative set before
enabling deletion.

## Operational checklist

- Persist the local stores and separately account for any remote vector data.
- Confirm remote collection existence and embedding compatibility on restore.
- Select the expected index ID when a directory holds multiple indexes.
- Invalidate cached transformations when behavior changes without a hash change.
- Back up or restore docstore state independently from pipeline cache state.
- Confirm that transformations and clients are safe for process workers.
- Keep document and reference IDs stable across runs.
- Use absence-based deletion only with a complete input inventory.
