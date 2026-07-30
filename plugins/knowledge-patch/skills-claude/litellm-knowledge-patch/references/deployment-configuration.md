# Deployment configuration

## Reusable named credentials

Top-level `credential_list` entries provide rotatable credentials shared by deployments through `litellm_credential_name`. Every entry needs a `credential_info` mapping, even when empty.

```yaml
model_list:
  - model_name: chat
    litellm_params:
      model: azure/gpt-4o
      litellm_credential_name: azure-prod
credential_list:
  - credential_name: azure-prod
    credential_values:
      api_key: os.environ/AZURE_API_KEY
      api_base: os.environ/AZURE_API_BASE
    credential_info: {}
```

## Environment-scoped exposure

Set `LITELLM_ENVIRONMENT` to `production`, `staging`, or `development`. `model_info.supported_environments` then exposes a model only in the selected environments.

```yaml
model_list:
  - model_name: chat
    litellm_params: {model: openai/gpt-4o}
    model_info:
      supported_environments: [production, staging]
```

## Prompt framing

A Proxy model can override automatically detected prompt formatting in `litellm_params`. The template accepts initial and final text, per-role `pre_message` and `post_message`, and `bos_token` and `eos_token`.

```yaml
model_list:
  - model_name: custom-chat
    litellm_params:
      model: huggingface/example/instruct
      initial_prompt_value: "\n"
      roles:
        user: {pre_message: "<|im_start|>user\n", post_message: "<|im_end|>"}
        assistant: {pre_message: "<|im_start|>assistant\n", post_message: "<|im_end|>"}
      final_prompt_value: "\n"
```

## Custom token counting

`model_info.custom_tokenizer` makes `/utils/token_counter` use a specified Hugging Face tokenizer for that Proxy model. Private tokenizers can supply `auth_token`.

```yaml
model_info:
  custom_tokenizer:
    identifier: deepseek-ai/DeepSeek-V3-Base
    revision: main
    auth_token: os.environ/HUGGINGFACE_API_KEY
```

## Config discovery and convergence

`CONFIG_FILE_PATH` starts `litellm` from a mounted config without `--config`. Alternatively, set `LITELLM_CONFIG_BUCKET_NAME` and `LITELLM_CONFIG_BUCKET_OBJECT_KEY` for S3; `LITELLM_CONFIG_BUCKET_TYPE=gcs` selects GCS.

`supported_db_objects` limits which stored object classes are loaded. `proxy_config_reload_interval_seconds` controls cross-pod database refresh and defaults to 30 seconds.

## Redis separation and globals

Coordination Redis can be independent of the response cache. The usage cache can be constructed from `REDIS_*` environment variables. The request allowlist in `general_settings` is applied to LiteLLM globals.
