# Integrations, cloud models, and web tools

## Configure integrations

`ollama launch` normally configures and starts the selected integration without
manual environment-variable or configuration-file setup.

Use `--config` to configure without launching:

```sh
ollama launch opencode --config
```

The default launcher menu contains popular integrations. Run the command
without an integration name to expose the broader selection:

```sh
ollama launch
```

## Allocate coding context

Set the Ollama context length to at least 64,000 tokens for coding tools.

Recommended local tags:

- `glm-4.7-flash`;
- `qwen3-coder`; and
- `gpt-oss:20b`.

Cloud tags with full context:

- `glm-4.7:cloud`;
- `minimax-m2.1:cloud`;
- `gpt-oss:120b-cloud`; and
- `qwen3-coder:480b-cloud`.

At 64K context, `glm-4.7-flash` requires about 23 GB of local VRAM.

```sh
ollama pull glm-4.7-flash
ollama pull glm-4.7:cloud
```

## Use cloud models locally

Cloud tags support the normal `run`, `pull`, `ls`, and `cp` commands, while
inference runs on ollama.com.

Sign in first. After pulling a cloud tag, local APIs and library clients use it
like a local model:

```sh
ollama signin
ollama pull gpt-oss:120b-cloud
curl http://localhost:11434/api/chat -d \
  '{"model":"gpt-oss:120b-cloud","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

## Launch Claude Code with cloud capabilities

Any Ollama cloud model launched through the Claude integration can use parallel
subagents and web search. Search is provided by the Anthropic compatibility
layer, so this route requires neither an MCP server nor a separate API key.

```sh
ollama launch claude --model minimax-m2.5:cloud
```

If subagents are not selected automatically, request them explicitly:

```text
Spawn subagents to inspect the authentication, payment, and notification flows in parallel.
```

## Call hosted web search

Standalone search uses `POST https://ollama.com/api/web_search`. Create an
account API key and send it as a bearer token with `query`.

Each result contains `title`, `url`, and `content`.

```sh
curl https://ollama.com/api/web_search \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{"query":"what is ollama?"}'
```

## Fetch a page

`POST https://ollama.com/api/web_fetch` accepts a URL. The response contains
the page `title`, extracted `content`, and discovered `links`.

```sh
curl https://ollama.com/api/web_fetch \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://ollama.com"}'
```

## Pass search helpers to chat

Python and JavaScript clients from version 0.6 expose:

- Python: `web_search` and `web_fetch`;
- JavaScript: `webSearch` and `webFetch`.

Pass these functions directly as chat tools. Give a standalone search agent
roughly 32K tokens of context or more because search results can be large.

```python
from ollama import chat, web_fetch, web_search

response = chat(
    model="qwen3:4b",
    messages=[{"role": "user", "content": "What is Ollama's new engine?"}],
    tools=[web_search, web_fetch],
    think=True,
)
```

## Expose web tools through MCP

Ollama's Python MCP server can expose search and fetch to stdio MCP clients.
For a Codex client, run the server script with `uv` and pass the API key in the
server environment:

```toml
[mcp_servers.web_search]
command = "uv"
args = ["run", "path/to/web-search-mcp.py"]
env = { "OLLAMA_API_KEY" = "your_api_key_here" }
```
