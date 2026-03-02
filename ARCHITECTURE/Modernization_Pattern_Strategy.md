# Modernization Pattern Strategy for Legacy Airline Sales Platform

## Pattern Evaluation Matrix

| Pattern | Applicability | Benefits | Risks | Tooling Required |
|---|---|---|---|---|
| **Rehost (lift-and-shift)** | **Medium** for quickly stabilizing CICS/COBOL runtime and DB2 dependencies without immediate code change. Most applicable as a short-term infrastructure move for non-differentiating operational components. | Fastest path to infra currency, lower datacenter risk, minimal functional disruption, creates runway for deeper modernization. | Preserves tight coupling (XCTL mesh, COMMAREA contracts), does not remove credential-file risk, can create "legacy in cloud" cost profile. | Mainframe emulation or managed z/OS hosting model, automated deployment pipelines, runtime observability, configuration management. |
| **Replatform** | **High** for externalizing platform concerns (IAM, monitoring, batch scheduling, print queue orchestration, API gateway) while keeping core business logic in COBOL initially. | Improves security/compliance posture, reduces ops burden, enables gradual API-first exposure, improves reliability/SLO management. | Interface drift risk between legacy data contracts and new platform services; partial modernization complexity across hybrid stack. | API gateway, IAM platform (OIDC/SAML), message broker, container/job orchestration, centralized logging/tracing, schema registry/data mapping tools. |
| **Refactor** | **High** for hotspots identified in discovery: SRCHTKT concern mixing, sales navigation mesh, shared SQL extraction opportunities. | Reduces technical debt, improves maintainability/testability, makes service extraction safer and cheaper. | Regression risk in transaction-heavy flows; requires deep domain and COBOL expertise; timeline longer than rehost/replatform. | Static/dependency analyzers for COBOL, automated test harnesses, contract tests, code quality tooling, CI/CD with staged rollout and rollback. |
| **Replace** | **Selective / Medium** for commodity capabilities (legacy crypto/password file validation, some print/queue integrations, possibly workforce ingestion adapters). **Low** for core sales workflow in first wave. | Accelerates retirement of high-risk legacy utilities, leverages SaaS/COTS maturity, lowers bespoke maintenance. | Business-rule loss risk if replacing core domains too early; integration overhead; vendor lock-in and migration dependency. | IAM SaaS/IDP, managed document/print service, ETL/iPaaS connectors, migration tooling, robust data reconciliation and dual-run capability. |
| **Strangler Fig** | **Very High** as the primary transformation pattern given clear bounded contexts and candidate service boundaries (Identity, Flight Discovery, Ticket Inquiry, Sales, Ingestion, Document Fulfillment). | Enables incremental cutover, reduces big-bang risk, supports value-first releases, preserves business continuity while shrinking legacy footprint over time. | Requires disciplined routing/governance and anti-corruption layers; dual-write/read complexity; prolonged hybrid operations if poorly governed. | Strangler routing layer/API facade, anti-corruption adapters, feature flags, traffic shaping, service mesh/API management, event and audit observability. |
| **Event-driven modernization** | **High** for decoupling workforce-to-identity updates, booking-to-ticket/print flows, and read-model propagation across contexts. | Loose coupling, better scalability, asynchronous resilience, supports domain event contracts and near-real-time integration. | Event ordering/idempotency challenges, eventual consistency impacts UX/processes, increased operational complexity. | Event broker/stream platform, schema contracts, outbox pattern implementation, idempotency framework, DLQ/retry handling, event monitoring. |
| **GenAI-assisted refactoring** | **Medium-High** as an accelerator for code comprehension, DCLGEN/SQL mapping analysis, unit-test scaffolding, and documentation modernization. | Faster analysis and transformation throughput, helps scarce COBOL skills, improves documentation and test artifact generation speed. | Hallucination/semantic drift risk; security/governance concerns for sensitive logic; cannot be unsupervised for transactional code changes. | Enterprise-controlled GenAI platform, code privacy controls, prompt/runbook governance, human-in-the-loop review workflow, semantic diff and test gates. |

## Recommended Pattern Strategy

Adopt a **hybrid strategy** led by **Strangler Fig + Replatform + targeted Refactor**, with **selective Replace** and **event-driven integration** as the backbone:

1. **Foundation phase:** Rehost only where needed for risk/capacity stabilization, then prioritize replatform of cross-cutting concerns (IAM, observability, integration fabric).
2. **Decomposition phase:** Use Strangler routing to carve out read-heavy and bounded contexts first (Flight Discovery, Ticket Inquiry), then Identity.
3. **Transaction phase:** Refactor and extract Sales orchestration and Ticketing/Fulfillment with event-driven contracts and anti-corruption adapters.
4. **Retirement phase:** Replace commodity legacy utilities (file-based crypto auth, brittle print patterns, fragmented ingestion contracts), then decommission residual COBOL modules.
5. **Acceleration layer:** Use GenAI-assisted refactoring under strict governance to increase throughput for analysis, test generation, and repetitive code transformation.

## Rationale

- The system shows **strong runtime coupling** in navigation and auth paths, making big-bang replacement high risk.
- Existing analysis already identifies **natural microservice boundaries**, making Strangler the most practical control pattern.
- Core revenue flows (search/sell/ticket) require **business continuity**, favoring phased extraction over direct replacement.
- High-risk technical debt (credential file dependency, mixed SRCHTKT concerns, unresolved target drift) is better handled by replatform + refactor sequencing.
- Event-driven interfaces align with the target bounded-context model and reduce future point-to-point coupling.

## Risks and Constraints

1. **Critical path fragility:** LOGIN and Sales mesh outages can stop operations; modernization must preserve transaction integrity.
2. **Data contract inconsistency:** Naming drift (`EMPLO` vs `EMPLOYEE`, target mismatches like `SRCHFLI/SRCHFLY`) can break interfaces during extraction.
3. **Dual-run complexity:** Strangler and event-driven rollout implies temporary duplicate logic/data paths requiring strong reconciliation.
4. **Skill and governance constraints:** COBOL + distributed architecture + eventing + AI governance skills must be staffed in parallel.
5. **Operational hybrid burden:** Legacy and modern platforms will coexist for multiple releases; SRE/ops model must be explicit.
6. **Compliance/security sensitivity:** Identity and employee/passenger data flows require strict controls before introducing AI-assisted tooling.

## Execution Order

1. **Stabilize & baseline**
   - Resolve dependency drift and missing targets; establish observability and regression baselines.
2. **Replatform cross-cutting controls**
   - Externalize IAM, logging/monitoring, API facade, and integration backbone.
3. **Strangler Wave 1 (read services)**
   - Extract Flight Discovery and Ticket Inquiry as API services with legacy adapters.
4. **Event backbone introduction**
   - Publish/consume domain events for workforce updates and booking/ticket/print lifecycles.
5. **Strangler Wave 2 (identity + ingestion)**
   - Carve out Identity service and unify HR/passenger ingestion contracts.
6. **Refactor + Strangler Wave 3 (write core)**
   - Decompose Sales orchestration and Ticketing fulfillment with transactional safeguards.
7. **Selective replace + decommission**
   - Replace commodity utilities, retire legacy modules by context, and collapse remaining bridge layers.
8. **Continuous optimization**
   - Use GenAI-assisted refactoring/test generation with mandatory human approval and production-quality gates.
