# Prompt Guide for Spec Documents

The three files in `spec/001_pgvector` are the handshake between intent, technical direction, and execution. Treat them as a staged pipeline—Scope → Architecture → Plan—where each phase validates the previous one, surfaces blockers early, and feeds the next artifact with traceable decisions.

## Shared Guardrails

- Write for stakeholders first, engineers second: keep prose plain-language, call out assumptions, and tag uncertainties with `Needs Clarification` (limit 3 at a time; propose defaults if more exist).
- Preserve traceability. Every success metric, decision, and task must map back to something in an earlier document. Use short IDs (`US1`, `FR-3`, `DEC-A`) so later phases can reference them.
- Carry over gates/checklists. End each document with a concise checklist stating what must be true before the next phase begins.
- Separate fact vs. hypothesis. Use `Evidence:` / `Assumption:` labels when citing data, and create a `Research Agenda` subsection if the next phase needs structured investigation.
- Avoid implementation detail leakage: no framework names in `scope.md`, no task lists in `architecture.md`, no fresh requirements in `plan.md`.

## `scope.md` — Phase 1: Intent & Outcomes

- Anchor on user/business value. Start with the problem statement, stakeholders, and situational triggers. Ensure each primary user story is independently deliverable and testable.
- Provide a prioritized user story table with columns for `Priority`, `Story`, `Independent Test`, and `Acceptance Signals`. Link stories to measurable outcomes and note if any rely on external data/contracts.
- Document constraints, edge cases, and failure handling in bullet form. Distinguish required guardrails vs. stretch resilience.
- Capture success metrics split into qualitative (experience) and quantitative (performance/cost) targets; tie them to the stories they validate.
- Maintain an `Assumptions & Clarifications` section: list up to three `Needs Clarification` items with proposed defaults; add an `Assumption` list that downstream teams must either confirm or replace with evidence.
- Close with a **Ready for Architecture?** checklist summarising prerequisites (e.g., “All P1 stories defined”, “Success metrics measurable”, “≤3 open clarifications”).

## `architecture.md` — Phase 2: Technical Strategy

- Start with a summary that restates the scope intent and highlights what success looks like in engineering terms.
- Evaluate multiple solution approaches, including “do nothing,” and score them across cost, engineering effort, operational overhead, security, performance, and maintainability. Explicitly note which `scope.md` stories or metrics each option supports or jeopardises.
- Record research findings and remaining unknowns. If further investigation is needed, spin up a `Research Agenda` list (feeds Phase 0 in planning) that references the `Needs Clarification` tags from scope.
- Describe the recommended architecture at component/data-flow depth: key services, integration paths, sequence diagrams, and where contracts or data models will be required. Map each component back to user stories or success metrics.
- Include “Gates & Dependencies” to confirm simplicity, avoidance of speculative architecture, and compliance with security/operations policies. Flag any violations with justification and mitigation.
- Detail risks, trade-offs, and fallback strategies. Use a table linking `Risk`, `Impact`, `Mitigation`, `Owner`, and `Decision By`.
- End with a **Ready for Planning?** checklist (e.g., “Chosen approach approved”, “Interfaces requiring contracts identified”, “Unresolved risks assigned with due dates”).

## `plan.md` — Phase 3: Execution Roadmap

- Organise work into a phased execution pattern: `Phase 0 – Setup/Research`, `Phase 1 – Foundations`, `Phase N – User Story <ID>` for each prioritized slice, followed by `Polish/Hardening`. Each phase should end with a checkpoint description that proves readiness to move on.
- Within each phase, list tasks as checkboxes annotated with `Priority (P0/P1/P2)`, `Dependency`, `Deliverable`, and `Acceptance`. Reference the IDs from `scope.md` and `architecture.md` so reviewers can see traceability.
- Identify parallelisable items by tagging them (`[P]`) and note ownership or team handoffs (e.g., Docs, Security, QA). Include explicit cross-team gates (approvals, sign-offs) and insert them before dependent work.
- Capture tooling, environments, and telemetry expectations. If a task creates a new doc/contract or requires test harnesses, state the exact path to be produced.
- Include a `Phase Gates` subsection that mirrors the readiness checklists defined earlier—list what proves each phase is done (tests passing, documentation updated, benchmarks collected).
- Finish with a `Launch Readiness` checklist aggregating all must-pass gates (security review, regression benchmarks, support materials, feature flag decisions). Note any exit criteria for feature flags or beta programs.

Following these prompts keeps the three artifacts in lockstep, reinforces disciplined phased delivery, and makes downstream execution predictable.
