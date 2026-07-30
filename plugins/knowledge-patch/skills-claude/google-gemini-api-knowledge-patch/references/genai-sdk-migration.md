# Google Gen AI SDK Migration

## Replace legacy packages with centralized clients

The current GA packages (`genai-sdk-migration`) are:

| Language | Replace | With |
| --- | --- | --- |
| Python | `google-generativeai` | `google-genai` |
| JavaScript/TypeScript | `@google/generative-ai` | `@google/genai` |
| Go | `github.com/google/generative-ai-go` | `google.golang.org/genai` |

Model objects and separate file or cache managers are replaced by services on
one client.

Python:

```python
from google import genai

client = genai.Client()
response = client.models.generate_content(
    model="MODEL_ID",
    contents="Hello",
)
```

JavaScript:

```javascript
import {GoogleGenAI} from "@google/genai";

const ai = new GoogleGenAI({
  apiKey: process.env.GEMINI_API_KEY,
});
const response = await ai.models.generateContent({
  model: "MODEL_ID",
  contents: "Hello",
});
```

Go:

```go
import "google.golang.org/genai"

client, err := genai.NewClient(ctx, nil)
result, err := client.Models.GenerateContent(
    ctx, "MODEL_ID", genai.Text("Hello"), nil)
```

## Move configuration to each call

Generation settings no longer live on a model instance. Put optional inputs
under the call's `config`, as a dictionary or a Pydantic configuration type
from `google.genai.types`.

Python asynchronous methods are mirrored under `client.aio`; they do not have
an `_async` suffix:

```python
response = await client.aio.models.generate_content(
    model="MODEL_ID",
    contents="Hello",
    config={"max_output_tokens": 200},
)
```

## Update JavaScript response and stream access

Generation returns the response itself. Text is the `response.text` property,
not `result.response.text()`.

`generateContentStream` returns the async iterable directly rather than an
object with a `.stream` member:

```javascript
const stream = await ai.models.generateContentStream({
  model: "MODEL_ID",
  contents: "Write a story.",
});
for await (const chunk of stream) {
  process.stdout.write(chunk.text);
}
```

## Control automatic Python function calling

Passing a Python callable in `tools` to `generate_content` runs it
automatically by default. The legacy SDK only did this for chat when explicitly
enabled. Disable execution when the application must inspect or dispatch the
call:

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

## Parse typed structured responses

Python accepts Pydantic model classes as structured-output schemas. When one is
passed as `response_schema`, the SDK validates the JSON and exposes its
instance at `response.parsed`:

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

## Reference cached content through generation config

Create caches with `client.caches`, then put the returned name in the generation
configuration. Do not construct a replacement model object from a cache:

```python
cache = client.caches.create(
    model="MODEL_ID",
    config={"contents": [document]},
)
response = client.models.generate_content(
    model="MODEL_ID",
    contents="Summarize it.",
    config=types.GenerateContentConfig(
        cached_content=cache.name,
    ),
)
```

## Read plural JavaScript embeddings

`ai.models.embedContent` accepts `contents` and returns `result.embeddings`,
not the legacy singular `result.embedding`. Set output dimensionality inside
the request's `config`:

```javascript
const result = await ai.models.embedContent({
  model: "EMBEDDING_MODEL_ID",
  contents: "Hello world",
  config: {outputDimensionality: 10},
});
console.log(result.embeddings);
```
