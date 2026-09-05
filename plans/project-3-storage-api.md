# Project 3 — Storage CRUD FastAPI: Implementation Plan

**Folder:** `03-storage-api/`
**Source of truth:** [SPEC.md §7](../SPEC.md#7-project-3--storage-crud-fastapi-no-sqlalchemy), [DESIGN.md §5](../DESIGN.md#5-api-contract--project-3), [DESIGN.md §6](../DESIGN.md#6-repository-interface-project-3-internal-design)

## Dependencies

- **Consumes:** `shared-schemas/` (`artifact.py`, `config.py`) — produced by Project 1 Milestone 1. If Project 3 starts before Project 1 exists, Milestone 1 below includes creating a minimal local copy that is later reconciled with Project 1's version.
- **Depends on (runtime):** PostgreSQL (+ pgvector extension), Qdrant (only when `VECTOR_BACKEND=qdrant`).
- **Consumed by:** Project 2 (`02-ingestion-client`) and Project 4 (`04-rag-search-ui`) over HTTP only — no other project touches the database directly. Per SPEC.md §10, this project should be built first since everything else depends on its contract.

## How to use this plan

Work through milestones in order. Milestone 1 defines the API contract and repository interface early and deliberately, so Projects 1/2/4 can develop against a stable shape without waiting for the full storage implementation. Every subtask has an explicit test; a milestone is done only when its gate passes.

---

<a id="p3-m1"></a>
## Milestone 1 — Project Scaffold, Repository Interface, Config-Driven Backend Selection

**Goal:** get a running FastAPI app with the backend-selection seam in place, even before real storage logic exists.

- [ ] 1.1 Scaffold `03-storage-api/`: `pyproject.toml`, its own virtual environment, `.gitignore`, `tests/` directory, dependency on `shared-schemas`.
  - **Test:** `pytest` collects with zero errors; `GET /health` (added in 1.1) returns `200`.
- [ ] 1.2 Define `repositories/base.py`: the `Repository` Protocol (`create_document`, `get_document`, `create_artifacts`, `search`, `get_lineage`, `delete_document`) per DESIGN.md §6.
  - **Test:** a `test_repository_protocol.py` asserts both `PostgresRepository` and `QdrantRepository` stub classes satisfy the Protocol (e.g. via `isinstance` check against a `runtime_checkable` Protocol, or a static method-presence check).
- [ ] 1.3 Wire `VECTOR_BACKEND` env var to a factory function that selects the repository implementation at app startup (stub implementations raise `NotImplementedError` for now).
  - **Test:** unit test asserts the factory returns a `PostgresRepository` instance when `VECTOR_BACKEND=postgres` and a `QdrantRepository` instance when `VECTOR_BACKEND=qdrant`, via dependency override in `TestClient`.

- [ ] **Milestone Gate:** app boots successfully under both `VECTOR_BACKEND` values; health check and factory-selection tests green.

---

<a id="p3-m2"></a>
## Milestone 2 — Postgres Schema & Migrations

**Goal:** the `documents` and `artifacts` tables exist and are reproducible.

- [ ] 2.1 Write raw SQL migration files (`migrations/0001_documents.sql`, `0002_artifacts.sql`, ...) plus a lightweight migration-runner script (no Alembic).
  - **Test:** running the migration runner twice is idempotent (second run is a no-op, no errors).
- [ ] 2.2 Enable the pgvector extension and define the `artifacts.embedding` vector column.
  - **Test:** migration includes `CREATE EXTENSION IF NOT EXISTS vector;`; a post-migration query confirms the extension is installed.
- [ ] 2.3 Add a Postgres service to the repo-root `docker-compose.yml` for local dev (pgvector-enabled image, named volume).
  - **Test:** manual checkpoint — `docker compose up postgres` starts cleanly and accepts connections on `:5432` (documented, not required in CI).

- [ ] **Milestone Gate:** integration test using `testcontainers-python` spins up an ephemeral Postgres, runs all migrations, and asserts both tables exist with the expected columns/types via an `information_schema` query.

---

<a id="p3-m3"></a>
## Milestone 3 — PostgresRepository Implementation

**Goal:** full CRUD + vector search against Postgres.

- [ ] 3.1 `create_document` / `get_document` / `delete_document`.
  - **Test:** testcontainers integration test — create then fetch returns matching fields; delete then fetch returns not-found.
- [ ] 3.2 `create_artifacts` (bulk insert, including the vector column).
  - **Test:** bulk-insert a mixed batch of artifact types; assert row count and field values match input, including `parent_artifact_id` chains.
- [ ] 3.3 `search` (pgvector `<->`/`<=>` operator, optional `artifact_type` filter, `top_k`).
  - **Test:** insert artifacts with known, well-separated embeddings; run `search` with a query vector close to one of them; assert it's returned first (nearest-neighbor ordering correct); assert `artifact_type` filter excludes non-matching types.
- [ ] 3.4 `get_lineage` (document_id → all derived artifacts across types).
  - **Test:** insert one document with a full artifact tree (semantic_chunk → qa_pair/factoid, plus a summary); assert `get_lineage` returns every artifact with correct `parent_artifact_id` relationships intact.

- [ ] **Milestone Gate:** all four PostgresRepository integration test groups green against an ephemeral Postgres container.

---

<a id="p3-m4"></a>
## Milestone 4 — Qdrant Schema & QdrantRepository Implementation

**Goal:** full CRUD + vector search against Qdrant, with Postgres always holding metadata/lineage.

- [ ] 4.1 Qdrant collection setup, vector size driven by `EMBEDDING_MODEL` dimension config.
  - **Test:** integration test asserts collection is created with the configured vector size on first use.
- [ ] 4.2 `create_artifacts`: write metadata to Postgres *and* upsert vector+payload to Qdrant.
  - **Test:** testcontainers (Postgres + Qdrant) integration test — after `create_artifacts`, rows exist in Postgres (metadata, no vector) and matching points exist in Qdrant (vector + payload), keyed by the same `artifact_id`.
- [ ] 4.3 `search`: query Qdrant, then join returned `artifact_id`s back to Postgres for lineage fields.
  - **Test:** same nearest-neighbor test as Milestone 3.3, but asserting results include Postgres-sourced lineage fields (`original_filename`, `source_location`) joined correctly.
- [ ] 4.4 Dual-write failure/rollback handling (DESIGN.md §6): if the Qdrant upsert fails after the Postgres insert, roll back the Postgres rows and return `502 BACKEND_UNAVAILABLE`.
  - **Test:** mock the Qdrant client to raise on upsert; assert the previously-inserted Postgres rows are gone afterward and the API surfaces `502` with `BACKEND_UNAVAILABLE`.

- [ ] **Milestone Gate:** all QdrantRepository integration tests green, including the forced-failure rollback test.

---

<a id="p3-m5"></a>
## Milestone 5 — HTTP API Layer (Routers) + Contract Tests

**Goal:** expose the repository behind the exact HTTP contract in DESIGN.md §5.

- [ ] 5.1 `documents` router: `POST/GET/PUT/DELETE /documents`, `/documents/{id}`.
  - **Test:** `TestClient`/`httpx.AsyncClient` tests for happy path (201/200) and error cases (400 malformed body, 404 unknown id).
- [ ] 5.2 `artifacts` router: `POST/GET/PUT/DELETE /artifacts`, `/artifacts/{id}` (bulk create supported).
  - **Test:** same pattern, plus a bulk-create test with a mixed-type batch.
- [ ] 5.3 `search` router: `POST /search`.
  - **Test:** happy-path test against a pre-seeded repository (using the Milestone 3/4 test fixtures); optional `artifact_type` omitted vs provided, both asserted.
- [ ] 5.4 `GET /documents/{id}/artifacts` (lineage).
  - **Test:** matches the exact response shape in DESIGN.md §5 for a pre-seeded document.
- [ ] 5.5 Standard error envelope (`VALIDATION_ERROR | NOT_FOUND | BACKEND_UNAVAILABLE | INTERNAL_ERROR`) and HTTP status mapping (400/404/502/500).
  - **Test:** one test per error code, asserting both the status code and the JSON error shape.
- [ ] 5.6 Contract tests: assert every endpoint's actual JSON response matches the frozen shapes in DESIGN.md §5, run parametrized across both `VECTOR_BACKEND` modes.
  - **Test:** schema/key-set comparison against the DESIGN.md §5 examples, for `postgres` and `qdrant` backend configurations.

- [ ] **Milestone Gate:** all router tests and contract tests green for both backend modes.

---

<a id="p3-m6"></a>
## Milestone 6 — Observability & NFRs

**Goal:** meet DESIGN.md §8's logging and deployment requirements.

- [ ] 6.1 Structured JSON logging: every request logs method/path/status/latency, plus `document_id`/`artifact_id` where applicable, plus which backend is active.
  - **Test:** a log-capture unit test asserts a sample request emits a JSON log line containing all required fields.
- [ ] 6.2 Dockerfile for `03-storage-api` + wiring into the repo-root `docker-compose.yml`.
  - **Test:** manual checkpoint — `docker compose up storage-api` (with its Postgres/Qdrant dependencies) starts cleanly; `curl localhost:8000/health` returns `200`. Record this checkpoint explicitly since it isn't part of the automated suite.

- [ ] **Milestone Gate:** logging unit test green; manual container smoke test performed and recorded.

---

## Definition of "Project 3 done" (v1 scope)

All six milestone gates pass under both `VECTOR_BACKEND` values, and the manual container smoke test has been run at least once against both backends.
