# Incident Copilot: Technical Specification

## 1. Goal and scope

Build a local service that helps investigate incidents using a problem description, documentation, and synthetic logs. It combines an LLM (large language model), document search, controlled tool calls, and quality checks.

The MVP supports one user and one dataset. Live production access, automatic command execution, infrastructure changes, model fine-tuning, Kubernetes, and a complex multi-agent system are outside the MVP.

## 2. User flows

1. Send an incident description and receive possible causes with steps to check them.
2. Upload Markdown/TXT documentation and receive answers with document and chunk IDs.
3. Provide synthetic JSONL logs and allow search by service and time range.
4. View job status, results, token usage, and analysis limitations.
5. Receive a clear "not enough information" answer when there is no supporting evidence.

## 3. Phase A: Basic model call

`POST /v1/analyses` accepts `description` (1–8,000 characters). The first version is synchronous and returns HTTP 200 on success. The response includes `analysis_id`, `status`, `summary`, `hypotheses[]`, `checks[]`, `missing_information[]`, `limitations[]`, `sources[]`, and `usage`.

Each hypothesis includes a possible cause and its basis. It is not a confirmed fact. Before document search is added, `sources` is empty and the limitations say that the answer uses only the user's description. Do not show numerical confidence scores without calibration.

Validate model output against a JSON Schema and business rules in Go. Use the provider's structured output feature when available, but keep server-side validation. Allow one attempt to fix the output format within the total budget.

API errors use `code`, `message`, and `request_id`. Handle invalid requests, timeouts, rate limits, provider outages, and invalid model output separately. Do not expose secrets or internal error details.

## 4. Phase B: RAG

RAG means retrieval-augmented generation: find useful information and give it to the model as context.

- `POST /v1/documents` accepts one UTF-8 Markdown/TXT file up to 1 MiB and returns its ID and indexing status.
- Split documents by headings and size. Store the original document, checksum, version, chunks, chunk positions, and embeddings. An embedding is a numeric vector used for similarity search.
- Fix the embedding model and vector size in configuration. Rebuild the index when the embedding model changes.
- Compare PostgreSQL full-text search and vector search on the same dataset. Choose a simple approach based on the results.
- Return at most five chunks per search and stay within the context token budget.
- Answers may cite only chunks that were actually retrieved. Reject unknown source IDs. A valid ID does not prove that the source supports a claim; check this separately during evaluation.
- Replace the old document version in search only after the new version is indexed successfully.

## 5. Phase C: Tools and background jobs

Load logs from a local synthetic JSONL dataset. Fields: `timestamp`, `service`, `level`, `message`, and optional `trace_id`.

Give the model one read-only tool: `search_logs(service, from, to, level, trace_id, limit)`. Validate arguments, allowed services, and the time range in the application. Limit results to 100 records and the time range to 24 hours. Allow at most three tool calls per analysis. Do not allow shell commands, arbitrary SQL, or arbitrary URLs.

In this phase, `POST /v1/analyses` returns HTTP 202 and an analysis ID. `GET /v1/analyses/{id}` returns `pending`, `running`, `completed`, or `failed`. Update the API documentation when changing this contract. Support `Idempotency-Key` for repeat submissions. The same key with a different request body returns HTTP 409.

A worker takes jobs from PostgreSQL using a lock and a lease. The lease limits how long a worker owns a job. A job with an expired lease may be processed again. Use at-least-once processing and save results idempotently. A repeated external call after a failure may cost extra. Do not promise exactly-once LLM calls.

## 6. Reliability and limits

- Overall analysis deadline: 90 seconds. Each external call has a 30-second deadline, limited by the remaining analysis time.
- Allow up to two retries for temporary errors only. Use backoff, jitter, and `Retry-After`. Do not retry invalid requests. Account for possible duplicate charges.
- Allow at most two analyses at once and 10 requests per minute per local API token. Return a clear error when a budget is exhausted; do not retry forever.
- Total budget per analysis: up to 20,000 input tokens and 4,000 output tokens across all calls. Reserve capacity before each call and record actual usage after it. If the provider does not report usage, store an estimate and mark it as estimated.
- Estimate cost using a configurable price list with a model name and date. This is not the provider's bill. Add a configurable daily spending limit and reserve cost capacity before concurrent calls.
- Context cancellation stops local work. It does not guarantee that the provider stops processing or charging for an accepted request.

## 7. Data and security

Bind the local API to the loopback interface and require an API token. Read secrets from environment variables. Do not log full prompts, documents, source logs, or tokens. Diagnostic logs include request/analysis IDs, duration, model, and error type.

Treat documents and logs as untrusted data. They must not change system instructions or available tools. Check permissions and limits in application code, regardless of the model's decision. Add attack cases that place instructions in documents or try to extract a test secret. Secret redaction is an extra safeguard, not a guarantee. Use synthetic data only.

## 8. Observability

Expose `/healthz`, `/readyz`, and `/metrics`. Track HTTP and LLM duration, completions and errors, queue age, in-flight requests, tokens, estimated cost, and tool calls.

Do not use analysis IDs, trace IDs, arbitrary URLs, or request text as metric labels. These values create too many distinct time series.

## 9. Quality checks and acceptance criteria

Prepare at least 20 fixed cases: 10 with enough evidence, 5 with missing evidence, and 5 with malicious instructions in sources. For each case, record expected facts, allowed sources, forbidden claims, and verification steps. Use the same cases for the baseline and RAG. Record model, settings, and prompt version.

- In at least 8 of the 10 answerable cases, the expected cause appears among the first three hypotheses and is supported by sources.
- All 5 cases with missing evidence clearly state that there is not enough supporting information.
- All 5 attack cases avoid forbidden actions and do not expose the test secret.
- Every source link points to an existing chunk. Manually record unsupported claims using the scoring guide.
- Tests cover provider errors, invalid JSON, cancellation, and timeouts. Invalid output is never reported as success.
- Worker tests cover idempotency keys and recovery after a lease expires.
- Include a baseline vs. RAG report with quality, response time, and tokens. Passing this small dataset is a project acceptance target, not proof of production reliability.
- After implementation, `docker compose up --build` starts the environment. The README has working commands and a request example without secrets.

## 10. Plan and definition of done

Weeks 1–4: Phase A and baseline evaluation. Weeks 5–8: Phase B and search comparison. Weeks 9–12: Phase C, limits, and the final report. If time is limited, finish A+B with measurements before adding the worker and tools.

A task is done when its acceptance criteria pass, meaningful checks are added, and documentation is updated. Current status: specifications are ready; the service is not implemented.
