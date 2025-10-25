# Architecture: Supabase pgvector & Jina Embedding Integration

## Summary

Scope establishes a need to deliver a Supabase-backed storage path with Jina embeddings that matches current Milvus/OpenAI behavior across CLI, MCP server, VS Code, and Chrome. This document focuses on a CLI-first rollout, documents alternatives, and identifies the research, dependencies, and gates required before planning.

## Solution Guardrails & Assumptions

- Maintain existing `VectorDatabase` and `Embedding` abstractions so clients do not require API changes and provider toggles stay transparent (supports scope A2).
- Customers manage their own Supabase projects; we handle schema provisioning, migrations, and validation of required extensions (`pgvector`, optionally `pg_trgm`).
- Shipping dense-only search is acceptable if hybrid parity is documented and scheduled (NC-1 default).
- Environment secrets flow through existing secure stores per surface (Node env vars, VS Code SecretStorage, Chrome storage + OS keychain). Provide a compliant relay or read-only path when service keys cannot live client-side (supports scope A3).
- Jina’s `jina-code-embeddings-1.5b` model (via `POST /v1/embeddings`) is the default; adapter must detect output dimensions and task types at runtime.

### Platform Tiers & Fallback (Evidence)

- Free-tier (Nano compute) provides shared CPU, ≈0.5 GB RAM, ~500 MB recommended database size, baseline ≈250 IOPS, and 60 direct connections; exceeding ~500 MB pushes projects into read-only mode.[^supabase-compute][^supabase-free-size]
- Data durability: Free tier lacks downloadable backups and PITR; operational guidance must enforce manual exports before storing critical data.[^supabase-backup]
- Extension availability: `pgvector` is managed via the Supabase `extensions` schema across tiers; migrations must verify presence and guide enablement steps.[^supabase-pgvector]
- Fallback: When limits or policy prevent Supabase adoption, recommend the Docker-based Postgres + pgvector deployment that reuses the same schema and migrations (planned in execution roadmap).

### Free-tier Capability Evaluation

- **Resource ceilings**: 60 direct connections (200 via pooler), shared CPU, and baseline ~250 IOPS require conservative pooling and sequential CLI-first indexing runs; parallel chunk workers must respect these caps.[^supabase-compute]
- **Storage window**: Projects enter read-only at ~500 MB of database storage; CLI must surface remaining budget and trigger local fallback workflows before writes fail.[^supabase-free-size]
- **Durability gap**: No downloadable backups or PITR on Free tier; operational docs must mandate manual exports before risky operations.[^supabase-backup]
- **Extension checks**: Provisioning flow verifies `pgvector` availability and supplies enablement SQL when absent; failure redirects users to local fallback.[^supabase-pgvector]
- **Support stance**: Paid Supabase plans are out of scope; if benchmarks or user limits fail, the product pivots to local Postgres/pgvector rather than recommending an upgrade.
- **Conclusion**: For repos that keep database size <500 MB and run sequential CLI-first indexing, Supabase Free tier is viable; larger or bursty workloads must switch to the local fallback.

## Options Considered

| Option ID | Description | Cost/Effort | Ops Overhead | Security | Performance | Maintainability | Notes |
|-----------|-------------|-------------|--------------|----------|-------------|-----------------|-------|
| OPT-1 | Direct SQL access to Supabase/Postgres via `pg` (Node) and Supabase JS REST client (browser). | **Engineering**: 8–10 weeks core adapter + embeddings + clients. | Medium – own migrations, monitoring. | High (full control over credential paths). | High (tunable SQL, close to data). | High – same code path works for self-hosted Postgres. | **Recommended** for portability and control. |
| OPT-2 | Wrap storage/search in Supabase Edge Functions or RPC endpoints. | 10–12 weeks (functions + deployment tooling). | High – users must deploy functions, monitor. | Medium – service role stored in Supabase. | Medium – extra latency, less tuning. | Medium – function code to version. | Rejected: higher friction for customers. |
| OPT-3 | Maintain Milvus and mirror into Supabase via sync bridge. | 6–8 weeks initial + ongoing sync ops. | Very High – dual storage, monitoring. | High – but doubles secrets. | Medium – read latency OK, writes slower. | Low – constant maintenance burden. | Rejected: undermines target objective. |

### Recommended Approach (OPT-1)

Direct SQL access aligns with goals of portability (self-hosted Postgres later), lower customer friction, and better control over performance. Requires disciplined SQL authoring and shared migrations, but avoids forcing customers to deploy serverless functions.

## Research Agenda (from Scope / New)

| RA ID | Question | Key Activities | Deliverable | Dependencies |
|-------|----------|----------------|-------------|--------------|
| RA1 | Can dense + lexical hybrid search meet relevance targets on Supabase? (NC-1) | Evaluate pgvector + pg_trgm viability, tunable weights, storage overhead. | Decision memo on launch scope + backlog item for hybrid follow-up. | Access to Supabase test projects. |
| RA2 | Can Supabase Free-tier sustain target workloads before fallback is required? (NC-2) | Measure Free-tier throughput, connection/extension limits, read-only triggers; define fallback thresholds. | Supportability checklist + signals that prompt local fallback messaging. | RA1 data; Free-tier benchmarking harness. |
| RA3 | How do we document the deferred telemetry scope? (NC-3) | Capture usage-display requirements, list deferred alerting scenarios, and flag follow-up owners. | Telemetry deferral memo feeding backlog. | Product + Support alignment. |
| RA4 | What latency hit do relays introduce for browser clients? (supports A3) | Prototype relay interactions, capture p95 under typical loads. | Report with thresholds + relay fallback guidance. | Platform relay infrastructure. |
| RA5 | When Supabase is not viable, how quickly can teams spin up local Postgres/pgvector? | Validate Docker-based reference deployment aligned with schema/migrations. | Deployment playbook feeding US2 fallback messaging + LP-T6 docs. | RA2 outputs. |

## Architecture Overview

### Components

1. **Embedding Service Layer (`JinaEmbedding`)**
   - Extends existing embedding abstraction and defaults to the `jina-code-embeddings-1.5b` model (`POST /v1/embeddings`).
   - Handles batching (32 items default), retry/backoff for 429/5xx, detects vector dimension during warm-up, and supports task flags (e.g., `nl2code.query`, `code2code.passage`).
   - Tracks token usage locally for UX messaging (no telemetry streaming in this release).

2. **Vector Storage Layer (`PgVectorDatabase`)**
   - Node contexts: `pg` pool with SSL, statement timeouts, connection retries.
   - Browser contexts: Supabase JS client for reads, relay for writes when anon key insufficient.
   - Schema per repository: `code_chunks_{hash}` with metadata JSONB, vector column, optional `tsvector`.
   - Supports upsert semantics, soft-delete, and incremental indexing markers.

3. **Configuration & Feature Flag Layer**
   - Extend config loaders to accept Supabase credentials (URL, service role, anon key) and embedding provider choice.
   - Feature flag toggles between Milvus/OpenAI and Supabase/Jina until migration complete.

4. **Health Checks & Observability**
   - CLI runs `SELECT 1`, verifies `pgvector` presence, checks row counts, and surfaces headroom guidance.
   - Embedding health ping hits Jina with minimal payload to confirm credentials.
   - No remote telemetry streaming in this release; progress is kept local to the CLI.

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
   - CLI tracks progress locally and surfaces remaining headroom guidance.
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
- **Performance Gate**: Need benchmark harness proving search latency within SC-3 tolerance and indexing throughput acceptable on the Supabase Free tier (dependency on RA2).

## Risks & Mitigations

| Risk ID | Description | Impact | Mitigation | Owner | Decision By |
|---------|-------------|--------|------------|-------|-------------|
| R1 | Supabase Free tier throttling stalls indexing mid-process. | Medium | Detect limits upfront, throttle concurrency, and guide users to the throttled mode or local fallback. | Platform Eng | Beta exit |
| R2 | Jina model dimension change requires migration. | High | Detect dimension at runtime, version tables, supply migration script for column type change. | Core Eng | Before GA |
| R3 | Browser clients cannot store service keys due to enterprise policy. | High | Provide relay service pattern + signed JWT flow; allow read-only mode. | Security + Chrome team | Before Phase 2 |
| R4 | Hybrid search complexity delays launch. | Medium | Ship dense-only with clear messaging; track hybrid work in separate milestone. | Search Eng | Alpha exit |
| R5 | Migration from Milvus causes downtime or data loss. | Medium | Provide export/import scripts with dry-run and checksum validation; require backups before migration. | Platform Eng | GA-2 weeks |

## Closed Decisions

- Hybrid search launches dense-only with pg_trgm follow-up (NC-1).
- Supabase Free tier is the only supported option; failures route to local Postgres fallback (NC-2).
- Telemetry remains usage-display only; proactive alerts deferred to future milestone (NC-3).

## Dependencies & External Systems

- Supabase project limits (connections, row counts, extensions).
- Jina API availability, pricing tiers, and regional endpoints.
- Existing relay infrastructure to support write operations from restricted clients.
- Milvus → Supabase migration tooling (if shipping before GA).

## Ready for Planning?

- [x] Recommended solution (OPT-1) reviewed with stakeholders; trade-offs documented.
- [x] Research agenda items assigned with owners/dates.
- [x] Gates (simplicity, security, performance) have clear pass criteria.
- [x] Risks R1–R5 assigned owners and mitigation plans.
- [x] Required interfaces (embedding API, storage schema, client configuration) identified for contracts/data models in planning phase.

[^supabase-compute]: Supabase Docs – “Compute and Disk” → _Compute Size_ and _Compute instance_ tables (`https://supabase.com/docs/guides/platform/compute-and-disk`), accessed 2025-03-09.
[^supabase-free-size]: Supabase Docs – “Understanding Database and Disk Size” → _Free Plan behavior_ (`https://supabase.com/docs/guides/platform/database-size#free-plan-behavior`), accessed 2025-03-09.
[^supabase-backup]: Supabase Docs – “Going into production” → _Database backup options and limitations_ (`https://supabase.com/docs/guides/platform/going-into-prod#database-backup-options-and-limitations`), accessed 2025-03-09.
[^supabase-pgvector]: Supabase Docs – “Using the pgvector extension” (`https://supabase.com/docs/guides/database/extensions/pgvector`), accessed 2025-03-09.

***
