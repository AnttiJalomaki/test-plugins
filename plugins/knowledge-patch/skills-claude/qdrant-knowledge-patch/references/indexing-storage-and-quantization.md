# Indexing, Storage, and Quantization

## HNSW construction

### Vulkan GPU indexing

Qdrant can build HNSW indexes on modern Vulkan-capable GPUs across major
hardware families (since 1.13.0). It supports concurrent segment indexing on
multiple GPUs and works with all Qdrant quantization options and data types.

For on-premises deployment, use the preconfigured GPU container images. Check
the server logs to confirm GPU detection and actual use; do not infer that the
GPU path is active merely because the host has a compatible device.

### Incremental graph extension

Qdrant can extend an existing HNSW graph as new points arrive rather than
rebuilding the entire graph (since 1.14.0). The incremental path applies to
upserts. Deletes and updates still trigger a full rebuild, so model their
maintenance cost separately from append-like ingestion.

### Inline vector storage

Set the collection HNSW option `inline_storage` to `true` to store vector data
inside HNSW nodes (since 1.16.0):

```json
{
  "hnsw_config": {
    "inline_storage": true
  }
}
```

This reduces random disk reads and can perform implicit rescoring from the
original vector stored with each node. Quantization must also be enabled. The
tradeoff is additional storage.

### Per-field graph participation

An individual payload field index can be configured not to be reflected in
the HNSW index (since 1.17.0). Disable participation for payload indexes that
are not used with dense-vector queries to avoid extra graph edges.

## Compression choices

### Higher-bit binary modes

Binary storage supports 1.5-bit and 2-bit modes (since 1.15.0):

| Mode | Compression versus 32-bit vectors | Characteristic |
| --- | ---: | --- |
| 1-bit | 32× | Maximum binary compression |
| 1.5-bit | 24× | Intermediate compression/accuracy tradeoff |
| 2-bit | 16× | Represents zero explicitly |

The 2-bit mode can preserve accuracy better for vectors below roughly 1,000
dimensions because of its explicit zero representation.

### Asymmetric quantization

Stored and query vectors can use different quantization algorithms (since
1.15.0). A notable configuration stores binary-quantized vectors but uses
scalar-quantized query vectors. It keeps storage and RAM close to binary
quantization while improving precision and reducing the need for rescoring.

### TurboQuant

TurboQuant rotates vectors before compression, so it does not require the
centered vector distribution expected by binary quantization (since 1.18.0).
It supports cosine, dot-product, and L2 distance.

Compared with scalar quantization, TurboQuant offers twice the compression
with similar recall and speed. At storage sizes comparable to binary
quantization, it favors recall over speed. Select it according to the
workload's storage, latency, and recall priorities rather than treating it as
a universally faster mode.
