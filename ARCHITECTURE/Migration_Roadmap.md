# Phased Migration Roadmap: Legacy COBOL/CICS/DB2 to Target Domain Services

## Executive Delivery Stance
This roadmap is built for execution under a strangler pattern: keep legacy services stable, migrate domain capabilities in controlled waves, and enforce measurable go/no-go criteria at every phase. It aligns to the discovered legacy dependency mesh (LOGIN/CRYPTVE, SRCHFLY/SRCHTKT/SELLCOB1, batch ingestion, spool print) and the target architecture of API gateway, IAM, domain services, and event backbone.

---

## Phase 1 — Preparation

### Activities
1. **Program governance and migration factory setup**
   - Establish modernization PMO cadence (weekly steering, daily workstream sync).
   - Define domain workstreams: Identity, Flight Discovery, Ticket Inquiry, Sales, Fulfillment, Ingestion, Data.
   - Finalize risk register and dependency map baselining.

2. **Platform and cross-cutting foundation**
   - Stand up Kubernetes baseline, API gateway, observability stack, CI/CD with quality gates.
   - Establish event backbone and schema governance (versioning, compatibility rules).
   - Implement centralized secrets/key management and environment config standards.

3. **Security hardening before extraction**
   - Introduce enterprise IdP integration path and token standards.
   - Define lockout, rate-limit, MFA, and audit policies to replace legacy weak controls.
   - Remove sensitive logging patterns from migration adapters and new components.

4. **Legacy discovery-to-backlog conversion**
   - Convert legacy modules into migration epics/stories with acceptance tests.
   - Capture unresolved dependencies (e.g., SELLCOB2 target, naming drifts) as blocking items.
   - Baseline current performance and defect metrics for later comparison.

5. **Data migration readiness**
   - Profile legacy data quality (EMPLO, PASSENGERS, FLIGHT, TICKET, BUY, CREW, SHIFT).
   - Define canonical IDs and mapping tables for employee/passenger/flight/ticket/bookings.
   - Build reconciliation framework (record-level and aggregate-level controls).

6. **Testing strategy mobilization**
   - Stand up automated test pyramid: unit, integration, contract, E2E, non-functional.
   - Build golden test packs from critical paths: login, flight search, ticket inquiry, booking, print/receipt.
   - Define baseline non-functional SLO targets and security test gates.

### Deliverables
- Migration charter, phase plan, RACI, RAID log.
- Reference architecture runbook (platform, IAM, API, events, observability).
- Domain-level backlog with sequencing and estimates.
- Data migration playbook v1 (mapping, reconciliation, exception handling).
- Test strategy and regression suite baseline.

### Entry Criteria
- Target-state architecture and domain boundaries approved.
- Legacy inventory and dependency map validated with SMEs.

### Exit Criteria
- Foundation services operational in non-prod and production-ready by policy.
- Security, CI/CD, observability, and test automation baselines signed off.
- Data mapping/reconciliation rules approved by business and data owners.

### Rollback Strategy
- No business traffic migrated in this phase; rollback is configuration-only.
- Revert platform/IAM/event changes via IaC version rollback.
- Maintain legacy-only runtime as production default.

### Risk Controls
- Architecture review board gate before any domain extraction.
- Mandatory threat modeling and security controls verification.
- Dependency closure checklist for unresolved program/table/interface gaps.

### Success Metrics
- 100% critical domains mapped with approved backlog.
- ≥90% critical user journeys covered by automated tests.
- Zero Sev-1 incidents introduced by foundational changes.

---

## Phase 2 — Coexistence

### Activities
1. **Introduce anti-corruption layer (ACL) and strangler routing**
   - Deploy routing facade in front of CICS transactions.
   - Route read-only use cases first to new services while preserving legacy fallback.
   - Add feature flags for per-capability traffic control.

2. **Extract read-heavy domains first**
   - Deliver Flight Catalog Service and Ticket Inquiry Service with read replicas/projections.
   - Build adapters to legacy DB2 and legacy events/CDC feed where needed.
   - Keep SRCHFLY/SRCHTKT as fallback during confidence ramp-up.

3. **Identity/workforce coexistence controls**
   - Introduce Identity Service with IdP-backed auth for pilot cohorts.
   - Replicate workforce attributes from legacy employee sources into new model.
   - Keep legacy LOGIN path available behind controlled switch.

4. **Data synchronization and reconciliation**
   - Run dual-read validation for flights/tickets between legacy and new read models.
   - Establish drift alerts and reconciliation workflows.
   - Track and remediate mapping exceptions.

5. **Operational readiness and support model**
   - Define L1/L2/L3 runbooks and on-call ownership per domain.
   - Add SLO dashboards (latency, error rate, event lag, data drift, auth failures).
   - Conduct chaos/failover exercises for API gateway/event backbone.

### Deliverables
- Strangler router with granular feature flags.
- Flight Catalog and Ticket Inquiry services in production coexistence mode.
- Identity pilot release with controlled user groups.
- Reconciliation dashboards and exception queues.
- Coexistence runbooks and incident playbooks.

### Entry Criteria
- Preparation phase exit criteria achieved.
- ACL/routing architecture approved with rollback hooks.

### Exit Criteria
- Read flows serve target traffic threshold (e.g., 60–80%) with stable SLOs.
- Data parity for migrated read domains meets agreed tolerance.
- Pilot identity users successfully authenticated with no critical regressions.

### Rollback Strategy
- Instant feature-flag rollback to legacy SRCHFLY/SRCHTKT/LOGIN flows.
- Disable new read projections and route all queries back to legacy.
- Preserve legacy as source of truth; no destructive schema change allowed.

### Risk Controls
- Canary deployments with automatic rollback on SLO breach.
- Contract-test gates on all APIs/events before promotion.
- Daily parity review and defect triage war-room during ramp-up.

### Success Metrics
- ≥99.5% availability for coexistence read APIs.
- Data parity error rate below agreed threshold (e.g., <0.5% critical mismatches).
- Mean rollback time <15 minutes through routing controls.

---

## Phase 3 — Transformation

### Activities
1. **Migrate transactional core (Sales + Fulfillment)**
   - Implement Sales Service with saga/orchestration for booking lifecycle.
   - Externalize ticket/receipt generation into Fulfillment Service with retry semantics.
   - Replace direct XCTL orchestration with API/event contracts.

2. **Modernize ingestion and master data flows**
   - Introduce Ingestion Gateway for JSON/XML/AS400 feeds.
   - Standardize validation, schema enforcement, and idempotent processing.
   - Decouple credential generation from employee ingestion.

3. **Credential and auth modernization completion**
   - Decommission CRYPTVE/PASSDOC dependency through phased credential cutover.
   - Enforce centralized password/hash policy and adaptive authentication controls.
   - Implement full audit trails and security analytics pipeline.

4. **Data migration execution (write paths)**
   - Use incremental migration by bounded context (Identity/Workforce, Passenger, Sales).
   - Apply dual-write only where strictly necessary and shielded by idempotency keys.
   - Perform cutline-based historical migration + near-real-time sync for deltas.

5. **Testing intensification for write domains**
   - Full E2E booking/ticket/receipt journey testing under load.
   - Deterministic replay tests for event-driven workflows and sagas.
   - Reconciliation tests for financial/booking integrity and audit completeness.

### Deliverables
- Sales Service and Fulfillment Service production release in partial traffic mode.
- Ingestion Gateway replacing legacy batch entrypoints for scoped feeds.
- Credential modernization completion report and security control attestation.
- Data migration wave reports with reconciliation sign-offs.
- Updated SOPs and business continuity procedures.

### Entry Criteria
- Coexistence phase stable and read domains operating within SLO.
- Event contracts and saga patterns approved by architecture and operations.

### Exit Criteria
- Majority of transactional traffic handled by new services with no Sev-1 regressions.
- Legacy credential validation disabled for active user population.
- Financial and booking reconciliation passes at agreed confidence level.

### Rollback Strategy
- Per-domain rollback: redirect booking/finalization to legacy SELLCOB paths.
- Pause event consumers and replay from checkpoints after remediation.
- Keep legacy write path hot until three consecutive stabilization windows are passed.

### Risk Controls
- Transaction guardrails: idempotency keys, outbox pattern, dead-letter handling.
- Change freeze windows during high-demand business periods.
- Independent QA + business UAT signoff before each transactional traffic increase.

### Success Metrics
- ≥95% transactional traffic on new stack before cutover decision.
- Booking success rate and completion latency equal or better than baseline.
- Zero unreconciled critical booking/ticket records at wave closure.

---

## Phase 4 — Cutover

### Activities
1. **Final production cutover planning and execution**
   - Execute runbooked cutover weekend/window with command center governance.
   - Switch default routing to target services for all migrated capabilities.
   - Maintain shadow monitoring of legacy outputs for assurance period.

2. **Legacy decommission steps**
   - Disable legacy LOGIN/CRYPTVE auth path and password file operations.
   - Retire unused CICS transactions and spool dependencies progressively.
   - Archive legacy artifacts, logs, and operational docs for compliance.

3. **Post-cutover hypercare**
   - Hypercare war room for 2–6 weeks with rapid defect response SLAs.
   - Tight monitoring of customer experience, auth failures, booking completion, print fulfillment.
   - Continuous reconciliation until closure criteria met.

4. **Operational and organizational transition**
   - Transition ownership to BAU teams with updated support model.
   - Finalize training, SOP handoff, and incident drills.
   - Complete cost and performance optimization pass.

### Deliverables
- Executed cutover checklist and command-center log.
- Legacy decommission and archival evidence pack.
- Hypercare report with defect/resolution trend.
- Final migration closure report and KPI attainment summary.

### Entry Criteria
- Transformation phase exit criteria achieved with signed go-live recommendation.
- Business readiness and support readiness approved.

### Exit Criteria
- 100% targeted business capabilities served by target architecture.
- Legacy components decommissioned or formally retained with exception approval.
- Hypercare closure with stable SLO compliance.

### Rollback Strategy
- Time-boxed rollback window: revert routing to legacy where still technically retained.
- If full rollback not possible, execute partial rollback by domain and activate continuity playbooks.
- Preserve immutable backups/snapshots for all cutover data operations.

### Risk Controls
- Command center with explicit go/no-go gates.
- Real-time KPI and error-budget dashboard with automated alerting.
- Business continuity drills before and after cutover.

### Success Metrics
- 0 critical business outage during cutover window.
- <2% post-cutover defect leakage in first 30 days.
- Target SLO attainment for availability, latency, and booking completion.

---

## Cross-Phase Execution Components

### Testing Strategy (end-to-end)
- **Shift-left controls:** static analysis, code quality, IaC and dependency scans in CI.
- **Contract-first validation:** API/event schema tests are mandatory for every release.
- **Golden path regression:** login → search → booking → ticket inquiry → receipt.
- **Non-functional:** load, soak, failover, resilience, security penetration tests each phase.
- **Go-live confidence:** rehearsal cutovers in staging with production-like datasets.

### Data Migration Approach
- **Principles:** domain-by-domain migration, canonical identifiers, no big-bang for write-heavy domains.
- **Method:**
  1. Profile and cleanse.
  2. Map and transform to target canonical models.
  3. Backfill historical data by cutline.
  4. Sync deltas through CDC/events/adapters.
  5. Reconcile and sign off.
- **Controls:** checksums, record counts, business rule reconciliation, exception queues, replay capability.

### Change Management
- Stakeholder map across operations, customer service, sales, IT, security, compliance.
- Role-based training by phase with practical runbooks and simulation exercises.
- Communication cadence: release notices, pilot updates, cutover bulletins, hypercare reporting.
- Adoption KPIs: user onboarding rates, support ticket trend, training completion, process conformance.

---

## Consolidated Risk Controls
- Feature-flag and canary-first releases for every migrated capability.
- Dual-run and reconciliation before any authoritative switch.
- No irreversible legacy decommission until stabilization gates pass.
- Security and compliance gates embedded in every phase exit.
- Executive go/no-go board with measurable acceptance thresholds.

## Consolidated Rollback Plan
1. **Immediate rollback (minutes):** routing/feature-flag reversal.
2. **Operational rollback (hours):** disable new consumers/workflows and restore legacy path ownership.
3. **Data rollback (hours-days):** snapshot restore + replay from durable event logs.
4. **Business continuity fallback:** predefined manual/legacy operation modes for critical functions.

## Program-Level Success Metrics (North-Star)
- Business continuity: zero unplanned critical outage from migration activities.
- Service quality: target availability/latency SLOs met for migrated domains.
- Data integrity: reconciliation pass rate >99.5% on critical entities.
- Security uplift: retirement of weak crypto/file-based credentials and adoption of centralized IAM.
- Delivery predictability: phase milestones achieved within agreed variance and risk tolerance.
