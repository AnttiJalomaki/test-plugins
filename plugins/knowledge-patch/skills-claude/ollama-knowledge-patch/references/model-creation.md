# Model creation, import, and quantization

## Import GGUF artifacts

Since 0.30, `FROM` accepts either a GGUF file or a directory containing GGUF
files. This allows a downloaded artifact to be packaged as a local Ollama
model. (batch 0.30-0.32)

```text
FROM ./my-model.Q4_K_M.gguf
```

```sh
ollama create -f Modelfile my-model
ollama run my-model
```

Tool calling carries over when the imported GGUF supports it. Use
`ollama show my-model` and verify that `tools` appears in its capabilities
before selecting it in an integration.

## Import Safetensors weights

A Modelfile can build directly from a directory of Safetensors weights for a
supported architecture. Place the Modelfile next to the weights and use:

```text
FROM .
```

```sh
ollama create my-model
```

Direct import supports Llama, Mistral/Mixtral, Gemma, and Phi-3, including
fine-tunes that have been fused with their foundation model.

## Apply adapters

`ADAPTER` accepts a Safetensors adapter directory or a GGUF adapter file. Paths
can be absolute or relative to the Modelfile.

```text
FROM llama3.2
ADAPTER ./adapter.gguf
```

The `FROM` value must identify the exact base used for fine-tuning; otherwise
results can be erratic. For Safetensors adapters, prefer non-quantized adapters
over QLoRA adapters because quantization methods differ across frameworks.

## Upload blobs and create through the API

Upload GGUF or Safetensors data with
`POST /api/blobs/sha256:<digest>`, then pass filename-to-digest mappings to
`POST /api/create`. Put base model files in `files` and LoRA adapter files in
`adapters`. Use `HEAD /api/blobs/sha256:<digest>` to avoid uploading an object
that is already present.

```sh
digest=$(sha256sum model.gguf | cut -d ' ' -f 1)
curl -T model.gguf -X POST \
  "http://localhost:11434/api/blobs/sha256:$digest"
curl http://localhost:11434/api/create -d \
  "{\"model\":\"my-model\",\"files\":{\"model.gguf\":\"sha256:$digest\"}}"
```

## Quantize while creating

The CLI accepts `-q` or `--quantize` to convert an FP16 or FP32 source as it
creates the model:

```text
FROM /path/to/fp16-model
```

```sh
ollama create --quantize q4_K_M my-model
```

The create API accepts `quantize` for a non-quantized source. Supported values
are `q4_K_M`, `q4_K_S`, and `q8_0`; `q4_K_M` and `q8_0` are recommended.

```sh
curl http://localhost:11434/api/create -d \
  '{"model":"llama3.2:quantized","from":"llama3.2:3b-instruct-fp16","quantize":"q4_K_M"}'
```

## Declare a minimum runtime

Use `REQUIRES` to record the minimum Ollama version needed by a model:

```text
FROM llama3.2
REQUIRES 0.14.0
```

## Alias a model

If a compatibility client has a hard-coded default model name, copy an
existing Ollama model to that name and send requests using the alias:

```sh
ollama cp llama3.2 gpt-3.5-turbo
```

## Set derived-model context

The OpenAI-compatible API does not have a per-request context-size field.
Create a derived model with `PARAMETER num_ctx`, then use that model in API
requests:

```text
FROM llama3.2
PARAMETER num_ctx 65536
```

```sh
ollama create mymodel
```
