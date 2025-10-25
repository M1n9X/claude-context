# Architecture: Supabase pgvector & Jina Embedding Integration

## Summary

Scope establishes a need to deliver a Supabase-backed storage path with Jina embeddings that matches current Milvus/OpenAI behavior across CLI, MCP server, VS Code, and Chrome. This document picks a technical strategy, documents alternatives, and identifies the research, dependencies, and gates required before planning.

## Solution Guardrails & Assumptions

- Maintain existing `VectorDatabase` and `Embedding` abstractions so clients do not require API changes and provider toggles stay transparent (supports scope A2).
- Customers manage their own Supabase projects; we handle schema provisioning, migrations, and validation of required extensions (`pgvector`, optionally `pg_trgm`).
- Shipping dense-only search is acceptable if hybrid parity is documented and scheduled (NC-1 default).
- Environment secrets flow through existing secure stores per surface (Node env vars, VS Code SecretStorage, Chrome storage + OS keychain). Provide a compliant relay or read-only path when service keys cannot live client-side (supports scope A3).
- Jina’s `jina-embeddings-v2-base-code` exposes 768–1024 dimensions over REST; adapter must detect dimension at runtime.

## Options Considered

| Option ID | Description | Cost/Effort | Ops Overhead | Security | Performance | Maintainability | Notes |
|-----------|-------------|-------------|--------------|----------|-------------|-----------------|-------|
| OPT-1 | Direct SQL access to Supabase/Postgres via `pg` (Node) and Supabase JS REST client (browser). | **Engineering**: 8–10 weeks core adapter + embeddings + clients. | Medium – own migrations, monitoring. | High (full control over credential paths). | High (tunable SQL, close to data). | High – same code path works for self-hosted Postgres. | **Recommended** for portability and control. |
| OPT-2 | Wrap storage/search in Supabase Edge Functions or RPC endpoints. | 10–12 weeks (functions + deployment tooling). | High – users must deploy functions, monitor. | Medium – service role stored in Supabase. | Medium – extra latency, less tuning. | Medium – function code to version. | Rejected: higher friction for customers. |
| OPT-3 | Maintain Milvus and mirror into Supabase via sync bridge. | 6–8 weeks initial + ongoing sync ops. | Very High – dual storage, monitoring. | High – but doubles secrets. | Medium – read latency OK, writes slower. | Low – constant maintenance burden. | Rejected: undermines target objective. |

### Recommended Approach (OPT-1)

Direct SQL access aligns with goals of portability (self-hosted Postgres later), lower customer friction, and better control over performance. Requires disciplined SQL authoring and shared migrations, but avoids forcing customers to deploy serverless functions.

## Research Agenda (from Scope / New)

1. **Hybrid search feasibility (NC-1)**: Benchmark pgvector + pg_trgm blend, cost vs. dense-only, and tuning requirements.
2. **Supabase plan requirements (NC-2)**: Measure Free vs. Pro tier indexing throughput, extension availability, connection caps.
3. **Usage telemetry for cost alerts (NC-3)**: Identify APIs or logs from Supabase and Jina for real-time usage.
4. **Browser credential relay latency**: Quantify overhead when VS Code/Chrome rely on relay writes; confirm <500 ms p95 (validates A3).

## Architecture Overview

### Components

1. **Embedding Service Layer (`JinaEmbedding`)**
   - Extends existing embedding abstraction.
   - Handles batching (32 items default), retry/backoff for 429/5xx, detects vector dimension during warm-up.
   - Emits usage estimates for cost transparency (feeds SC-4).

2. **Vector Storage Layer (`PgVectorDatabase`)**
   - Node contexts: `pg` pool with SSL, statement timeouts, connection retries.
   - Browser contexts: Supabase JS client for reads, relay for writes when anon key insufficient.
   - Schema per repository: `code_chunks_{hash}` with metadata JSONB, vector column, optional `tsvector`.
   - Supports upsert semantics, soft-delete, and incremental indexing markers.

3. **Configuration & Feature Flag Layer**
   - Extend config loaders to accept Supabase credentials (URL, service role, anon key) and embedding provider choice.
   - Feature flag toggles between Milvus/OpenAI and Supabase/Jina until migration complete.

4. **Health & Observability**
   - CLI/MCP run `SELECT 1`, verify `pgvector` presence, and check row counts.
   - Embedding health ping hits Jina with minimal payload to confirm credentials.
   - Clients emit telemetry for indexing progress, latency, and provider usage (opt-in).

### Data Model Snapshot

| Table | Columns | Notes |
|-------|---------|-------|
| `code_chunks_{repo}` | `id UUID PK`, `relative_path TEXT`, `content TEXT`, `metadata JSONB`, `start_line INT`, `end_line INT`, `embedding VECTOR(768/1024)`, `lexeme TSVECTOR (optional)` | Partitioned per repo for access control and cleanup. |
| `chunk_state_{repo}` | `id UUID`, `hash TEXT`, `updated_at TIMESTAMPTZ`, `status TEXT` | Tracks incremental indexing checkpoints. |
| `settings_global` | `repo_id UUID`, `provider TEXT`, `dimension INT`, `migration_version INT` | Stores migration metadata and defaults. |

Indexes:
- `btree(relative_path)` for path filters.
- `GIN (lexeme)` when lexical search enabled.
- `btree(metadata->>'language')` for language-specific filters (optional).

### Integration Flows

1. **Initial Setup**
   - Validate Supabase credentials via lightweight query.
   - Run idempotent migrations (SQL files versioned, executed via CLI).
   - Probe Jina embedding dimension; store in `settings_global`.
2. **Indexing**
   - Chunker produces documents; `JinaEmbedding` batches requests.
   - `PgVectorDatabase` writes in transactions with `ON CONFLICT` upserts.
   - Progress events emitted to clients; usage estimates updated.
3. **Search**
   - Query text embedded with `retrieval.query`.
   - SQL: `SELECT ..., embedding <=> $1 AS distance FROM code_chunks ... ORDER BY distance ASC LIMIT k`.
   - Optional lexical boost: `distance * dense_weight + (1 - ts_rank/ts_rank_max) * lexical_weight`.
4. **Fallback / Provider Switch**
   - Structured errors instruct clients to pause indexing or switch provider.
   - Provider toggle updates config; search/indexing recompute embeddings only if changing dimension.

## Gates & Dependencies

- **Simplicity Gate**: ≤3 new modules introduced (`JinaEmbedding`, `PgVectorDatabase`, migration runner). _Status_: Pass; no extra frameworks.
- **Anti-Speculation Gate**: No migration wizard or advanced cost alerts in architecture scope; documented as future work. _Status_: Pass.
- **Security Gate**: Credential storage for browser clients requires relay path or secret storage per platform; pending security review (owner: Security, due: before Phase 1 plan).
- **Performance Gate**: Need benchmark harness proving search latency within SC-3 tolerance and indexing throughput acceptable on Free + Pro tiers (dependency on Research item #2).

## Risks & Mitigations

| Risk ID | Description | Impact | Mitigation | Owner | Decision By |
|---------|-------------|--------|------------|-------|-------------|
| R1 | Supabase Free tier throttling stalls indexing mid-process. | Medium | Detect limits upfront, throttle concurrency, recommend Pro tier in UI. | Platform Eng | Beta exit |
| R2 | Jina model dimension change requires migration. | High | Detect dimension at runtime, version tables, supply migration script for column type change. | Core Eng | Before GA |
| R3 | Browser clients cannot store service keys due to enterprise policy. | High | Provide relay service pattern + signed JWT flow; allow read-only mode. | Security + Chrome team | Before Phase 2 |
| R4 | Hybrid search complexity delays launch. | Medium | Ship dense-only with clear messaging; track hybrid work in separate milestone. | Search Eng | Alpha exit |
| R5 | Migration from Milvus causes downtime or data loss. | Medium | Provide export/import scripts with dry-run and checksum validation; require backups before migration. | Platform Eng | GA-2 weeks |

## Open Decisions

1. Confirm whether we require `pg_trgm` for launch or treat lexical search as optional (ties to NC-1).
2. Decide minimal Supabase tier for “supported” status and document official requirements (NC-2).
3. Define telemetry/privacy posture for Supabase usage data (who can opt in, granularity).

## Dependencies & External Systems

- Supabase project limits (connections, row counts, extensions).
- Jina API availability, pricing tiers, and regional endpoints.
- Existing relay infrastructure to support write operations from restricted clients.
- Milvus → Supabase migration tooling (if shipping before GA).

## Ready for Planning?

- [ ] Recommended solution (OPT-1) reviewed with stakeholders; trade-offs documented.
- [ ] Research agenda items assigned with owners/dates.
- [ ] Gates (simplicity, security, performance) have clear pass criteria.
- [ ] Risks R1–R5 assigned owners and mitigation plans.
- [ ] Required interfaces (embedding API, storage schema, client configuration) identified for contracts/data models in planning phase.***
