# Agents, ML Commons, and Flows

Use this reference for connectors, inference request processing, agent
registration and execution, memory and sessions, built-in tools, MCP and AG-UI,
Flow Framework, and application authoring.

## Query templating and inference inputs

- The 2.19.0 `template` query leaves placeholders unresolved until a search
  request processor assigns them.
- ML inference search request extensions in 2.19.0 let a search supply
  additional model-specific input fields.
- The 3.1.0 ML inference request processor can parse nested JSON objects during
  its Update Query step.

## Connectors and remote inference

### Connector request contracts

- Inline model connectors no longer require a connector name in 3.1.0.
- Schema-defined strings remain strings during 3.1.0 connector validation
  instead of being coerced.
- ML Commons connectors support custom-named actions and PUT and DELETE in
  3.5.0, allowing one connector to expose a wider external REST surface.
- Connector headers accept per-request `${parameters.*}` substitutions in
  3.7.0, for example `X-Trace-ID: ${parameters.trace_id}`.
- Outbound connector paths add private-IP and ReDoS protections in 3.7.0 and
  consistently enforce `trusted_connector_endpoints_regex`.

### Built-in preprocessors and endpoints

- ML Commons 2.19.0 includes a built-in Cohere multimodal preprocessor
  selectable by function name and Bedrock reranking pre/postprocessing.
- DeepSeek and Amazon Rekognition are trusted endpoints in 2.19.0.

### Embedding and execution options

- Text-embedding models add `LAST_TOKEN` pooling for decoder-only models and
  `NONE` for already-pooled outputs in 3.6.0.
- Conversational, AG-UI, and plan-execute-reflect agents can report token usage
  in 3.6.0.

## Agent registration and lifecycle

### Updating and registering agents

- The 3.1.0 Update Agent API can change model IDs, workflow tools, and prompts
  on an existing agent.
- The disabled-by-default unified registration API in 3.6.0 creates the
  connector, model, agent, and parameter mappings in one request.
- Its `conversational_v2` agent accepts plain text, multimodal content blocks,
  and conversation history without custom connector configuration.
- The unified registration API and `conversational_v2` become
  production-ready in 3.7.0.
- In 3.7.0, V2 `inferenceConfig.model_parameters` values are honored rather
  than silently ignored.

### Agentic search

- Disabled-by-default agentic search in 3.2.0 adds an agentic query clause and
  a request processor that translates natural-language questions into
  OpenSearch DSL through planning, execution, and summarization.
- Agentic search becomes generally available in 3.3.0. Agents select tools,
  generate queries, retain multi-turn context, and use custom search
  templates.
- Conversational agents in 3.3.0 can use the Query Planning Tool and carry an
  agent summary and memory ID.
- Agentic query processing preserves the request source parameter in 3.4.0.
- Agentic planning in 3.6.0 supports aliases and wildcard index patterns, a
  custom fallback query, neural-query embedding-model selection, and
  reranking.

## Memory, sessions, and context management

### Memory containers

- ML Commons 3.2.0 adds memory-container lifecycle APIs. Memory supports add,
  search, update, and delete operations.
- Agents in 3.2.0 can receive current date/time and set a message-history
  limit.
- Agentic memory is generally available and enabled by default in 3.3.0, with
  semantic-fact extraction, preference learning, and conversation
  summarization strategies.
- Session support in 3.3.0 adds message IDs and update timestamps.
- Memory-container deletion in 3.3.0 can control whether contained memories
  are deleted.
- Long-term memory in 3.6.0 adds semantic and hybrid retrieval, memory types
  accept message arrays, and context managers add a structured post-memory
  hook.
- Agentic-memory fact extraction can use constrained structured output in
  3.7.0.

### Context hooks

- Agents in 3.5.0 can run context-management hooks at multiple execution
  stages. Strategies include automatic truncation, summarization, and sliding
  windows before model requests.
- Conversation memory in 3.5.0 persists conversation context and intermediate
  tool reasoning in a structured form and validates misconfiguration.

## Agent tools and processor chains

### Tool inputs and execution

- Conversational-agent tools in 2.19.0 can receive action inputs as parameters
  and use generated inputs as search parameters.
- ML Commons 3.2.0 adds a query-planning tool and Execute Tool API.
- The Execute Tool feature is enabled by default in 3.3.0.
- `AbstractRetrieverTool` adds `input_schema` for function calling in 3.7.0.
- `VectorDBTool` accepts runtime parameter overrides during agentic search in
  3.7.0.

### Processor chains

- ML Commons processor chains in 3.3.0 can run sequential transformations
  through 10 processor types, including JSONPath filters, regular expressions,
  conditions, and array iteration, and can invoke models and tools.

### Built-in tools

- The 3.3.0 built-ins include scratchpad read/write, index-insight,
  log-pattern-analysis, and data-distribution tools.
- In 3.6.0, new tools retrieve documents around a selected document and
  compare metric percentiles across baseline and selection periods.
  `LogPatternAnalysisTool` also accepts a service filter.

## MCP and streaming protocols

### Native MCP

- OpenSearch 3.0.0 has disabled-by-default native MCP integration with external
  agents. ML Commons adds an MCP server, session handling, and a
  plan-execute-reflect agent type with user-supplied prompts.
- Experimental MCP support in 3.1.0 adds list-tools and update-tools APIs,
  persists tools in a system index across restarts, and lets the MCP client use
  a custom SSE endpoint.
- The ML Commons MCP server adopts Streamable HTTP with role-based
  authorization in 3.3.0 and deprecates its SSE transport.
- MCP connectors in 3.3.0 can act as Streamable HTTP clients for external
  servers.

### Inference and UI streaming

- Disabled-by-default 3.3.0 streaming APIs use SSE for partial remote-model
  predictions and agent execution results.
- The disabled-by-default 3.5.0 AG-UI protocol uses event streaming to connect
  agents to user interfaces.

## Flow Framework and application authoring

### Workflows and templates

- OpenSearch Flow in Dashboards 2.19.0 can compose custom ML application
  flows, including RAG and vector-search workflows.
- Flow Framework 2.19.0 adds synchronous workflow provisioning.
- `WorkflowRequest` no longer has `useCase` or `defaultParams` in 2.19.0.
- Flow Framework thread-pool sizes are configurable in 3.1.0.
- Dashboards 3.1.0 adds a sparse-encoder semantic-search workflow template.
- Flow Framework 3.1.0 adds a data-summary template that uses a log-pattern
  agent.

### Dashboards authoring

- The redesigned no-code agentic-search flow in 3.4.0 integrates external MCP
  servers and search templates and supports conversational memory,
  single-model operation, and agent summaries.
- OpenSearch Launchpad in 3.6.0 turns sample documents and conversational
  requirements into a local search application, provisioning semantic
  encoding, cluster configuration, architecture, and a working UI, with IDE
  integration.

## ML Commons observability

- ML Commons 3.1.0 integrates with the OpenSearch metrics framework and
  OpenTelemetry-compatible monitoring.
- It supports runtime instrumentation on selected code paths and scheduled
  collection of state-level metrics.
