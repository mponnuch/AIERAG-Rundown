# Master Implementation Tracker

Execution controller for the Enterprise RAG Solution's four projects. This document tracks **when** each milestone can start and whether it's done; each linked project plan tracks **how** (tasks, subtasks, tests) within that milestone.

## Linked Plans

- [Project 1 — Chunking & Derivative Artifact Generation](project-1-chunking-pipeline.md) (`01-chunking-pipeline/`)
- [Project 2 — Ingestion Client](project-2-ingestion-client.md) (`02-ingestion-client/`)
- [Project 3 — Storage CRUD FastAPI](project-3-storage-api.md) (`03-storage-api/`)
- [Project 4 — RAG Search UI](project-4-rag-search-ui.md) (`04-rag-search-ui/`)

Background: [SPEC.md](../SPEC.md) (scope/data model), [DESIGN.md](../DESIGN.md) (runtime contracts, sequence diagrams, testing strategy).

## How to use this tracker

1. Work wave by wave, top to bottom — don't start a milestone whose dependencies aren't checked off yet.
2. Within a wave, the listed milestones have no dependency on each other and can be worked in parallel (e.g. by different people, or sequentially if it's just you).
3. For each milestone: open its linked project plan, complete every task/subtask checkbox, run that checkpoint's tests immediately, and only check off the **Milestone Gate** once its tests pass.
4. Come back here and check off the milestone in the wave list below only after its Gate is checked in the project plan — this file and the per-project files should never disagree about what's done.
5. Get sign-off before moving to the next wave (per current project convention: pause for approval at the end of each milestone).

---

## Cross-Project Dependency Matrix

| Milestone | Depends on | Unblocks |
|---|---|---|
| P1.M1 — Shared contract + scaffold | — (start here) | P1.M2, P3.M1, P2.M1, P4.M1 |
| P1.M2 — Docling conversion | P1.M1 | P1.M3 |
| P1.M3 — Semantic chunking | P1.M2 | P1.M4, P1.M5 |
| P1.M4 — Ollama derivative generators | P1.M3 | P1.M6 |
| P1.M5 — RAPTOR | P1.M3, P1.M4 | P1.M6 |
| P1.M6 — Orchestration & handoff | P1.M2, P1.M3, P1.M4, P1.M5, P2.M1 | P2.M5 |
| P3.M1 — Scaffold, repo interface, config selection | P1.M1 | P3.M2 |
| P3.M2 — Postgres schema/migrations | P3.M1 | P3.M3, P3.M4 |
| P3.M3 — PostgresRepository | P3.M2 | P3.M5 |
| P3.M4 — Qdrant + dual-write | P3.M2 | P3.M5 |
| P3.M5 — HTTP routers + contract tests | P3.M3, P3.M4 | P3.M6, P2.M4, P4.M2 (real), P4.M3 |
| P3.M6 — Observability & NFRs | P3.M5 | P2.M5, P4.M4 (full manual checkpoint) |
| P2.M1 — Scaffold + contract alignment | P1.M1 | P2.M2, P1.M6 (as a stub) |
| P2.M2 — Embedding provider toggle | P2.M1 | P2.M3 |
| P2.M3 — Retry & failure semantics | P2.M2 | P2.M4 |
| P2.M4 — Storage API integration | P2.M3, P3.M5 | P2.M5 |
| P2.M5 — E2E wire-up with Project 1 | P1.M6, P2.M4, P3.M6 | P4.M4 (manual checkpoint needs a populated corpus) |
| P4.M1 — Scaffold + provider toggles | P1.M1 | P4.M2 |
| P4.M2 — Query → embed → search integration | P4.M1, P3.M1 (contract shape; real validation needs P3.M5) | P4.M3 |
| P4.M3 — Prompt assembly & generation | P4.M2 | P4.M4 |
| P4.M4 — Streamlit smoke test & NFRs | P4.M3 (automated parts); P2.M5 + P3.M6 (manual checkpoint) | — (final) |

---

## Execution Waves

### Wave 1 — Foundation

- [ ] [P1.M1 — Shared Artifact/Document Contract + Project Scaffold](project-1-chunking-pipeline.md#p1-m1)

### Wave 2 — Independent scaffolds off the shared contract

- [ ] [P3.M1 — Project Scaffold, Repository Interface, Config-Driven Backend Selection](project-3-storage-api.md#p3-m1)
- [ ] [P2.M1 — Scaffold + Contract Alignment](project-2-ingestion-client.md#p2-m1)
- [ ] [P4.M1 — Scaffold + LLM/Embedding Provider Toggle](project-4-rag-search-ui.md#p4-m1)
- [ ] [P1.M2 — Docling Conversion](project-1-chunking-pipeline.md#p1-m2)

### Wave 3 — First layer of real logic

- [ ] [P3.M2 — Postgres Schema & Migrations](project-3-storage-api.md#p3-m2)
- [ ] [P2.M2 — Embedding Provider Toggle](project-2-ingestion-client.md#p2-m2)
- [ ] [P1.M3 — Semantic Chunking (Chonkie)](project-1-chunking-pipeline.md#p1-m3)
- [ ] [P4.M2 — Query → Embed → Search Integration](project-4-rag-search-ui.md#p4-m2) *(against mock/frozen P3 contract shape; re-validate once P3.M5 lands)*

### Wave 4 — Backend implementations + generation logic

- [ ] [P3.M3 — PostgresRepository Implementation](project-3-storage-api.md#p3-m3)
- [ ] [P3.M4 — Qdrant Schema & QdrantRepository Implementation](project-3-storage-api.md#p3-m4)
- [ ] [P2.M3 — Retry & Failure Semantics](project-2-ingestion-client.md#p2-m3)
- [ ] [P1.M4 — Ollama-Driven Derivative Generators](project-1-chunking-pipeline.md#p1-m4)
- [ ] [P1.M5 — RAPTOR (Hierarchical Clustering + Recursive Summarization)](project-1-chunking-pipeline.md#p1-m5)
- [ ] [P4.M3 — Prompt Assembly & Answer Generation](project-4-rag-search-ui.md#p4-m3)

### Wave 5 — API surface stabilizes; pipeline completes

- [ ] [P3.M5 — HTTP API Layer (Routers) + Contract Tests](project-3-storage-api.md#p3-m5)
- [ ] [P1.M6 — Pipeline Orchestration & Handoff to Project 2](project-1-chunking-pipeline.md#p1-m6)
- [ ] [P4.M4 — Streamlit App Smoke Test & NFRs](project-4-rag-search-ui.md#p4-m4) *(automated smoke test + logging only — manual checkpoint waits for Wave 7)*

### Wave 6 — Real integration against the stable API

- [ ] [P3.M6 — Observability & NFRs](project-3-storage-api.md#p3-m6)
- [ ] [P2.M4 — Storage API Integration](project-2-ingestion-client.md#p2-m4)

### Wave 7 — Full end-to-end

- [ ] [P2.M5 — End-to-End Wire-Up with Project 1](project-2-ingestion-client.md#p2-m5)
- [ ] P4.M4 manual checkpoint completed — real query run against a real ingested corpus (needs P2.M5 + P3.M6 done first)

---

## Platform v1 — Definition of Done

- [ ] All 21 milestone gates above are checked off
- [ ] [Project 1's "done" definition](project-1-chunking-pipeline.md#definition-of-project-1-done-v1-scope) met
- [ ] [Project 2's "done" definition](project-2-ingestion-client.md#definition-of-project-2-done-v1-scope) met
- [ ] [Project 3's "done" definition](project-3-storage-api.md#definition-of-project-3-done-v1-scope) met
- [ ] [Project 4's "done" definition](project-4-rag-search-ui.md#definition-of-project-4-done-v1-scope) met
