# Vector Search and Ranking

## Filter by Vector Availability

Use `has_vector` to select points containing a specified named vector (since
1.13.0). This supports heterogeneous multi-vector collections where different
points carry different embeddings:

```http
POST /collections/{collection_name}/points/scroll
{
  "filter": {
    "must": [
      { "has_vector": "image" }
    ]
  }
}
```

## Rescore and Diversify Results

### Formula-based scoring

The points query API can rescore prefetched vector results with `formula`
(since 1.14.0). A formula can combine `$score` with payload conditions and
numeric expressions. Conditions act as score inputs, decay expressions can use
datetime payloads or geographic distance, and `defaults` supplies fallback
payload values.

```http
POST /collections/{collection_name}/points/query
{
  "prefetch": {
    "query": [0.2, 0.8],
    "limit": 50
  },
  "query": {
    "formula": {
      "sum": [
        "$score",
        {
          "mult": [
            0.5,
            {
              "key": "tag",
              "match": { "any": ["h1", "h2"] }
            }
          ]
        }
      ]
    }
  }
}
```

### Maximal Marginal Relevance

Use MMR to trade relevance for diversity (since 1.15.0). `diversity` ranges
from `0.0` for relevance to `1.0` for diversity, and `candidates_limit`
controls the size of the initial candidate set:

```http
POST /collections/{collection_name}/points/query
{
  "query": {
    "nearest": [0.01, 0.45, 0.67, 0.12],
    "mmr": {
      "diversity": 0.5,
      "candidates_limit": 100
    }
  },
  "limit": 10
}
```

### Relevance feedback

A Relevance Feedback Query refines retrieval from context pairs containing
more-relevant and less-relevant examples (since 1.17.0). It changes similarity
scoring across the vector space, avoiding both a separate large rescoring pass
and embedding retraining.

### Reciprocal Rank Fusion

The RRF `k` constant is configurable (since 1.16.0). Weighted RRF can also
assign a weight to each contributing query (since 1.17.0), preventing a weak
ranker from diluting stronger result sets through equal weighting.

## Improve Filtered HNSW Search

### Enable ACORN per query

ACORN helps when several low-selectivity filters remove direct HNSW graph
neighbors (since 1.16.0). It explores neighbors of those neighbors to improve
result quality.

Enable the optional `acorn` query parameter only for affected filtered queries.
It requires no index-time change but adds runtime overhead.

### Control per-field HNSW participation

An individual payload field index can be configured not to be reflected in the
HNSW index (since 1.17.0). Disable participation for fields that are not used
with dense-vector queries to avoid creating unnecessary graph edges.

## Select Quantization

### Multi-bit binary modes

Binary storage can use 1.5 or 2 bits per dimension (since 1.15.0):

| Mode | Compression relative to uncompressed 32-bit vectors | Use |
| --- | --- | --- |
| 1-bit | 32× | Maximum binary compression |
| 1.5-bit | 24× | Intermediate compression and accuracy |
| 2-bit | 16× | Explicit zero representation and higher accuracy potential |

The 2-bit mode can preserve accuracy better for vectors below roughly 1,000
dimensions because it represents zero explicitly. The 1.5-bit mode provides a
middle compression/accuracy tradeoff.

### Asymmetric quantization

Stored vectors and query vectors can use different quantization algorithms
(since 1.15.0). A notable pairing is binary storage with scalar-quantized
queries. It keeps storage and RAM close to binary quantization while improving
precision and reducing the need for rescoring.

### TurboQuant

TurboQuant rotates vectors before compression (since 1.18.0), so it does not
depend on the centered vector distribution expected by binary quantization. It
supports cosine, dot-product, and L2 distance.

Compared with scalar quantization, it provides twice the compression with
similar recall and speed. At storage sizes comparable to binary quantization,
it favors recall over speed.
