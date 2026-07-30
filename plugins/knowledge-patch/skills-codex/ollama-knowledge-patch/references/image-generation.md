# Image generation

Image generation is experimental. Keep callers tolerant of endpoint and
platform changes, and decode base64 API output before treating it as an image
file.

## Native generate API

`POST /api/generate` automatically detects image-generation models.

Request controls:

- `width`;
- `height`; and
- `steps`.

Streams report `completed` and `total`. The final `image` field contains a
base64-encoded image.

```sh
curl http://localhost:11434/api/generate -d \
  '{"model":"x/z-image-turbo","prompt":"a sunset over mountains","width":1024,"height":768}'
```

## OpenAI-compatible image API

`/v1/images/generations` accepts:

- `model`;
- `prompt`; and
- `size`.

`response_format` must be `"b64_json"`. The endpoint does not support `n`,
`quality`, `style`, or `user`.

```python
response = client.images.generate(
    model="x/z-image-turbo",
    prompt="A robot learning to paint",
    size="1024x1024",
    response_format="b64_json",
)
```

This compatibility endpoint may change or be removed.

## Generate from the CLI on macOS

Pass a prompt directly to an image model. Ollama saves the image in the current
directory. A terminal that supports image rendering also displays an inline
preview.

```sh
ollama run x/z-image-turbo "a watercolor lighthouse in a winter storm"
ollama run x/flux2-klein "a neon sign reading OPEN 24 HOURS"
```

Windows and Linux were not supported when this CLI feature was announced.

## Configure an interactive session

Within an image-model session, set dimensions with:

```text
/set width 1024
/set height 768
```

Each model supplies a recommended default step count. Interactive generation
also supports reproducible random seeds and negative prompts.
