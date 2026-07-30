# Vector Search and GenAI

Use this reference for vector-index configuration, vector import, Cypher 25
`SEARCH`, vector procedure migrations, and GenAI plugin functions.

## Hi-Fidelity quantized vector indexes

Preview Hi-Fidelity Quantized Vector Search (HFQ) expands an initial search
over quantized vectors and reranks the candidates with full-precision vectors
(since 2026.06.0). Configure the quantization type and default search expansion
factor per index:

```cypher
CREATE VECTOR INDEX moviePlots IF NOT EXISTS
FOR (m:Movie)
ON m.embedding
OPTIONS {indexConfig: {
  `vector.quantization.type`: 'binary',
  `vector.default_search_expansion_factor`: 2.0,
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}}
```

An existing vector index must be rebuilt to use HFQ; setting the new options
does not retrofit the existing index.

## Filtered vector `SEARCH`

Cypher 25 vector searches accept `IN` within the search filter predicate:

```cypher
MATCH (movie:Movie)
SEARCH movie IN (
  VECTOR INDEX moviePlots
  FOR $queryVector
  WHERE movie.genre IN ['Horror', 'SciFi']
  LIMIT $topK
)
RETURN movie.title AS title, movie.rating AS rating
```

## Vector import parsing

For `neo4j-admin database import`:

- `--vector-delimiter` must differ from both `--delimiter` and `--quote`.
- Native Parquet list types can supply vector values directly.

Update generated commands that reuse a delimiter, and prefer the native list
representation when the source format supports it.

## Vector procedure migrations

In Cypher 25, replace deprecated vector-query procedures with `SEARCH`:

```text
db.index.vector.queryNodes() -> SEARCH
db.index.vector.queryRelationships() -> SEARCH
```

Two older procedures are removed:

```text
db.index.vector.createNodeIndex() -> CREATE VECTOR INDEX
db.create.setVectorProperty() -> db.create.setNodeVectorProperty()
```

Update allowlists, generated statements, test fixtures, and result mappings.
The `SEARCH` clause is not a drop-in procedure name substitution; rewrite the
query shape.

## Azure OpenAI base URL

The GenAI plugin provides `GENAI_AZURE_OPENAI_BASE_URL` (since 2026.04.0). Set
it when `ai.text` calls must use a different Azure OpenAI base URL. Keep the
endpoint aligned with the deployment configuration.

## Token-aware text processing

The GenAI plugin provides:

- `ai.text.countToken` to estimate the token count of input; and
- `ai.text.chunkByTokenLimit` to divide input into chunks that fit a token
  limit.

## File-based batch embeddings

`ai.file.embedBatch` reads text from a local or remote file and creates
embeddings (since 2026.05.0). It can optionally split the input into chunks and
returns one row per chunk with:

- the chunk index;
- the chunk content; and
- its embedding vector.

## Validation checklist

1. Confirm that vector indexes use the intended dimensions, similarity
   function, quantization type, and expansion factor.
2. Rebuild pre-existing indexes before expecting HFQ behavior.
3. Test `SEARCH` filters and compare recall after migration from procedures.
4. Validate vector delimiter choices and native Parquet lists with production
   import samples.
5. Exercise token counting, chunk boundaries, local and remote file reads, and
   embedding-row mappings independently.
