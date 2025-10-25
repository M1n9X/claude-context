# Scope: Supabase pgvector Storage & Jina Coding Embeddings

## Project Overview

Claude Context currently relies on Milvus/Zilliz hosting and OpenAI embeddings. Teams operating in Postgres-first environments or seeking predictable pricing face heavy lift to adopt the product. This feature delivers a first-class Supabase (Postgres + pgvector) deployment path and adds Jina’s code-focused embeddings so customers achieve higher-quality retrieval without switching tooling stacks.

## Stakeholders & Triggers

- **Platform engineers (primary)** — provision Supabase projects and guarantee search performance for production repositories.
- **Individual developers (primary)** — trial the feature on Supabase Free tier and need immediate feedback on quotas and index progress.
- **Security administrators (secondary)** — audit key storage, enforce rotation, and ensure desktop/extension clients align with policy.
- **AI engineers / search owners (secondary)** — compare retrieval quality between Milvus/OpenAI and Supabase/Jina and decide on roll forward/back.
- **Support & enablement (supporting)** — document onboarding, incident recovery, and cost guidance for end users.

## Problem Statement & Objectives

- **Problem**: Postgres-centric teams cannot adopt Claude Context without maintaining parallel infrastructure, and current embeddings lag on code relevance.
- **Objectives**:
  - Deliver a “Supabase mode” that works across CLI, MCP server, VS Code, and Chrome with production-ready ergonomics (supports US1, US3).
  - Provide turnkey adoption of Jina coding embeddings with seamless fallback to existing providers (supports US4).
  - Make onboarding self-serve with immediate validation, clear failure guidance, and visibility into resource consumption (supports US1, US2).
  - Preserve or improve responsiveness and UX relative to the Milvus/OpenAI combination (supports SC-2, SC-3).

## Scenario Coverage

- **Greenfield onboarding (Pro tier)** — Expect one-click validation, automated schema creation, production-hardening guidance for RLS and key rotation.
- **Hobby usage (Free tier)** — System highlights extension/row limits upfront and offers graceful degradation (dense-only search, throttled indexing) instead of failing silently.
- **Enterprise locked-down clients** — Workflows document read-only credentials and optional relay services for writes in desktop/web extensions.
- **Migration from Milvus** — Users can import historical vectors or rebuild indexes after flipping Supabase mode; UX must surface the choice even if tooling ships later.
- **Incident recovery** — If Supabase or Jina is unreachable, product guides user to pause indexing, retry, or switch providers without losing state.

## User Stories & Independent Tests

| ID  | Priority | Story                                                                                     | Independent Test                                                                                      | Acceptance Signals |
|-----|----------|-------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|--------------------|
| US1 | P1       | Platform engineer connects Supabase project, validates setup, and indexes production repo without SQL chores. | Provision Supabase Pro instance, run guided setup flow, execute sample repo indexing to completion.   | Validation checks pass, tables/indices auto-created, first search returns relevant results in <2 minutes. |
| US2 | P1       | Individual developer on Supabase Free tier measures indexed docs and Jina quota impact.   | Point feature at free-tier project, index demo repository, review usage dashboard.                   | Indexing completes within Free-tier caps, usage panel shows row count and estimated Jina tokens. |
| US3 | P1       | Security administrator audits secret storage, rotates keys, and confirms access logs.     | Rotate Supabase service role and Jina keys across CLI/VS Code/Chrome clients.                         | Rotation causes no downtime, audit log export captures access history, read-only mode works. |
| US4 | P2       | AI engineer compares Jina-based search to previous backend and can roll back if relevance dips. | Run benchmark harness on both providers for same repo, toggle provider, observe quality metrics.      | Provider switch requires no restart, benchmark deltas within tolerance, rollback completes in <5 minutes. |

## Constraints, Edge Cases & Guardrails

- Supabase Free tier may block extensions and limit row counts; UI must surface these limits and provide fallback modes (dense-only search, throttled indexing).
- Offline or air-gapped usage should degrade gracefully to local keyword search without corrupting indexed data.
- Browser-based clients may be prohibited from storing service keys; solution must support read-only or relay-assisted write paths.
- Repositories ≥500k LOC require batching/throttling to complete indexing within Supabase quotas and without timeouts.
- Jina rate limits and outages must prompt actionable options (wait, reduce concurrency, switch provider) without data loss.

## Success Metrics

- **SC-1 (Adoption)**: ≥60% of new self-hosted trials choose Supabase on first session and complete indexing without support intervention within 24 hours.
- **SC-2 (Relevance)**: Jina embeddings deliver ≥5% improvement in search satisfaction (thumbs-up rate or CTR) for code-heavy queries versus baseline.
- **SC-3 (Performance)**: Search latency p95 remains within +20% of current Milvus/OpenAI stack for repositories ≤250k chunks.
- **SC-4 (Cost Transparency)**: ≥90% of onboarding sessions display Supabase storage usage and Jina token estimates before first repository finishes indexing.

## Assumptions & Clarifications

### Assumptions

- **A1**: Customers can enable pgvector (and optional pg_trgm) or accept guided instructions to do so. _Evidence_: Supabase documentation confirms availability on Free and Pro tiers as of 2024-04.
- **A2**: Clients must be able to toggle between vector/embedding providers without new UX flows or API changes. _Assumption_: Any additional adapter work can stay inside shared libraries without surfacing to end users.
- **A3**: Desktop and browser clients require a compliant path when service keys cannot be stored locally. _Assumption_: We can provide either a read-only fallback or a mediated write channel (e.g., relay) without exceeding latency targets.

### Needs Clarification (max 3)

- **NC-1**: Is hybrid (lexical + vector) search parity mandatory at launch, or can dense-only ship with a documented roadmap? _Default_: Ship dense-only with optional lexical blend marked beta.
- **NC-2**: What is the minimum Supabase plan we support for production workloads? _Default_: Recommend Pro tier for production, Free tier for evaluation with guardrails.
- **NC-3**: Do in-product cost alerts need proactive notifications, or is surfacing usage data sufficient? _Default_: MVP provides usage display; proactive alerts deferred.

## Research Agenda

- Benchmark Supabase Free vs. Pro tiers for indexing throughput, extension availability, and limits (addresses NC-2).
- Validate feasibility, performance, and operational overhead of hybrid search on pgvector + pg_trgm (addresses NC-1).
- Investigate Supabase and Jina usage telemetry APIs to gauge effort for proactive cost alerts (addresses NC-3).

## Dependencies & External Factors

- pgvector extension availability, connection limits, and service role permissions within Supabase.
- Jina API rate limits, pricing changes, or model dimension updates.
- Milvus migration tooling requirements to import existing embeddings for users transitioning mid-stream.
- Security review and compliance requirements for credential storage across CLI, VS Code, and Chrome surfaces.

## Ready for Architecture?

- [ ] All P1 user stories (US1–US3) defined with independent acceptance signals.
- [ ] Success metrics (SC-1–SC-4) measurable without prescribing implementation details.
- [ ] ≤3 outstanding `Needs Clarification` items with proposed defaults.
- [ ] Research agenda enumerated for unresolved questions.
- [ ] Stakeholder objectives, constraints, and edge cases documented for downstream design.
