# Ingestion and Storage

## Choose the Ingestion Surface

`VectorStoreIndex.from_documents` is concise for simple index construction.
`IngestionPipeline` is the maintainable surface when ingestion needs:

- an explicit and reusable transformation chain;
- transformation caching;
- duplicate, update, and deletion handling through a docstore;
- direct insertion into a vector store;
- controlled synchronous, asynchronous, or process-worker execution.

Attach the docstore and vector store deliberately. The docstore and pipeline
cache solve different problems and must not be treated as interchangeable
state.

## Cache Identity and Persistence

Cache reuse is keyed by each node together with each transformation. Stable
serialization or hashes must change when custom behavior changes. Otherwise a
modified custom transformation can silently reuse output produced by its old
implementation.

Pipeline cache state can be saved and restored:

```python
pipeline.persist("./pipeline-state")
pipeline.load("./pipeline-state")
```

This state is separate from docstore state. The cache remembers transformation
results; the docstore tracks document identity and hashes used to classify
duplicate, updated, and deleted documents. Persist, restore, invalidate, and
test both according to their separate roles.

## Document Identity and Update Strategies

Stable `document.doc_id` or `node.ref_doc_id` values connect generated nodes to
source documents. With a docstore and vector store attached, hashes under
these identifiers drive duplicate and update detection.

Changing identifiers between runs turns updates into apparently unrelated
documents. Derive identifiers from a stable source key rather than crawl
order, a transient local path, or randomly regenerated values.

Absence-based `DocstoreStrategy` modes assume that one run is an authoritative
inventory. They can delete documents not present in that run. A partial crawl,
tenant shard, time window, or retry subset is not a complete inventory and
must not use such a strategy against shared state.

For partial input, choose a strategy that only handles presented documents or
partition the docstore and vector store so the input is authoritative for its
partition.

## Async and Process-Worker Execution

Use the asynchronous path from async application code:

```python
nodes = await pipeline.arun(documents=documents)
```

`num_workers` enables process-based parallelism for suitable synchronous
transformations:

```python
nodes = await pipeline.arun(
    documents=documents,
    num_workers=4,
)
```

Process workers require their inputs and transformations to cross a process
boundary. Provider clients and non-picklable transformations can fail in this
mode. Process startup and serialization also mean a small or I/O-bound
workload can become slower rather than faster. Benchmark representative
workloads and keep provider-owned async clients on an async path where
appropriate.

## What `StorageContext.persist()` Saves

Local persistence and an external vector store form a distributed logical
backup. Calling:

```python
index.storage_context.persist(persist_dir="./storage")
```

saves local stores known to the storage context. When vectors live in an
external service, the persist directory may contain only auxiliary state.
Conversely, the remote collection may contain vectors while omitting node
text, document mappings, or other local state needed by the application.

A robust backup or migration therefore accounts for:

- the local docstore, index store, graph store, and other configured stores;
- the remote vector collection and its service-native backup mechanism;
- index and collection identifiers;
- the embedding implementation and configuration;
- package and schema compatibility.

## Reloading Is Not Validation

A standard local reload is:

```python
from llama_index.core import StorageContext, load_index_from_storage

storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

This does not verify that a remote collection still exists, contains the
expected vectors, or uses embeddings compatible with the current embedder.
Run a representative retrieval check and independently validate the remote
collection.

If one persist directory contains multiple indexes,
`load_index_from_storage()` may need the expected index ID. Treat that ID as
part of deployment and restore configuration rather than relying on an
implicit single-index layout.

## Embedding Changes

Persisted vectors encode both an embedding implementation and its
configuration. Changing either can change dimensions or semantic space.
Rebuild the index or prove compatibility; do not rely on successful loading.

A verification plan should include:

1. ingesting representative documents with stable IDs;
2. rerunning unchanged documents to check deduplication;
3. updating content to check replacement behavior;
4. deleting within an authoritative inventory;
5. persisting and restoring both cache and docstore state;
6. validating the remote collection;
7. retrieving known queries after reload.
