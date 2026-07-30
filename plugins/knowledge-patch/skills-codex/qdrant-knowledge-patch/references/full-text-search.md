# Full-Text Search

## Configure Language Processing

### Tokenize multilingual text

Use the built-in `multilingual` tokenizer for languages without whitespace word
boundaries, including Japanese and Chinese (since 1.15.0). It removes the need
for a custom build or external preprocessing:

```http
PUT /collections/{collection_name}/index
{"field_name":"description","field_index_params":{"type":"text","tokenizer":"multilingual"}}
```

### Remove stop words

Configure stop words on the full-text index so Qdrant removes them
automatically instead of requiring each query to strip them (since 1.15.0):

```http
PUT /collections/{collection_name}/index
{"field_name":"title","field_index_params":{"type":"text","stopwords":"english"}}
```

### Apply Snowball stemming

Use a language-specific Snowball stemmer to normalize grammatical variants to
common roots and match related word forms (since 1.15.0):

```http
PUT /collections/{collection_name}/index
{"field_name":"body","field_index_params":{"type":"text","stemmer":{"type":"snowball","language":"english"}}}
```

### Fold diacritics to ASCII

Set `ascii_folding` to `true` when creating a full-text payload index (since
1.16.0). Qdrant normalizes diacritics in indexed text and search terms, so
`cafe` can match `café`:

```json
{
  "type": "text",
  "ascii_folding": true
}
```

## Choose Match Semantics

### Require an exact phrase

Exact phrase matching needs an additional index structure. Enable
`phrase_matching` when the full-text index is created, then use `match.phrase`
to require words in the supplied order (since 1.15.0):

```http
PUT /collections/{collection_name}/index
{"field_name":"headline","field_index_params":{"type":"text","phrase_matching":true}}

POST /collections/{collection_name}/points/query
{
  "query": [0.01, 0.45, 0.67, 0.12],
  "filter": {
    "must": {
      "key": "headline",
      "match": {"phrase": "machine time"}
    }
  },
  "limit": 10
}
```

Setting only `match.phrase` at query time cannot compensate for an index
created without phrase support.

### Match any term

Use `text_any` to tokenize a multi-term query and match a text field containing
at least one term (since 1.16.0). It replaces client-generated `should`
conditions with one match condition:

```json
{
  "match": {
    "text_any": "apple banana cherry"
  }
}
```
