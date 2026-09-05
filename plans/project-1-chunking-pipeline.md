# Project 1 — Chunking & Derivative Artifact Generation: Implementation Plan

**Folder:** `01-chunking-pipeline/`
**Source of truth:** [SPEC.md §5](../SPEC.md#5-project-1--chunking--derivative-artifact-generation), [DESIGN.md §3](../DESIGN.md#3-sequence-diagram-document-ingestion-flow)

## Dependencies

- **Produces:** `shared-schemas/` (`artifact.py`, `config.py`) — this project defines the contract first, since it's the first producer of `Document`/`Artifact` objects. Project 3 consumes the same contract from Milestone 1 onward.
- **Depends on (runtime):** Ollama server (`OLLAMA_BASE_URL`) for all derivative generators except semantic chunking.
- **Hands off to:** Project 2 (`02-ingestion-client`) via a direct in-process function call — Milestone 6 is the integration point. Until Project 2 exists, Milestone 6 tests against a local stub matching Project 2's expected signature.

## How to use this plan

Work through milestones in order. Each subtask lists the automated (or explicitly manual) check that must pass before it's marked done. A milestone is not complete until every subtask checkbox **and** the milestone gate are green. Do not start the next milestone without confirming the current gate passes.

---

<a id="p1-m1"></a>
## Milestone 1 — Shared Artifact/Document Contract + Project Scaffold

**Goal:** establish the data contract every other project will import, and get the project importable/testable.

- [ ] 1.1 Create `shared-schemas/artifact.py`: `ArtifactType` enum (`semantic_chunk`, `contextual_chunk`, `summary`, `raptor_node`, `qa_pair`, `factoid`), `Artifact` and `Document` models (pydantic) per SPEC.md §4.
  - **Test:** `shared-schemas/tests/test_artifact.py` — valid payloads construct successfully; missing required fields raise `ValidationError`; enum rejects unknown `artifact_type` values; `.model_dump()` round-trips against the exact JSON shapes in DESIGN.md §5.
- [ ] 1.2 Create `shared-schemas/config.py`: pydantic settings models for `VECTOR_BACKEND`, `EMBEDDING_PROVIDER`, `EMBEDDING_MODEL`, `LLM_PROVIDER`, `LLM_MODEL`, `GENERATION_MODEL` (+ per-type overrides), `POSTGRES_DSN`, `QDRANT_URL`, `OLLAMA_BASE_URL`, `OPENAI_API_KEY`, per DESIGN.md §7.
  - **Test:** `test_config.py` — defaults match DESIGN.md §7 table; invalid enum value (e.g. `VECTOR_BACKEND=mysql`) raises a validation error; env vars override defaults correctly (using `monkeypatch.setenv`).
- [ ] 1.3 Scaffold `01-chunking-pipeline/` project: `pyproject.toml`, its own virtual environment, `.gitignore`, `tests/` directory, local path dependency on `shared-schemas`.
  - **Test:** `pytest` collects and runs with zero errors (even with no real tests yet, a placeholder `test_smoke.py::test_imports` passes); `python -c "from shared_schemas.artifact import Artifact"` succeeds inside the project's venv.
- [ ] 1.4 Define `pipeline.py` skeleton: `run_pipeline(file_path: Path) -> tuple[Document, list[Artifact]]`, body raises `NotImplementedError`.
  - **Test:** `test_pipeline_skeleton.py` asserts calling `run_pipeline()` with a dummy path raises `NotImplementedError`.

- [ ] **Milestone Gate:** all four subtask tests green; `shared-schemas` package importable from every other project folder once they're scaffolded.

---

<a id="p1-m2"></a>
## Milestone 2 — Docling Conversion

**Goal:** turn raw PDF/HTML/TXT into normalized Markdown with page/section metadata.

- [ ] 2.1 Implement `docling_convert.py`: format detection (pdf/html/txt) and conversion to normalized Markdown.
  - **Test:** `tests/fixtures/` sample files (one small PDF, one HTML, one TXT) each convert to non-empty Markdown without exceptions.
- [ ] 2.2 Preserve page/section boundaries as structured metadata (feeds `source_location` downstream).
  - **Test:** for the PDF and HTML fixtures, assert returned metadata includes page numbers and/or section headings with correct ordering; TXT fixture returns a reasonable single-section fallback.
- [ ] 2.3 Implement long-document section-split threshold (configurable page/token count).
  - **Test:** a "long" fixture (exceeds threshold) is split into multiple sections pre-chunking; a "short" fixture (below threshold) is returned as a single unit. Both asserted via a parametrized test.

- [ ] **Milestone Gate:** all three Docling tests green across all three formats, including both long- and short-document branches.

---

<a id="p1-m3"></a>
## Milestone 3 — Semantic Chunking (Chonkie)

**Goal:** produce `semantic_chunk` artifacts from normalized Markdown.

- [ ] 3.1 Implement `chunkers/semantic.py` using Chonkie.
  - **Test:** feeding the Milestone 2 Markdown fixture produces N chunks (N > 0), each with non-empty `content`.
- [ ] 3.2 Populate `source_location` (page/section/offset) per chunk; set `artifact_type=semantic_chunk`.
  - **Test:** every produced chunk has a valid, non-null `source_location`; concatenating chunk offset ranges covers the source Markdown without large gaps (a coverage assertion, not exact reconstruction).

- [ ] **Milestone Gate:** semantic chunking unit tests green; one manual spot-check of chunk boundaries against a real sample document, recorded in the PR/commit description.

---

<a id="p1-m4"></a>
## Milestone 4 — Ollama-Driven Derivative Generators

**Goal:** produce `contextual_chunk`, `summary`, `qa_pair`, and `factoid` artifacts.

- [ ] 4.1 Build an Ollama client wrapper honoring `OLLAMA_BASE_URL` and per-artifact-type `GENERATION_MODEL` overrides.
  - **Test:** wrapper unit test with the HTTP call mocked (`respx`/`unittest.mock`) — correct URL, model name, and prompt payload sent; response parsed correctly.
- [ ] 4.2 Implement `contextual.py` (contextual chunk augmentation).
  - **Test:** mocked-Ollama unit test asserts `artifact_type=contextual_chunk`, `model_metadata` populated, content includes the augmented context.
- [ ] 4.3 Implement `summarize.py` (abstractive summary per document/section).
  - **Test:** mocked-Ollama unit test asserts `artifact_type=summary`, one summary per document or per major section as configured.
- [ ] 4.4 Implement `qa_pairs.py` (synthetic Q/A per chunk).
  - **Test:** mocked-Ollama unit test asserts `artifact_type=qa_pair`, `parent_artifact_id` correctly links back to the source `semantic_chunk`.
- [ ] 4.5 Implement `factoids.py` (atomic factual statements per chunk).
  - **Test:** mocked-Ollama unit test asserts `artifact_type=factoid`, `parent_artifact_id` linkage correct, multiple factoids per chunk supported.

- [ ] **Milestone Gate:** all four generator unit-test suites green, with `parent_artifact_id` linkage explicitly verified for QA pairs and factoids. Optional: one real-Ollama integration test per generator, gated behind an env flag (e.g. `RUN_OLLAMA_INTEGRATION=1`) so it doesn't block CI.

---

<a id="p1-m5"></a>
## Milestone 5 — RAPTOR (Hierarchical Clustering + Recursive Summarization)

**Goal:** build a hierarchical summary tree for long documents.

- [ ] 5.1 Implement `raptor.py`: clustering over chunk embeddings + recursive summarization into a tree of `raptor_node` artifacts.
  - **Test:** unit test with mocked Ollama calls and a small fixed set of chunk fixtures asserts the resulting tree has more than one level, and every leaf node's `parent_artifact_id` points to a real `semantic_chunk` id from the input.
- [ ] 5.2 Wire RAPTOR to trigger automatically whenever the Milestone 2 long-document branch is taken.
  - **Test:** running the long-doc fixture through the combined chunkers produces `raptor_node` artifacts; running the short-doc fixture produces none.

- [ ] **Milestone Gate:** RAPTOR unit tests green; long-document fixture run through Milestones 2+3+5 together produces a well-formed tree (leaf-to-root traceable via `parent_artifact_id`).

---

<a id="p1-m6"></a>
## Milestone 6 — Pipeline Orchestration & Handoff to Project 2

**Goal:** wire every prior milestone into `pipeline.py` and hand results to Project 2.

- [ ] 6.1 Implement `pipeline.py` fully: Docling → chunkers → generators → assemble `Document` + `Artifact[]` → call Project 2's client function.
  - **Test:** end-to-end unit test (all Ollama calls mocked) running the full pipeline against each fixture format (pdf/html/txt) and both doc-length branches; assert a well-formed `Document` + mixed-type `Artifact[]` list is produced with correct cross-linkage.
- [ ] 6.2 Error handling: on any generator exception, log and abort *before* calling Project 2 — never hand off partial artifacts (per DESIGN.md §8).
  - **Test:** force one generator to raise inside the mocked pipeline; assert Project 2's client stub is never called and the document is marked/logged as failed.
- [ ] 6.3 Integrate with Project 2's real `ingest()` client once `02-ingestion-client` Milestone 1 exists (replace the stub from 6.1).
  - **Test:** re-run the Milestone 6.1 end-to-end test against the real Project 2 client with Project 2's own HTTP calls mocked; assert Project 2 receives a valid `Document` + `Artifact[]` matching the shared schema.

- [ ] **Milestone Gate:** full pipeline test green on all fixture formats and both long/short branches; failure-path test green; Project 2 hand-off test green once Project 2 is available.

---

## Definition of "Project 1 done" (v1 scope)

All six milestone gates pass, and one real sample document (not a fixture) has been run end-to-end through the pipeline at least once with real Ollama models, with output manually inspected for sanity (recorded, not necessarily automated).
