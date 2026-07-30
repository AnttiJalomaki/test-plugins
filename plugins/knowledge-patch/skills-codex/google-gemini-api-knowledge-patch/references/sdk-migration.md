# Google Gen AI SDK migration

Source batch: `genai-sdk-migration`.

Use this reference when replacing a legacy language SDK or adapting code to the
service-oriented Google Gen AI clients.

## GA packages and centralized clients

Replace the legacy packages as follows:

| Legacy | Current |
| --- | --- |
| `google-generativeai` | `google-genai` |
| `@google/generative-ai` | `@google/genai` |
| `github.com/google/generative-ai-go` | `google.golang.org/genai` |

Model objects and separate file/cache managers are replaced by services on one
client.

```python
from google import genai

client = genai.Client()
response = client.models.generate_content(
    model="MODEL_ID", contents="Hello"
)
```

```javascript
import {GoogleGenAI} from "@google/genai";

const ai = new GoogleGenAI({apiKey: process.env.GEMINI_API_KEY});
const response = await ai.models.generateContent({
  model: "MODEL_ID",
  contents: "Hello",
});
```

```go
import "google.golang.org/genai"

client, err := genai.NewClient(ctx, nil)
result, err := client.Models.GenerateContent(
    ctx, "MODEL_ID", genai.Text("Hello"), nil)
```

## Per-call config and Python async access

Generation settings no longer belong to a model instance. Put optional inputs
under each call's `config`, as a dictionary or a Pydantic class from
`google.genai.types`. Python async methods are mirrored under `client.aio`
instead of using an `_async` suffix.

```python
response = await client.aio.models.generate_content(
    model="MODEL_ID",
    contents="Hello",
    config={"max_output_tokens": 200},
)
```

## Flattened JavaScript responses and streams

Generation returns the response itself. Text is the `response.text` property,
not `result.response.text()`. `generateContentStream` returns the async iterable
directly, not an object with a `.stream` member.

```javascript
const stream = await ai.models.generateContentStream({
  model: "MODEL_ID",
  contents: "Write a story.",
});
for await (const chunk of stream) process.stdout.write(chunk.text);
```

## Automatic Python function calling

Passing a Python callable in `tools` now executes the call automatically. The
legacy SDK did this only for chat when explicitly enabled. Disable automatic
execution when the application needs to inspect and dispatch the call itself.

```python
response = client.models.generate_content(
    model="MODEL_ID",
    contents="What is the weather?",
    config=types.GenerateContentConfig(
        tools=[get_current_weather],
        automatic_function_calling={"disable": True},
    ),
)
call = response.candidates[0].content.parts[0].function_call
```

## Parsed structured responses

Python `response_schema` accepts a Pydantic model class. The SDK validates the
JSON and exposes a model instance at `response.parsed`.

```python
class Answer(BaseModel):
    value: str

response = client.models.generate_content(
    model="MODEL_ID",
    contents="Answer the question.",
    config={
        "response_mime_type": "application/json",
        "response_schema": Answer,
    },
)
answer = response.parsed
```

## Cached content through generation config

Create caches through `client.caches`, then pass the returned cache name in
generation config. Do not construct a replacement model object from a cache.

```python
cache = client.caches.create(
    model="MODEL_ID", config={"contents": [document]}
)
response = client.models.generate_content(
    model="MODEL_ID",
    contents="Summarize it.",
    config=types.GenerateContentConfig(cached_content=cache.name),
)
```

## Plural JavaScript embeddings

`ai.models.embedContent` accepts `contents` and returns `result.embeddings`, not
the legacy singular `result.embedding`. Put output dimensionality in `config`.

```javascript
const result = await ai.models.embedContent({
  model: "EMBEDDING_MODEL_ID",
  contents: "Hello world",
  config: {outputDimensionality: 10},
});
console.log(result.embeddings);
```
