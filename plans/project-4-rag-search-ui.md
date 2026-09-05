# Project 4 — RAG Search (Streamlit): Implementation Plan

**Folder:** `04-rag-search-ui/`
**Source of truth:** [SPEC.md §8](../SPEC.md#8-project-4--rag-search-streamlit), [DESIGN.md §4](../DESIGN.md#4-sequence-diagram-search-flow), [DESIGN.md §5](../DESIGN.md#5-api-contract--project-3)

## Dependencies

- **Consumes:** `shared-schemas/` (config toggles); Project 3's `POST /search` endpoint (stable from Project 3 Milestone 5 onward — development can start earlier against a mock server honoring the same contract).
- **Requires a populated corpus to be useful end-to-end:** meaningful manual testing of Milestones 2–4 needs at least one document already ingested via Projects 1+2 into a running Project 3 instance. Automated tests use fixture search results instead, so this project's own test suite is not blocked on that.
- Built last per SPEC.md §10, since it only needs Project 3's `/search` endpoint and a working corpus.

## How to use this plan

Each subtask has an explicit test; a milestone is done only when its gate passes. Milestones 1–3 can be fully unit-tested with Project 3 and the LLM mocked. Milestone 4's manual checkpoint needs a real running stack with ingested data.

---

<a id="p4-m1"></a>
## Milestone 1 — Scaffold + LLM/Embedding Provider Toggle

**Goal:** get the Streamlit project set up with both provider toggles in place.

- [ ] 1.1 Scaffold `04-rag-search-ui/`: `pyproject.toml`, its own virtual environment, `.gitignore`, `tests/` directory, `app.py` skeleton, dependency on `shared-schemas`.
  - **Test:** `pytest` collects with zero errors; a minimal Streamlit `AppTest` run of the skeleton renders without exceptions.
- [ ] 1.2 `llm_provider.py`: OpenAI + Ollama toggle for answer generation (`LLM_PROVIDER`/`LLM_MODEL`).
  - **Test:** unit tests per provider with the HTTP call mocked — correct request payload sent, response text parsed correctly.
- [ ] 1.3 Reuse the embedding-provider logic (same `EMBEDDING_PROVIDER`/`EMBEDDING_MODEL` pattern as Project 2) for embedding the user's query at search time.
  - **Test:** unit test asserts the same provider/model config used at ingestion time is used here (a config-consistency check), with the HTTP call mocked.

- [ ] **Milestone Gate:** all provider unit tests green (LLM and embedding), including the toggle-selection tests.

---

<a id="p4-m2"></a>
## Milestone 2 — Query → Embed → Search Integration

**Goal:** wire the query path up to Project 3's `/search` endpoint.

- [ ] 2.1 Embed the user's query text using the Milestone 1.3 provider.
  - **Test:** unit test asserts the query string is passed to the embedding provider and a vector is returned.
- [ ] 2.2 Call `POST /search` with `{query_vector, artifact_type?, top_k}`.
  - **Test:** unit test mocking Project 3's `/search` response per DESIGN.md §5's exact shape — asserts the request is built correctly and the response is parsed into the expected result objects.
- [ ] 2.3 Handle search failures inline: on error, show a message and do **not** attempt an LLM call (per DESIGN.md §8).
  - **Test:** forced search-failure unit test asserts an error state is set/rendered and the mocked LLM provider is never invoked.

- [ ] **Milestone Gate:** search-integration unit tests green, including the failure short-circuit test.

---

<a id="p4-m3"></a>
## Milestone 3 — Prompt Assembly & Answer Generation

**Goal:** turn search results into a cited answer.

- [ ] 3.1 Assemble a prompt from the top-k retrieved artifacts, including citation metadata.
  - **Test:** unit test asserts the assembled prompt contains the expected chunk content and citation markers for a fixture set of search results.
- [ ] 3.2 Call the configured LLM provider (Milestone 1.2) for answer generation.
  - **Test:** unit test with the LLM mocked asserts the prompt from 3.1 is passed through and the response text is captured.
- [ ] 3.3 Render the answer plus source citations (`original_filename` + `source_location`) in the UI.
  - **Test:** Streamlit `AppTest` asserts the rendered page includes the answer text and a citation list matching the fixture search results' `original_filename`/`source_location` fields.

- [ ] **Milestone Gate:** prompt-assembly, generation, and rendering tests all green.

---

<a id="p4-m4"></a>
## Milestone 4 — Streamlit App Smoke Test & NFRs

**Goal:** full app wiring, observability, and deployability.

- [ ] 4.1 Full `app.py` wiring: query box → results → answer → citations, using Milestones 1–3 together.
  - **Test:** Streamlit `AppTest` smoke test — simulate typing a query and submitting; assert no exceptions and all expected UI elements (query box, results, answer, citations) render, with Project 3 and the LLM mocked.
- [ ] 4.2 Structured logging: query (hashed/truncated if sensitive), retrieved artifact count, LLM latency, per DESIGN.md §8.
  - **Test:** log-capture unit test asserts a sample query emits a JSON log line with the required fields.
- [ ] 4.3 Dockerfile for `04-rag-search-ui` + wiring into the repo-root `docker-compose.yml`.
  - **Test:** manual checkpoint — `docker compose up rag-search-ui` starts cleanly; open the app in a browser, run one real query against a populated Project 3 instance, and confirm an answer with citations is rendered. Record this checkpoint explicitly.

- [ ] **Milestone Gate:** the `AppTest` smoke test and logging unit test are green; the manual container checkpoint has been performed and recorded.

---

## Definition of "Project 4 done" (v1 scope)

All four milestone gates pass, and at least one real end-to-end query (against a real ingested document and a real LLM/embedding provider, not mocks) has been run manually and its answer+citations verified as sensible.
