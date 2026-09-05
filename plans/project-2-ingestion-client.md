# Project 2 — Ingestion Client: Implementation Plan

**Folder:** `02-ingestion-client/`
**Source of truth:** [SPEC.md §6](../SPEC.md#6-project-2--ingestion-client), [DESIGN.md §3](../DESIGN.md#3-sequence-diagram-document-ingestion-flow), [DESIGN.md §5](../DESIGN.md#5-api-contract--project-3)

## Dependencies

- **Consumes:** `shared-schemas/` (from Project 1 Milestone 1); Project 3's HTTP contract (`POST /documents`, `POST /artifacts`) from Project 3 Milestone 5 (contract is stable from Project 3 Milestone 1 onward, so development can start against a mock server before Project 3's real implementation is finished).
- **Called by:** Project 1 (`pipeline.py`, Milestone 6) via a direct in-process function call — not HTTP.
- **Calls out to:** the configured embedding provider (OpenAI or Ollama) and Project 3's Storage API over HTTP.

## How to use this plan

Milestones 1–3 can be developed and fully tested against mocks (no real Project 1 or Project 3 needed). Milestone 4 needs Project 3's real contract available (or a mock server that honors it exactly). Milestone 5 needs Project 1's real pipeline output. Each subtask has an explicit test; a milestone is done only when its gate passes.

---

<a id="p2-m1"></a>
## Milestone 1 — Scaffold + Contract Alignment

**Goal:** get the project set up and its client function signature aligned with what Project 1 will call and what Project 3 expects.

- [ ] 1.1 Scaffold `02-ingestion-client/`: `pyproject.toml`, its own virtual environment, `.gitignore`, `tests/` directory, dependency on `shared-schemas`.
  - **Test:** `pytest` collects with zero errors.
- [ ] 1.2 Define `ingest.py`: `ingest(document: Document, artifacts: list[Artifact]) -> IngestResult`.
  - **Test:** build a fixture `Document` + `Artifact[]` using `shared-schemas` types; call a stubbed `ingest()` and assert no type errors and a well-formed `IngestResult` shape returned.
- [ ] 1.3 Contract test against DESIGN.md §5's exact request/response JSON, using a mock HTTP server (`respx`) standing in for Project 3.
  - **Test:** assert the client sends `POST /documents` and `POST /artifacts` payloads whose JSON shape exactly matches DESIGN.md §5's examples.

- [ ] **Milestone Gate:** signature and contract tests green.

---

<a id="p2-m2"></a>
## Milestone 2 — Embedding Provider Toggle

**Goal:** generate embeddings via either OpenAI or Ollama, selected by config.

- [ ] 2.1 `embedding_provider.py`: OpenAI `text-embedding-3-*` client.
  - **Test:** unit test with the OpenAI HTTP call mocked — asserts correct request payload (model, input text) and that the returned vector has the expected dimension.
- [ ] 2.2 `embedding_provider.py`: Ollama embedding client (e.g. `nomic-embed-text`).
  - **Test:** unit test with the Ollama HTTP call mocked — same assertions as 2.1 for the Ollama path.
- [ ] 2.3 `EMBEDDING_PROVIDER` env toggle selects the implementation; dimension is validated against `EMBEDDING_MODEL` config.
  - **Test:** toggle test asserts the correct provider class is instantiated for each `EMBEDDING_PROVIDER` value; a mismatched-dimension case raises a clear validation error.

- [ ] **Milestone Gate:** both provider unit-test suites and the toggle test are green.

---

<a id="p2-m3"></a>
## Milestone 3 — Retry & Failure Semantics

**Goal:** implement DESIGN.md §8's retry/abort rules for embedding calls.

- [ ] 3.1 Exponential backoff retry (3 attempts) for transient failures (timeout, rate limit).
  - **Test:** simulate a transient failure followed by success; assert 2–3 calls are made and the call ultimately succeeds.
- [ ] 3.2 Immediate abort (no retry) for non-retryable errors (auth error, malformed input).
  - **Test:** simulate an auth error; assert exactly 1 call is made and the document ingestion aborts immediately.
- [ ] 3.3 Persistent-failure path: retries exhausted.
  - **Test:** simulate failure on every attempt; assert exactly 3 calls are made and a clear failure is raised/returned (not silently swallowed).

- [ ] **Milestone Gate:** all three retry/abort unit tests green.

---

<a id="p2-m4"></a>
## Milestone 4 — Storage API Integration

**Goal:** wire the real HTTP calls to Project 3, including partial-failure handling.

- [ ] 4.1 Call `POST /documents` first; capture `document_id`.
  - **Test:** integration test against a mock server (or a running Project 3 test instance) honoring DESIGN.md §5 — asserts `document_id` is captured and used in the next call.
- [ ] 4.2 Call `POST /artifacts` (bulk) with embeddings attached, using the captured `document_id`.
  - **Test:** happy-path integration test asserts a single bulk call is made with all artifacts embedded and the correct `document_id`.
- [ ] 4.3 Partial-failure semantics: if any artifact fails to embed or store, mark the document `status=failed` (via `PATCH` or a dedicated failure endpoint) rather than leaving a partial write.
  - **Test:** forced-failure integration test (one artifact's embed call fails) asserts: no `POST /artifacts` call is made with a partial batch, a failure-status call is made instead, and no duplicate `POST /documents` call occurs on a hypothetical retry.

- [ ] **Milestone Gate:** integration tests green for both the happy path and the partial-failure path, against either a mock server or a real Project 3 test instance.

---

<a id="p2-m5"></a>
## Milestone 5 — End-to-End Wire-Up with Project 1

**Goal:** confirm the real Project 1 → Project 2 → Project 3 handoff works, not just each side's mocks.

- [ ] 5.1 Project 1 replaces its Milestone 6 stub with this project's real `ingest()` function.
  - **Test:** re-run Project 1's Milestone 6.1 end-to-end pipeline test with Project 2's real `ingest()` in place (Project 2's own HTTP calls still mocked); assert Project 2 receives a valid `Document` + `Artifact[]` and produces a correctly-shaped outbound request.
- [ ] 5.2 Full local run: one small fixture document through Project 1 → Project 2 → a real, locally running Project 3 instance.
  - **Test:** manual/documented checkpoint — run the pipeline on one sample document, then query Project 3's `GET /documents/{id}/artifacts` and verify the expected artifact types/counts appear. Optionally automate this as an end-to-end test gated behind the full docker-compose stack (per DESIGN.md §11, kept separate from the fast suite).

- [ ] **Milestone Gate:** Milestone 5.1's re-run test is green; the Milestone 5.2 manual checkpoint has been performed and recorded (or its optional automated version is green).

---

## Definition of "Project 2 done" (v1 scope)

All five milestone gates pass, and at least one real (non-fixture) document has been ingested end-to-end into a running Project 3 instance with correct lineage confirmed via the API.
