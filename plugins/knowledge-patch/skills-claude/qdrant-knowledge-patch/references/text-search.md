# Full-Text Search

Full-text behavior is defined on the payload index. Features that require
additional index structures must be selected when creating that index.

## Multilingual tokenization

Use the built-in `multilingual` tokenizer for languages without whitespace
word boundaries, including Japanese and Chinese (since 1.15.0). It needs
neither a custom build nor external preprocessing:

```http
PUT /collections/{collection_name}/index
{"field_name":"description","field_index_params":{"type":"text","tokenizer":"multilingual"}}
```

## Stop-word removal

Set `stopwords` on a full-text index to remove configured common words
automatically (since 1.15.0). Queries no longer need to strip them manually:

```http
PUT /collections/{collection_name}/index
{"field_name":"title","field_index_params":{"type":"text","stopwords":"english"}}
```

## Snowball stemming

Configure a language-specific Snowball stemmer to normalize grammatical
variants to common roots (since 1.15.0):

```http
PUT /collections/{collection_name}/index
{"field_name":"body","field_index_params":{"type":"text","stemmer":{"type":"snowball","language":"english"}}}
```

This increases matches between related word forms; choose the language that
matches the indexed field.

## Exact phrase matching

Phrase search needs `phrase_matching: true` when the index is created because
Qdrant builds an additional data structure (since 1.15.0). Query with
`match.phrase` to require words in the supplied order:

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

## Match any query term

Use `text_any` to tokenize a multi-term query and match a field containing at
least one term (since 1.16.0):

```json
{
  "match": {
    "text_any": "apple banana cherry"
  }
}
```

This replaces a client-built group of `should` conditions with one match
condition.

## ASCII folding

Set `ascii_folding` to `true` when creating the text payload index to
normalize diacritics in indexed text and search terms (since 1.16.0):

```json
{
  "type": "text",
  "ascii_folding": true
}
```

For example, this allows `cafe` to match `café`.
