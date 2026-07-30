# Cloud models and web tools

## Cloud tags through the local interface

Sign in before using cloud tags. The normal `run`, `pull`, `ls`, and `cp`
commands work with them, while inference executes on ollama.com. Once a cloud
tag has been pulled, the local API and Ollama libraries use it like a local
model.

```sh
ollama signin
ollama pull gpt-oss:120b-cloud
curl http://localhost:11434/api/chat -d \
  '{"model":"gpt-oss:120b-cloud","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

## Coding integration agents

An Ollama cloud model launched through the Claude integration can use parallel
subagents and web search. Search is provided through the Anthropic compatibility
layer, so this route does not need a separate MCP server or API key. Explicitly
request subagents if they are not selected automatically.

```sh
ollama launch claude --model minimax-m2.5:cloud
```

```text
Spawn subagents to inspect the authentication, payment, and notification flows in parallel.
```

## Hosted web search

Send standalone search requests to `POST https://ollama.com/api/web_search`.
Create an account API key and send it as a bearer token with `query`. Each
result contains `title`, `url`, and `content`.

```sh
curl https://ollama.com/api/web_search \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{"query":"what is ollama?"}'
```

## Hosted page fetch

`POST https://ollama.com/api/web_fetch` accepts a URL. Its response contains
the page `title`, extracted `content`, and discovered `links`.

```sh
curl https://ollama.com/api/web_fetch \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://ollama.com"}'
```

## Python and JavaScript helpers

Python and JavaScript clients from version 0.6 provide
`web_search`/`webSearch` and `web_fetch`/`webFetch`. Their functions can be
passed directly to chat as tools. Give a standalone search agent roughly 32K
tokens of context or more because search results can be large.

```python
from ollama import chat, web_fetch, web_search

response = chat(
    model="qwen3:4b",
    messages=[{"role": "user", "content": "What is Ollama's new engine?"}],
    tools=[web_search, web_fetch],
    think=True,
)
```

## MCP server

Ollama's Python MCP server can expose both search and fetch to stdio MCP
clients. Run the server script with `uv` and supply the account API key through
the server environment:

```toml
[mcp_servers.web_search]
command = "uv"
args = ["run", "path/to/web-search-mcp.py"]
env = { "OLLAMA_API_KEY" = "your_api_key_here" }
```
