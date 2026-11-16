# Epic: Incident Copilot — Go, LLM, RAG, and incident analysis

## Goal

A user sends an incident description and receives possible causes supported by test documentation and synthetic logs. The service limits model and tool calls, measures quality and cost, and reports missing evidence.

The epic is done when the acceptance criteria in section 9 of the [technical specification](SPEC.md) pass. All tasks are currently open. T01–T12 are task IDs linked to individual GitHub Issues.

## Phase 1: First working model call, weeks 1–4

- [ ] [T01](https://github.com/alekslesik/incident-copilot/issues/2) — **T01: Go service skeleton.** Add configuration, an HTTP server, graceful shutdown, request IDs, `/healthz`, `/readyz`, and `.env.example`. **Acceptance:** The service starts locally and stops cleanly on a shutdown signal. Missing required configuration produces a clear error. **Dependencies:** None.

- [ ] [T02](https://github.com/alekslesik/incident-copilot/issues/3) — **T02: LLM client.** Put one model provider behind an interface. Add request deadlines, error handling, and token usage tracking. **Acceptance:** A model call works. Tests with a stub cover timeouts, HTTP 429, and HTTP 5xx. Logs do not contain secrets. **Dependencies:** T01 (#2)

- [ ] [T03](https://github.com/alekslesik/incident-copilot/issues/4) — **T03: Analysis API and response format.** Add synchronous `POST /v1/analyses`. Define a JSON Schema and validate model responses. Allow only one attempt to fix an invalid response format. **Acceptance:** A valid request returns possible causes and limitations. Invalid model output is never returned as a successful analysis. **Dependencies:** T02 (#3)

- [ ] [T04](https://github.com/alekslesik/incident-copilot/issues/5) — **T04: Test dataset and baseline.** Prepare 20 cases with expected facts and a scoring guide. Add a runner that records the model, settings, and results. A baseline is the first result used for later comparisons. **Acceptance:** Produce a baseline report using fixed inputs. Include quality, response time, and token usage. Note that model output can vary between runs. **Dependencies:** T03 (#4)

## Phase 2: Answers from documentation, weeks 5–8

- [ ] [T05](https://github.com/alekslesik/incident-copilot/issues/6) — **T05: Document storage and import.** Add PostgreSQL migrations and import UTF-8 Markdown/TXT files up to 1 MiB. Store checksums, versions, and indexing status. **Acceptance:** Define and test what happens when the same document is imported again. A new version becomes active only after successful indexing. **Dependencies:** T01 (#2)

- [ ] [T06](https://github.com/alekslesik/incident-copilot/issues/7) — **T06: Document search.** Split documents into chunks. Build a PostgreSQL full-text search baseline. Add embeddings with pgvector and compare search results. **Acceptance:** Measure how often search finds relevant chunks on labeled questions. Limit the context size sent to the model. **Dependencies:** T04 (#5), T05 (#6)

- [ ] [T07](https://github.com/alekslesik/incident-copilot/issues/8) — **T07: RAG answers and sources.** Send retrieved chunks to the model. Validate source IDs. Return a clear answer when there is not enough evidence. **Acceptance:** Meet the quality targets in the specification and compare results with the baseline. Check source IDs and whether sources support the claims as separate checks. **Dependencies:** T06 (#7)

## Phase 3: Tools and operations, weeks 9–12

- [ ] [T08](https://github.com/alekslesik/incident-copilot/issues/9) — **T08: Read-only log search tool.** Use synthetic JSONL logs. Add filters for service, time, level, and trace ID. Validate all tool arguments. **Acceptance:** Allow at most 3 tool calls per analysis, 100 records per call, and a 24-hour time range. Do not allow arbitrary SQL, shell commands, or URLs. **Dependencies:** T07 (#8)

- [ ] [T09](https://github.com/alekslesik/incident-copilot/issues/10) — **T09: Worker and analysis job lifecycle.** Return HTTP 202 when an analysis is accepted. Add a GET status endpoint, a PostgreSQL job queue, a lease, and `Idempotency-Key` support. A lease gives a worker a limited time to own a job. **Acceptance:** The same key and request body return the same analysis ID. The same key with a different body returns HTTP 409. Jobs with expired leases can be retried without saving duplicate results. **Dependencies:** T05 (#6), T08 (#9)

- [ ] [T10](https://github.com/alekslesik/incident-copilot/issues/11) — **T10: Budgets and access controls.** Add API token checks, rate and concurrency limits, an overall deadline, token and cost reservations, and tests with malicious instructions in sources. **Acceptance:** Limits also work with concurrent requests. Prepared attack tests do not expose secrets. Errors are clear. Add basic limits during T02/T03; complete end-to-end checks here. **Dependencies:** T09 (#10)

- [ ] [T11](https://github.com/alekslesik/incident-copilot/issues/12) — **T11: Metrics and diagnostic logs.** Add `/metrics`. Track duration, errors, queue state, tokens, and estimated cost. **Acceptance:** Metrics cover both successful and failed requests. Metric labels do not contain unique request IDs or other values with many possible values. User data is not logged. **Dependencies:** T09 (#10)

- [ ] [T12](https://github.com/alekslesik/incident-copilot/issues/13) — **T12: Local setup and demo.** Add Docker Compose, CI checks for fmt/vet/test, curl examples, a final quality report, and short notes on architecture decisions. **Acceptance:** A clean checkout starts by following the README. The demo shows a successful analysis, a case with missing data, and a provider error. **Dependencies:** T10 (#11), T11 (#12)

## Starting order

Start with T01 → T02 → T03 → T04. Do not add a message broker, Kubernetes, a frontend, or multiple providers before the first measured result. T05 can start after the first model call, but add retrieval after the baseline is ready.

## Issue template

**Title:** Txx — short task name.

**Goal:** the user outcome.

**Work:** changes to make.

**Acceptance criteria:** observable results and meaningful checks.

**Dependencies:** related task IDs and issue links.
