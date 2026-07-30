# Agents, ML Commons, and Flow Framework

Use this reference for agent registration and execution, memory, tools,
connectors, MCP transport, ML inference processing, and Flow Framework.

## Flow authoring and provisioning

### Flow authoring and provisioning

Since 2.19.0, OpenSearch Flow in Dashboards composes custom ML application
flows, including RAG and vector-search workflows. Flow Framework supports
synchronous workflow provisioning. `WorkflowRequest` no longer has `useCase`
or `defaultParams`.

### Flow and replication lifecycle changes

Since 3.0.0, Dashboards Flow Framework accepts ingestion input as JSON Lines.
Index State Management's `unfollow` action invokes stop replication for
cross-cluster replication.

### Flow configuration and templates

Since 3.1.0, Flow Framework thread-pool sizes are configurable. Dashboards
provides a sparse-encoder semantic-search workflow, and Flow Framework includes
a data-summary template that uses a log-pattern agent.

## Connectors and inference processing

### ML connectors and agent tool inputs

Since 2.19.0, ML Commons provides a built-in Cohere multimodal preprocessor
selectable by function name, plus preprocessing and postprocessing for Bedrock
reranking. DeepSeek and Amazon Rekognition are trusted endpoints.
Conversational-agent tools can receive action inputs as parameters and use
language-model-generated inputs as search parameters.

### ML connector request behavior

Since 3.1.0, inline model connectors do not require a connector name.
Schema-defined strings remain strings during validation instead of being
coerced, and the ML inference request processor's Update Query step parses
nested JSON objects.

### ML connector actions

Since 3.5.0, ML Commons connectors support custom-named actions and the PUT and
DELETE HTTP methods, so one connector can expose a wider external REST surface.

### ML connector runtime controls

Since 3.7.0, connector headers accept per-request `${parameters.*}`
substitutions, such as `X-Trace-ID: ${parameters.trace_id}`. Outbound connector
paths add private-IP and ReDoS protection and consistently enforce
`trusted_connector_endpoints_regex`.

## Agent registration and execution

### Experimental MCP and reflective agents

OpenSearch 3.0.0 includes disabled-by-default native Model Context Protocol
integration. ML Commons adds an MCP server and sessions plus a
plan-execute-reflect agent with user-supplied prompts.

### Agent and MCP lifecycle APIs

Since 3.1.0, Update Agent can modify model IDs, workflow tools, and prompts.
Experimental MCP adds list-tools and update-tools APIs, persists tools in a
system index across restarts, and permits a custom SSE endpoint for the client.

### Experimental agentic search

OpenSearch 3.2.0 introduces disabled-by-default agentic search: an agentic query
clause and a request processor translate natural-language questions to
OpenSearch DSL through planning, execution, and summarization workflows.

### Agentic search general availability

In 3.3.0, agentic search becomes generally available. Agents can select tools,
generate queries, keep multi-turn context, and use custom search templates.
Conversational agents can use Query Planning Tool and carry an agent summary
and memory ID.

### Agentic-search authoring

Since 3.4.0, Dashboards has a redesigned no-code agentic-search flow with
external MCP and search-template integration, conversational memory,
single-model support, and agent summaries. Agentic query processing preserves
the request's source parameter.

### Agent context hooks and conversation memory

Since 3.5.0, context-management hooks run at different execution stages and can
truncate, summarize, or use sliding windows before language-model requests.
Conversation memory persists conversation context and intermediate tool
reasoning in a structured form and validates misconfiguration.

### Experimental agent and web protocols

OpenSearch 3.5.0 includes disabled-by-default Agent-User Interaction (AG-UI)
event streaming for connecting agents to user interfaces and disabled-by-default
server-side HTTP/3.

### Conversational V2 agents

OpenSearch 3.6.0 introduces a disabled-by-default unified registration API that
creates the connector, model, agent, and parameter mappings in one request.
Its `conversational_v2` agent accepts plain text, multimodal content blocks,
and conversation history without custom connector configuration.

### Production-ready unified agents

In 3.7.0, unified registration and `conversational_v2` become production-ready.
V2 `inferenceConfig.model_parameters` values are honored rather than ignored,
and agentic-memory fact extraction can use constrained structured output.

## Planning, memory, and context

### Agent tools and memory

Since 3.2.0, ML Commons includes a query-planning tool, Execute Tool API, and
memory-container lifecycle APIs. Memory supports add, search, update, and
delete. Agents can receive current date and time and configure message-history
limits.

### Persistent agentic memory and sessions

In 3.3.0, agentic memory is generally available and enabled by default, with
semantic fact extraction, preference learning, and conversation summarization.
Sessions include message IDs and update times. Deleting a memory container can
optionally delete its memories.

### Agentic-search planning and memory

Since 3.6.0, planning supports aliases, wildcard index patterns, custom fallback
queries, embedding-model selection for neural queries, and reranking. Long-term
memory supports semantic and hybrid retrieval, memory types accept message
arrays, and context managers have a structured post-memory hook.

### Agent and embedding runtime options

Since 3.6.0, conversational, AG-UI, and plan-execute-reflect agents can report
token usage. Text-embedding models support `LAST_TOKEN` pooling for decoder-only
models and `NONE` for outputs that are already pooled.

## Tools and processor chains

### Processor chains and agent tools

Since 3.3.0, ML Commons processor chains run sequential transformations through
10 processor types, including JSONPath filters, regular expressions,
conditions, and array iteration, and can invoke models and tools. New built-ins
include scratchpad read/write, index insight, log pattern analysis, and data
distribution. Execute Tool is enabled by default.

### Built-in agent tools

Since 3.6.0, built-in tools can retrieve documents surrounding a selected
document and compare metric percentiles across baseline and selection periods.
`LogPatternAnalysisTool` accepts a service filter.

### Agent tool contracts

Since 3.7.0, `AbstractRetrieverTool` has `input_schema` for function calling,
and `VectorDBTool` accepts runtime parameter overrides during agentic search.

## MCP and streaming transport

### MCP and inference streaming

Since 3.3.0, the ML Commons MCP server uses Streamable HTTP with role-based
authorization and deprecates SSE transport. MCP connectors can be Streamable
HTTP clients for external servers. Separately, disabled-by-default SSE APIs
stream partial remote-model predictions and agent execution results.

## Application generation

### OpenSearch Launchpad

Since 3.6.0, Launchpad turns sample documents and conversational requirements
into a local search application. It provisions semantic encoding, cluster
configuration, architecture, and a working UI and integrates the result with
an IDE.
