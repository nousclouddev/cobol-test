# Modernization Program Estimate (Effort, Cost, Risks)

## Assumptions and Basis
- Scope and sequencing follow the defined phased roadmap (Preparation → Coexistence → Migration → Optimization). 
- Complexity is driven by strongly coupled CICS XCTL flows (`SRCHFLY`/`SRCHTKT`/`SELLCOB1`), mixed persistence and file-based credentialing (`PASSDOC`), and unresolved dependencies (e.g., `SELLCOB2` target). 
- Program covers seven capability streams: Identity, Flight Discovery, Ticket Inquiry, Sales, Fulfillment/Print, Ingestion, and Data Platform.
- Estimates are for enterprise-grade delivery with security, observability, DevSecOps, and controlled cutover.

---

## Effort Estimation (T-shirt + Indicative Person-Months)

| Workstream | Scope Summary | Complexity | T-Shirt | Indicative Effort (Person-Months) |
|---|---|---:|---:|---:|
| Foundation Platform | Kubernetes, API gateway, CI/CD, observability, event backbone, secrets | High | L | 24–32 |
| Identity & Access | Replace `LOGIN`/`CRYPTVE` patterns, IdP integration, RBAC, lockout/MFA, audit | Very High | XL | 20–28 |
| Flight Discovery | Extract read-heavy search from legacy flow, reconciliation and fallback | Medium | M | 10–14 |
| Ticket Inquiry | Complex joins/pagination behavior and read-model parity validation | High | L | 16–22 |
| Sales Transactions | Replace write-path from `SELLCOB1` mesh, saga/idempotency, rollback paths | Very High | XL | 24–34 |
| Fulfillment & Print | Decouple spool-driven print path to document service + job tracking | Medium-High | M/L | 10–16 |
| Ingestion Modernization | JSON/XML/AS400 ingestion normalization + validation + retries | Medium | M | 8–12 |
| Data Migration & Reconciliation | CDC, backfill, canonical IDs, parity dashboard, cutover controls | Very High | XL | 22–30 |
| QA, NFR, Security Validation | Contract/E2E/perf/security testing, resiliency drills, hardening | High | L | 18–24 |
| PMO, Architecture, Change Management | Governance, architecture runway, business readiness, training | Medium-High | M/L | 14–20 |

**Program total:** **166–232 person-months** (central estimate ~198 PM).

Sizing interpretation:
- **M**: deliverable likely feasible in one major release train with moderate dependencies.
- **L**: multi-team dependency and cross-cutting controls required.
- **XL**: mission-critical cutover risk, high coupling, and mandatory phased fallback.

---

## Timeline (Management Baseline)

### Recommended plan: 15–20 months (phased, overlapping)

1. **Phase 1 – Preparation (3–4 months)**
   - Governance, platform baseline, IAM/security standards, data-mapping, test factory.
2. **Phase 2 – Coexistence (4–5 months)**
   - Strangler routing, read-domain extraction (Flight/Ticket), identity pilot, dual-run reconciliation.
3. **Phase 3 – Migration (5–7 months)**
   - Sales write migration, fulfillment migration, broader identity cutover, controlled retirement of legacy paths.
4. **Phase 4 – Optimization & Decommission (3–4 months)**
   - Performance tuning, cost optimization, legacy decommission, final operating model transition.

### Critical milestone view
- **M3/M4:** Platform + controls ready for production.
- **M7/M9:** Read domains at 60–80% traffic with SLO stability.
- **M12/M15:** Sales write-path cutover complete with rollback confidence.
- **M15/M20:** Legacy decommission and BAU handoff complete.

### Schedule scenarios
- **Aggressive (12–14 months):** requires high SME availability, low scope volatility, and minimal legacy surprises.
- **Base case (15–20 months):** recommended planning baseline.
- **Conservative (21–24 months):** if major data-quality debt or unresolved dependencies materialize.

---

## Team Model (Target Operating Structure)

### Core delivery model (peak ~28–36 FTE)
- **Program & Governance (4–5 FTE)**
  - Program Director, PMO lead, Finance/Commercial analyst, Change manager, Risk manager.
- **Architecture & Platform (5–6 FTE)**
  - Chief/Lead Architect, Platform architect, Security architect, Data architect, SRE lead.
- **Domain Squads (14–18 FTE)**
  - 4 squads (Identity, Read Services, Sales, Fulfillment/Ingestion), each 3–4 engineers incl. QA automation.
- **Data Migration Cell (4–5 FTE)**
  - Data engineers + reconciliation analyst + DBA support.
- **Testing & Release Assurance (3–4 FTE)**
  - Perf/security/contract test specialists.

### Leadership cadence
- Weekly steering committee (business + technology).
- Daily migration control-tower standup.
- Biweekly architecture/risk gate for go/no-go decisions.

### Sourcing recommendation
- **40–50% internal** (domain ownership continuity).
- **50–60% partner support** (accelerate cloud-native, DevSecOps, data migration specialization).

---

## Risk Register

| ID | Risk Type | Risk Description | Probability | Impact | Rating | Mitigation | Contingency Plan |
|---|---|---|---|---|---|---|---|
| T1 | Technical | Identity migration from `LOGIN`/`CRYPTVE` to IdP breaks auth edge cases | Medium | Very High | High | Pilot cohorts, dual-path auth, contract tests, lockout and audit controls | Immediate feature-flag rollback to legacy login path |
| T2 | Technical | Sales write-path refactor from tightly coupled XCTL mesh causes transaction defects | High | Very High | Very High | Saga orchestration, idempotency keys, exhaustive E2E regression, shadow writes | Freeze cutover; reroute writes to legacy while defects are remediated |
| T3 | Technical | Data parity drift between legacy DB2 and new read models | High | High | High | CDC quality checks, reconciliation dashboard, drift alert SLAs | Pause domain rollout; run replay/backfill and correct mappings |
| T4 | Technical | Unresolved dependency (`SELLCOB2` and naming drifts) blocks migration sequence | Medium | High | High | Dependency closure sprint before migration wave, architectural decision log | Re-sequence wave plan; isolate blocked capability via adapter |
| B1 | Business | SMEs unavailable, resulting in incomplete business rule capture | High | High | High | SME ringfencing, structured decision workshops, sign-off checkpoints | Use decision council for expedited policy calls; defer non-critical variants |
| B2 | Business | Scope creep from adjacent modernization demands inflates timeline/cost | High | Medium | High | Strict change-control board, MVP-by-domain scope discipline | Activate backlog triage; move enhancements post-cutover |
| B3 | Business | User adoption friction (new auth/process changes) impacts operations | Medium | Medium | Medium | Change communications, role-based training, phased rollout by cohort | Hypercare command center and temporary dual-process support |
| O1 | Operational | Insufficient L1/L2 readiness for coexistence incidents | Medium | High | High | Runbooks, on-call drills, incident game-days, SLO dashboards | Dedicated war room during release windows |
| O2 | Operational | Release pipeline instability slows deployment and hotfix turnaround | Medium | High | High | Harden CI/CD gates, artifact signing, rollback automation | Introduce release freeze and controlled hotfix lane |
| O3 | Operational | Vendor/tooling delays (platform, SIEM, gateway) create critical-path slips | Medium | Medium | Medium | Early procurement, parallel PoCs, fallback tooling shortlist | Use interim open-source stack where feasible |

---

## Cost Drivers (Order-of-Magnitude)

### Primary cost categories
1. **People costs (largest driver, ~65–75%)**
   - Multi-squad engineering, architecture, QA, data migration, PMO.
2. **Platform & tooling (~10–15%)**
   - Cloud runtime, observability, CI/CD, security tooling, event platform.
3. **Migration execution overhead (~8–12%)**
   - Dual-run operations, reconciliation, extended support windows.
4. **Change/adoption/compliance (~5–10%)**
   - Training, documentation, audits, controls evidence.

### Indicative budget range (program total)
- **Base-case estimate:** **USD 8.5M – 12.5M** over 15–20 months.
- **Aggressive lower bound:** ~USD 7.0M (highly constrained scope + high internal capacity).
- **Risk-adjusted upper bound:** ~USD 14.0M (extended coexistence, high defect remediation, data-quality issues).

### Cost sensitivity triggers
- Duration of dual-run coexistence.
- Data quality/remediation effort discovered after CDC start.
- Identity and sales cutover defect rates.
- SME availability and governance decision latency.

---

## Confidence Level

- **Overall estimate confidence:** **Medium (≈65%)**.
- **Why not high:** unresolved legacy dependencies, hidden business rules in procedural code, and expected data quality variance increase uncertainty.
- **Conditions to raise confidence to High (≥80%):**
  1. Complete dependency closure for unresolved modules/interfaces.
  2. Execute 8–10 week discovery spike with representative pilot (Identity + one read service).
  3. Baseline reconciliation error rates from first CDC stream.
  4. Validate NFR targets under production-like load in coexistence.

---

## Executive Recommendation
Proceed with the **base-case 15–20 month plan**, funding in **stage gates** tied to measurable outcomes (platform readiness, read-domain parity, sales cutover stability, decommission readiness). Maintain **strict scope control**, treat **identity and sales** as top-risk tracks, and preserve **instant rollback** capability until each domain meets SLO and parity thresholds for two consecutive release cycles.
