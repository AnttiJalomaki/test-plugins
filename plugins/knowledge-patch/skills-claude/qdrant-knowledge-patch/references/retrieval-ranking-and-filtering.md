# Retrieval, Ranking, and Filtering

## Filter by named-vector presence

Use the `has_vector` condition to select points that contain a particular
named vector (since 1.13.0). This is useful in heterogeneous collections where
not every point has every embedding:

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

## Formula-based score boosting

The points query API can rescore prefetched vector candidates with a `formula`
(since 1.14.0). A formula can combine `$score` with payload conditions and
numeric expressions. Decay expressions can use datetime payloads or
geographic distance, and `defaults` can provide missing payload values.

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

Size the prefetch limit for the pool that needs rescoring; the formula acts on
that candidate set rather than replacing first-stage retrieval.

## Diversity with MMR

Maximal Marginal Relevance reranks nearest-neighbor candidates to trade
relevance for result diversity (since 1.15.0). `diversity` ranges from `0.0`
for relevance to `1.0` for diversity, while `candidates_limit` controls the
initial candidate pool:

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

## Filtered HNSW search with ACORN

ACORN improves HNSW result quality when multiple low-selectivity filters
remove direct graph neighbors (since 1.16.0). It explores neighbors of those
neighbors to find paths around filtered-out points.

Enable ACORN with the optional query-time `acorn` parameter. It needs no
index-time changes. Because the additional traversal adds runtime overhead,
reserve it for filtered queries affected by disconnected local neighborhoods.

## Reciprocal Rank Fusion

The RRF `k` constant is configurable (since 1.16.0). Adjusting it changes how
quickly contribution decays with rank.

RRF can also assign a weight to each contributing query (since 1.17.0). Use
weights when rankers differ in quality so a weak result set does not dilute a
stronger one through equal weighting.

## Relevance feedback

Relevance Feedback Query accepts context pairs containing more-relevant and
less-relevant examples (since 1.17.0). It changes similarity scoring across
the vector space, avoiding both a separate large rescoring pass and embedding
model retraining.

Use it when the caller can supply comparative feedback examples and needs
that context to influence first-class vector retrieval.

## Replica tail-latency mitigation

A read can delay before sending a second request to another replica, then use
the response that arrives first (since 1.17.0). Configure the latency
threshold to hedge slow-replica outliers without immediately fanning every
read out to multiple replicas.
