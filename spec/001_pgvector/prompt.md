# **Universal Spec Docs Generation Prompt**

## **Preamble: How to Use This Template**

This guide provides a universal template for generating a three-part project specification. It is designed for you to follow its structure and principles precisely. All content within square brackets `[...]` are placeholders that must be replaced with your project's specific details. All content preceded by `e.g.,` is purely for illustration to clarify the expected format and should not be copied directly.

## **Guiding Principles: The "Living Document" Philosophy**

* **Start Strong**: These documents are for maximizing clarity and consensus *before* heavy implementation begins.
* **Embrace Change**: Plans inevitably change. Treat these documents as a baseline. Minor changes should live in task-tracking tools. Only major changes to scope or architecture warrant updating these documents.

---

# **`scope.md` — Phase 1: Intent & Value (Viewpoint: Project Manager)**

**Goal & Principles**: This document is the "contract" for the project's intent, defining the "Why" and the "What". **All content must revolve around user and business value**, setting clear boundaries and measurable success criteria for the project. **Strictly prohibit any technical implementation details here** to ensure requirements are pure and outcome-oriented.

1. **Project Overview**
    * *This section aims to summarize the project's core in the simplest terms. A reader should understand its purpose and value within 30 seconds.*
    * **Problem Statement**: `[A single, clear sentence describing the core problem to be solved, e.g., "Our customers lack a way to find relevant products based on their free-text search queries."]`
    * **Target User/Business Area**: `[Identify the primary beneficiary, e.g., "E-commerce Website Shoppers" or "Internal Marketing Department."]`

2. **User Stories & Acceptance Criteria**
    * *This is the core of the scope. Each user story must be the smallest unit of value that can be independently tested and delivered. Acceptance signals must be specific and quantifiable, serving as the sole standard for subsequent testing and validation.*

    | ID | Priority | User Story | Acceptance Signals (Measurable) |
    | :--- | :--- | :--- |:--- |
    | `US1` | P0 | As a `[User Role]`, I want to `[Perform an action]` so that I can `[Achieve a goal]`. | `e.g., 1. [A performance metric, e.g., 'P95 search latency is under 300ms']. 2. [A business metric, e.g., 'Click-through rate on search results increases by 5%'].` |

3. **Key Non-Functional Requirements (NFRs)**
    * *Define the "invisible" but critical quality attributes. These requirements are not tied to a specific function but determine the product's viability and must be prioritized from day one.*
    * **Performance**: `e.g., The system must support 200 requests per second (RPS) at peak.`
    * **Security**: `e.g., All user-identifiable data must be encrypted at rest using AES-265.`
    * **Reliability**: `e.g., The service must maintain a 99.9% uptime.`

4. **Constraints & Boundaries**
    * *Clearly define the "battlefield". Constraints are hard rules that must be followed, while "Out of Scope" items protect the delivery of core value by proactively deferring other work.*
    * **Constraints (Must-haves)**: `e.g., Must use the existing corporate identity provider for authentication.`
    * **Out of Scope (Won't do)**: `e.g., This project will not include a user-facing admin dashboard.`

5. **Assumptions & Clarifications**
    * *Expose the "unknowns" in the project. Assumptions are the foundation upon which the project is built; if proven false, the project may be at risk. Clarifications are specific questions that need to be answered by stakeholders to remove ambiguity.*
    * **Assumptions**: `A1`: `e.g., We assume the upstream data source has a reliable daily update schedule.`
    * **Needs Clarification**: `NC1`: `e.g., What is the data retention policy for user activity logs? (Default proposal: 90 days).`

6. **Phase Gate: Ready for Architecture?**
    * `[ ]` All P0 user stories are defined and have measurable acceptance signals.
    * `[ ]` Key NFRs (Performance, Security, etc.) are explicitly stated.
    * `[ ]` Open clarifications are under 3 and have default proposals.

---

# **`architecture.md` — Phase 2: Technical Strategy (Viewpoint: Architect)**

**Goal & Principles**: This document is the bridge between "requirements" and "implementation". It must **evaluate and determine a viable technical solution based on a full understanding of `scope.md`**. The focus is on making trade-offs across multiple dimensions (cost, risk, maintainability, etc.) and transparently documenting the decision-making process. **Strictly prohibit the introduction of new requirements or specific task lists here.**

1. **Preamble: Scope Confirmation**
    * *Begin by restating and confirming the goal you understand, ensuring the architectural design is perfectly aligned with the project scope.*
    * **Objective**: "This architecture is designed to implement user stories `US1`, `US2`, ... from `scope.md` and satisfy all its stated NFRs."

2. **Solution Evaluation**
    * *This is the core of architectural design. At least two viable options (including "do nothing" or "maintain status quo") must be objectively compared across multiple dimensions to make the decision process transparent.*

    | Option | Description | Pros | Cons | Verdict |
    | :--- | :--- | :--- |:--- |:--- |
    | **A** | `e.g., A serverless, event-driven approach.` | `e.g., Scales automatically, low idle cost.` | `e.g., Can suffer from cold starts, complex to debug.` | **Recommended** |

3. **Recommended Architecture (`DEC-A`)**
    * *Describe the chosen solution in detail. The description should be at the level of components and data flows, defining service boundaries, integration points, and data contracts, while avoiding code-level implementation details.*
    * **High-Level Diagram/Flow**: `[A text-based description of the main components and data flow, e.g., "User Request -> API Gateway -> Lambda Function -> DynamoDB."]`
    * **Technology Stack**:
        * **Primary Language**: `e.g., Go, Python`
        * **Primary Datastore**: `e.g., PostgreSQL`
    * **Design Rationale & Open Questions**: `[A space for non-structured thinking, e.g., "Our design prioritizes scalability over initial development speed. An open question remains on the long-term cost implications of this approach."]`

4. **Risks & Mitigation**
    * *Proactively identify potential risks in the technical solution and provide concrete mitigation plans. This turns unknown risks into manageable tasks.*
    * **Technical Risks**: `R1`: `e.g., The chosen database may not meet the low-latency NFR under heavy write load.` **Mitigation**: `e.g., Conduct a proof-of-concept (PoC) benchmark in Phase 0 of the plan.`

5. **Phase Gate: Ready for Planning?**
    * `[ ]` The recommended architecture has been approved by technical leadership.
    * `[ ]` The approach clearly addresses how all key NFRs will be met.
    * `[ ]` Major risks are identified, with mitigation strategies assigned.

---

# **`plan.md` — Phase 3: Execution Roadmap (Viewpoint: Developer)**

**Goal & Principles**: This document translates the approved architecture into a **concrete, ordered, and executable checklist of tasks**. It must be highly actionable. **All tasks must trace back to a decision in `architecture.md` or a requirement in `scope.md`**. **Strictly prohibit the introduction of any new requirements or architectural decisions here.**

1. **Preamble: Plan Overview**
    * *Clarify the basis for this plan and its rules of execution.*
    * **Architecture Reference**: "This plan executes the 'Recommended Architecture' (`DEC-A`) defined in `architecture.md`."
    * **Execution Logic**: "Tasks are listed in their recommended execution order. A task implicitly depends on the completion of the one before it. A block of consecutive tasks marked with `[Parallel]` can be worked on concurrently."

2. **Phased Task List**
    * *Break the entire project into logical phases, each with a clear checkpoint to validate its outcome. Each task should be a small unit of work with a clear deliverable.*

    **Phase 0: Setup & Research**
    * *Checkpoint: All necessary tools, environments, and knowledge are in place.*
    * [ ] **T0.1 (P0)**: Set up the source code repository and CI/CD pipeline foundation. **[Deliverable: `e.g., A README.md and a basic passing build script.`]**

    **Phase 1: Core Foundation**
    * *Checkpoint: The application's skeleton is running and can connect to its core dependencies.*
    * [ ] **T1.1 (P0)**: Provision core infrastructure using `[e.g., Terraform]`.

3. **Final Launch Readiness Checklist**
    * *A final checklist to ensure everything is ready after all development work is complete.*
    * `[ ]` All P0 tasks are complete.
    * `[ ]` Acceptance signals for all `scope.md` stories are met in a production-like environment.
    * `[ ]` All NFRs (Performance, Security) have been validated through testing.
    * `[ ]` Monitoring dashboards and alerts are configured.
