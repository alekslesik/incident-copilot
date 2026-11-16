# Incident Copilot

An assistant for investigating incidents using logs and documentation.

## Status

The specification and task plan are ready. Service implementation has not started.

## How it works

A user describes a failure. The service finds relevant documentation and test logs, suggests possible causes, provides sources, and lists checks to run. If evidence is missing, it explains the limits and asks for more information. It does not fix production systems automatically.

## Stack and approach

- Go: HTTP API, request flow, and model client.
- PostgreSQL: jobs, documents, chunks, and results. Add pgvector for vector search in the RAG phase.
- One LLM provider behind an interface. Choose the model through configuration.
- Docker Compose for the local environment once it is implemented.
- Start with one process and synchronous requests. Add a worker and a PostgreSQL job queue when background analysis is needed. The MVP does not need a separate message broker.

## Phases

1. **LLM API:** incident description → structured analysis with clear limitations.
2. **RAG:** Markdown/TXT upload → document search → answers with source links.
3. **Tools and reliability:** synthetic log search, limits, metrics, and quality regression checks.

Initial estimate: 12 weeks at 4–6 hours per week. Adjust the plan after the first phase.

## Documentation

- [Technical specification](docs/SPEC.md)
- [Epic and tasks](docs/EPIC.md)
- [GitHub epic](https://github.com/alekslesik/incident-copilot/issues/1)

## First milestone

Build `POST /v1/analyses`: accept an incident description and return JSON with possible causes, verification steps, and missing information. Check successful responses, timeouts, provider errors, and invalid model JSON. Then add RAG.

## Data handling

Use synthetic logs and test documentation. Read API keys from environment variables. Do not commit secrets or real work logs. Redact possible secrets before sending data to the provider. Treat document and log text as untrusted input.

## Local setup

Run commands will be added with the first implementation phase. The repository currently contains project documentation only.

## Development resources

Use these resources as you build each part. Start with model calls and response format, then quality checks, RAG, and tools. The OpenAI examples show one integration option; the architecture keeps the provider replaceable.

| Resource | What to learn | Project tasks |
| --- | --- | --- |
| [Official OpenAI Go SDK](https://github.com/openai/openai-go) | Create a client, call the Responses API, handle errors, and configure retries. Coordinate SDK and service retries so they do not multiply requests. | T02: first model call |
| [OpenAI: Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) | JSON Schema, supported schema limits, refusals, and incomplete responses. Valid JSON does not guarantee a correct diagnosis. | T03: response format and validation |
| [OpenAI: Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices) | Test cases, scoring rules, comparisons, and regression checks. A baseline is the first result used for comparison. | T04 and T07: baseline vs. RAG |
| [OpenAI: Retrieval](https://developers.openai.com/api/docs/guides/retrieval) | Semantic search, embeddings, document chunks, and relevant context. This guide uses OpenAI-managed vector stores; apply the concepts to PostgreSQL in this project. | T05–T07: document-based answers |
| [pgvector documentation](https://github.com/pgvector/pgvector) | Vector distances, exact and approximate search, HNSW/IVFFlat, and filters. Start with exact search and add indexes based on measurements. | T06: embeddings and search |
| [OpenAI: Function calling](https://developers.openai.com/api/docs/guides/function-calling) | Model requests a tool → application validates and runs it → model receives the result. Keep permissions, limits, and execution in application code. | T08: read-only log search |
| [OpenTelemetry Go: Getting Started](https://opentelemetry.io/docs/languages/go/getting-started/) | Add telemetry to an HTTP service. Use spans and metrics to measure document search, LLM requests, and tool calls separately. | T11: diagnose latency and errors |

### Where to start

1. Use the Go SDK guide to make a small model request and handle errors: T02.
2. Define the analysis JSON format with Structured Outputs. Test invalid output, refusals, and incomplete responses: T03.
3. Build a fixed test set and measure the baseline before adding RAG: T04.
4. Move to Retrieval and pgvector. After measuring answer quality, add function calling.

For RAG, first understand this flow: document → chunks → embeddings → search → model context → answer with sources. After each phase, compare quality, response time, and token usage on the same test cases.
