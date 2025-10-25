# Plan: Supabase pgvector & Jina Integration

**Branch**: `[001-pgvector]` | **Spec**: `spec/001_pgvector/scope.md` | **Architecture**: `spec/001_pgvector/architecture.md`  
**Goal**: Deliver Supabase/pgvector storage with Jina embeddings that matches current Milvus/OpenAI behavior across CLI, MCP server, VS Code, and Chrome surfaces while satisfying SC-1…SC-4.

## Phase Structure

- **Phase 0 – Research & Setup**: Resolve outstanding clarifications, validate external dependencies, and prepare environments.
- **Phase 1 – Foundations**: Build shared infrastructure (schema, adapters, credential decisions) required by all user stories.
- **Phase 2+ – User Stories**: Execute slices aligned to US1–US4 with independent checkpoints.
- **Final Phase – Hardening & Launch**: Benchmarks, documentation, migration tooling, and launch gates.

`[P]` indicates tasks that can run in parallel. Each task lists priority (P0/P1/P2), dependencies, deliverables, and acceptance criteria. IDs reference scope (US#, SC#) and architecture decisions (OPT-, R#).

---

## Phase 0 – Research & Setup (P0)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| P0-T1 | Gather Supabase Free-tier throughput/extension data (feeds RA2). | P0 | None | Report covering connection caps, `vector` enablement steps, read-only triggers. | Confirms Free-tier support boundaries. |
| P0-T2 | Hybrid search feasibility study (feeds RA1). | P0 | P0-T1 | Findings on pg_trgm cost/latency + recommendation. | Decision logged: dense-only launch vs hybrid. |
| P0-T3 | Record telemetry deferral and backlog follow-up (feeds RA3). | P1 | None | Memo capturing deferred alerting scope + owners. | Stakeholders aligned on deferral; no MVP action. |
| P0-T4 | Credential & environment audit across surfaces (feeds RA4). | P0 | None | Storage/relay matrix + risks. | Reviewed by security; informs browser relay plan. |
| P0-T5 | Local Postgres/pgvector fallback validation (feeds RA5). | P1 | P0-T1 | Compose + scripts matching migrations. | Indexed search runs locally within 30 min; fallback deemed viable. |

**Checkpoint**: Research reports archived, NC-1…NC-3 decisions documented (or scheduled), credential risks understood.

---

## Phase 1 – Foundations (P0)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| P1-T1 | Design pgvector schema + migrations (supports US1, US2). | P0 | Phase 0 checkpoint | SQL migration files + ER diagram. | Handles 768/1024 dims, rerunnable migrations, reviewed by 2 senior engineers. |
| P1-T2 | Implement `PgVectorDatabase` for Node contexts. | P0 | P1-T1 | TS module + unit tests. | Tests cover upsert, filtering, error handling; synthetic search p95 <400 ms on 50k rows. |
| P1-T3 | Decide and implement browser write strategy (direct vs relay). | P0 | P0-T4, P1-T1 | Decision doc + prototype. | VS Code/Chrome dev build shows authenticated writes; decision approved in arch review. |
| P1-T4 | Build `JinaEmbedding` client with batching, retries, dimension detection. | P0 | None | TS module + mocked tests. | Handles 429 backoff, detects dimension, logs usage estimates. |
| P1-T5 | Feature flag + configuration plumbing for Supabase/Jina mode. | P0 | P1-T2, P1-T4 | Config schema updates, CLI/MCP/IDE settings. | Allows toggling providers without restart; defaults follow scope assumptions. |
| P1-T6 | Health check endpoints (Supabase + Jina). | P1 | P1-T2, P1-T4 | CLI health command + guidance copy. | CLI command validates connections, emits clear guidance on failure; no telemetry streaming. |

**Checkpoint**: Schema + adapters ready, embedding client functional, feature flag toggles exist, health checks pass in dev environments.

---

## Phase 2 – User Story Execution

### Phase 2A – US1: Platform Engineer Supabase Setup (P0)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| US1-T1 | Guided setup flow in CLI (collect Supabase URL/key, verify). | P0 | Phase 1 checkpoint | CLI prompts, validation messaging. | Handles success/failure cases, informs next steps for extension enablement. |
| US1-T2 | Automatic schema creation + migrations trigger. | P0 | US1-T1 | Migration runner integrated into CLI/MCP startup. | Creates tables/indexes without manual SQL; idempotent. |
| US1-T3 | Repository indexing pipeline hooked to PgVector adapter. | P0 | P1-T2, P1-T4 | End-to-end indexing command. | Indexing completes for sample repo, exposes progress, surfaces errors with actionable guidance. |
| US1-T4 | Health & diagnostics reporting (Supabase + Jina). | P1 | P1-T6 | CLI health command output. | Reports status for connections, extensions, and embedding provider (CLI-first). |

**Checkpoint**: Supabase Free-tier repo indexing completes via CLI without manual SQL; fallback guidance documented when limits exceeded (ties to US1 acceptance).

### Phase 2B – US2: Free-Tier Experience & Usage Transparency (P1)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| US2-T1 | Free-tier capability detection & messaging. | P1 | P0-T5, US1 checkpoint | CLI-first messaging + limit detection. | Alerts user when hitting limits; suggests throttled/dense-only mode or local fallback. |
| US2-T2 | Usage dashboard (row count + Jina token estimates). | P1 | P1-T5, P1-T6 | CLI panel first; extend to MCP/VS Code/Chrome post-CLI. | Displays metrics before first indexing completes (SC-4). |
| US2-T3 | Throttled indexing mode for Free tier. | P1 | US1-T3 | Config option limiting batch size/concurrency. | Keeps operations within Free-tier caps; logs when throttling triggers. |
| US2-T4 | Guided fallback to local Postgres/pgvector (Docker). | P1 | P0-T5 | CLI/Docs walkthrough + scripts. | User can pivot to local deployment with same schema; messaging ties to capacity warnings. |

**Checkpoint**: Free-tier repo indexes without exceeding limits OR recommended fallback executed; users see usage data in-product (SC-4).

### Phase 2C – US3: Security & Key Rotation (P1)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| US3-T1 | Secret storage updates (CLI env, VS Code SecretStorage, Chrome). | P0 | P0-T4 | Implementation details per client. | Secrets encrypted at rest; read-only mode available. |
| US3-T2 | Key rotation workflow + docs. | P1 | US3-T1 | CLI command + extension UI for updating keys. | Rotation causes no downtime; logs record rotation event. |
| US3-T3 | Audit log export guidance. | P1 | US3-T2 | Docs + support macros. | Security admin can retrieve access logs across clients. |

**Checkpoint**: Rotation path validated end-to-end; security sign-off complete (ties to SC-1 adoption and US3 acceptance).

### Phase 2D – US4: Provider Comparison & Rollback (P2)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| US4-T1 | Provider toggle UI across surfaces. | P2 | P1-T5 | Settings UI updates. | Users switch providers without restart; state persists. |
| US4-T2 | Benchmark harness enhancements. | P2 | US1-T3, US2-T2 | Scripts comparing Jina vs legacy provider. | Outputs recall/latency metrics, auto-publishes chart. |
| US4-T3 | Rollback workflow (config + docs). | P2 | US4-T1 | CLI command + docs. | Rollback completes in <5 min; warns when embeddings must be regenerated. |

**Checkpoint**: Provider comparison + rollback validated; benchmark results documented (supports SC-2, US4).

---

## Final Phase – Migration, Validation & Launch (P1/P2)

| ID | Task | Priority | Dependencies | Deliverable | Acceptance |
|----|------|----------|--------------|-------------|------------|
| LP-T1 | Milvus → Supabase migration CLI (optional but recommended). | P1 | Phase 2 checkpoints | CLI command with resumable checkpoints. | Migrates sample repos (S/M/L) with <5% recall delta. |
| LP-T2 | Regression benchmarking & performance validation. | P1 | US1–US4 checkpoints | Report + charts. | Meets SC-2, SC-3 thresholds; deviations signed off. |
| LP-T3 | Documentation & support materials. | P1 | LP-T2, US3-T3 | README/docs updates, troubleshooting guide, Zendesk macros. | Includes setup checklist, error catalog, cost guidance; reviewed by docs & support leads. |
| LP-T4 | Launch readiness review. | P0 | All above | Checklist (below) signed off. | All gates satisfied; go/no-go recorded. |
| LP-T5 | Post-MVP nice-to-haves (usage metering enhancements, cost guardrail alerts). | P2 | LP-T2 | UI improvements + notification thresholds. | Displays Supabase/Jina usage with <10% error; optional alerts configurable after GA. |
| LP-T6 | Data durability & backup guidance (Supabase vs local). | P1 | LP-T3 | Docs + in-product links. | Explains Free-tier read-only triggers, lack of backups, and fallback workflow before GA. |

---

## Phase Gates

- **Phase 0 Gate**: NC-1..NC-3 resolved; research reports archived; credential audit approved by security.
- **Phase 1 Gate**: Schema + adapters pass unit tests; embedding client stable; feature flag + health checks merged.
- **Phase 2A Gate**: Supabase Free-tier indexing completes E2E (CLI-first); if limits are hit, fallback guidance executed before proceeding.
- **Phase 2B Gate**: Free-tier usage panel live; throttled mode prevents overages; fallback doc tested for customers exceeding caps.
- **Phase 2C Gate**: Secret storage + rotation sign-off by security; audit docs published.
- **Phase 2D Gate**: Benchmark harness produces comparison report; provider toggle stable.
- **Final Gate**: Launch readiness checklist fully checked.

---

## Launch Readiness Checklist

- [ ] P0/P1 tasks complete; demo artifacts recorded.
- [ ] SC-1–SC-4 validated via telemetry/benchmarks.
- [ ] Security review approves credential handling + rotation.
- [ ] Support playbooks, troubleshooting guides, and onboarding docs published.
- [ ] Migration plan (if applicable) tested; rollback instructions documented.
- [ ] Feature flag strategy and rollout sequencing defined (beta, GA, fallback).

---

## Parallelization Notes

- P0-T1..T5 can run in parallel except where outputs feed later tasks (e.g., P0-T2 relies on P0-T1’s extension data; P0-T5 waits for schema decisions).
- In Phase 1, P1-T2 (Node adapter) and P1-T4 (embedding client) can proceed in parallel; P1-T3 waits on credential audit.
- During Phase 2, each user story’s tasks can run in parallel once prerequisites met; mark `[P]` within execution tickets when splitting further.

Following this plan ensures each user story remains independently deliverable, success metrics stay traceable, and launch gates capture the cross-team dependencies required for production readiness.***
