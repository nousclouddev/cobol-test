# Target-State Architecture for Modernized Airline Sales Platform

## Target Architecture Overview

The target state uses a **strangler modernization** model: legacy CICS/COBOL remains operational during transition, while new domain services are introduced behind an API facade and event backbone. The architecture is organized by bounded contexts already identified in discovery: Identity, Sales, Flight Discovery, Ticket Inquiry, Passenger, Workforce, Fulfillment, and Ingestion.

### Architectural style
- **Hybrid cloud-native + legacy coexistence** during migration waves.
- **Domain-aligned microservices** for independently scalable business capabilities.
- **API-first synchronous integration** for channel requests.
- **Event-driven asynchronous integration** for decoupled workflows and read-model propagation.
- **Zero-trust security posture** with centralized identity, policy, and audit.

### Transformation approach
1. **Stabilize and replatform cross-cutting concerns** (IAM, API gateway, observability, CI/CD).
2. **Extract read-heavy domains first** (Flight Discovery, Ticket Inquiry).
3. **Migrate transactional orchestration** (Sales + Ticket/Fulfillment) with idempotent event contracts.
4. **Retire legacy authentication and file-based crypto** by moving to managed IAM.
5. **Decommission residual COBOL components** after dual-run and reconciliation closure.

---

## Component Diagram (textual)

```text
[Users/Agents]
    |
    v
[Web/Mobile/Branch Channel Apps]
    |
    v
[API Gateway + WAF + Rate Limiter]
    |
    +--> [Identity Service] <--> [IAM/IdP (OIDC/SAML, MFA, RBAC/ABAC)]
    |
    +--> [Flight Catalog Service] ------+
    |                                   |
    +--> [Ticket Inquiry Service] ------+--> [Query Cache / Search Index]
    |                                   |
    +--> [Passenger Service] -----------+
    |
    +--> [Sales Service] ---> [Saga/Workflow Orchestrator]
    |                              |
    |                              +--> [Inventory/Availability Adapter]
    |                              +--> [Payment Adapter (if applicable)]
    |                              +--> [Fulfillment Service]
    |
    +--> [Workforce Service] <--- [Ingestion Gateway (JSON/XML/AS400)]

[Event Bus / Streaming Platform]
    ^        ^         ^         ^
    |        |         |         |
[Sales] [Ticket] [Workforce] [Fulfillment] publish/consume domain events

[Data Layer]
 - Identity DB
 - Workforce DB
 - Passenger DB
 - Sales OLTP DB
 - Ticket Read Model DB
 - Flight Catalog Read DB
 - Fulfillment Job DB
 - Object Storage (documents, archives)
 - Data Lake/Warehouse (audit, analytics)

[Legacy Zone]
[CICS/COBOL/DB2 modules] <--> [Anti-Corruption Layer + Strangler Router]
```

---

## Technology Stack

### Platform & runtime
- **Container platform:** Kubernetes (managed offering).
- **Service runtime:** Java/Spring Boot or .NET for core services; lightweight Node/Python for adapters where suitable.
- **Service mesh:** Istio/Linkerd for mTLS, traffic policy, and progressive delivery.

### API & integration
- **API management:** Kong / Apigee / Azure API Management.
- **Event backbone:** Kafka (or cloud-native equivalent such as Event Hubs/Pub/Sub).
- **Schema contracts:** Avro/JSON Schema + schema registry.
- **Workflow orchestration:** Temporal / Camunda / Conductor for long-running sales sagas.

### Data
- **Operational stores:** PostgreSQL/Aurora for new microservices.
- **Caching:** Redis.
- **Search/read optimization:** OpenSearch/Elasticsearch for ticket inquiry and full-text queries.
- **Object storage:** S3/Blob for generated receipt/ticket artifacts.
- **Analytics:** Lakehouse/Warehouse (Snowflake/BigQuery/Redshift).

### Security architecture
- **Identity provider:** Enterprise IdP (OIDC/SAML) integrated with workforce lifecycle.
- **AuthN/AuthZ:** OAuth2.1 + JWT/opaque tokens; RBAC + policy-based controls (OPA).
- **Secrets & keys:** Vault/KMS/HSM-backed key management.
- **Data protection:** TLS 1.2+, mTLS east-west, encryption at rest, tokenization for sensitive fields.
- **App security controls:** WAF, bot protection, rate limits, account lockout, adaptive MFA.
- **Security operations:** SIEM integration, centralized audit trails, anomaly detection.

### Observability
- **OpenTelemetry** instrumentation across APIs, events, and workflows.
- **Metrics:** Prometheus + Grafana dashboards (SLO/SLI based).
- **Logs:** Structured logs to ELK/OpenSearch/Splunk.
- **Tracing:** Distributed tracing (Jaeger/Tempo/X-Ray).
- **Reliability automation:** error budget alerts, synthetic probes, event lag monitors, DLQ alarms.

### CI/CD
- **Source & pipeline:** GitHub/GitLab + Actions/CI.
- **Builds:** containerized builds, SBOM generation, signed artifacts.
- **Quality gates:** unit/integration/contract tests, SAST/DAST, IaC scanning, policy checks.
- **Deployment:** GitOps (ArgoCD/Flux), progressive rollout (blue-green/canary), automated rollback.
- **Release controls:** feature flags for strangler routing and phased cutover.

---

## Integration Flow

### 1) Employee authentication and authorization
1. User authenticates through channel application.
2. API Gateway forwards auth to Identity Service.
3. Identity Service validates credentials via IdP and retrieves workforce attributes (dept/role).
4. Token with claims is issued and propagated to domain services.
5. Security audit event is published to the event bus and SIEM.

### 2) Flight search to sale confirmation
1. Channel requests flights via Flight Catalog Service.
2. User selects flight and passenger; Sales Service initiates booking saga.
3. Sales Service validates passenger data and seat availability through domain APIs.
4. On confirmation, Sales Service commits booking transaction and publishes `BookingConfirmed`.
5. Ticket Inquiry read model updates asynchronously from events.
6. Fulfillment Service consumes `TicketIssued`/`ReceiptRequested`, generates document, updates job status.

### 3) Workforce and passenger ingestion
1. External feeds arrive via Ingestion Gateway (JSON/XML/AS400 exports).
2. Gateway validates schema, performs mapping, and emits normalized ingestion events.
3. Workforce/Passenger services process events idempotently and persist authoritative records.
4. Identity consumes workforce lifecycle events for access provisioning/deprovisioning.

### 4) Legacy coexistence during migration
1. Strangler router directs selected endpoints to new services; non-migrated flows remain on CICS.
2. Anti-corruption adapters translate COMMAREA/file contracts into modern APIs/events.
3. Dual-run reconciliation compares legacy vs modern outputs before final cutover.

---

## Data Migration Strategy

### Principles
- **Incremental, domain-wise migration** over big-bang conversion.
- **Authoritative ownership per bounded context** with explicit canonical IDs.
- **Dual-write only when unavoidable**, otherwise event-carried state transfer.
- **Deterministic reconciliation** with business-rule-level validation.

### Migration waves
1. **Foundation:** replicate reference/master entities (employee, department, airport, flight baselines).
2. **Read-side first:** build Flight and Ticket projections from DB2 CDC/events and validate parity.
3. **Identity cutover:** replace PASSDOC/custom crypto with IdP-backed credential model.
4. **Transactional migration:** move sales writes to new OLTP with idempotent command handling.
5. **Fulfillment migration:** reroute print/document workflow to Fulfillment Service.
6. **Legacy retirement:** disable legacy transaction paths, archive immutable records, decommission modules.

### Data movement mechanics
- DB2 **CDC pipeline** to streaming platform.
- Batch backfill for history + checksum validation.
- Upsert/idempotency keys for replay safety.
- Reconciliation dashboards for record-count, field-level, and business-outcome parity.
- Controlled cutover with rollback checkpoints per domain.

---

## Non-Functional Requirements (target commitments)

### Scalability
- Stateless services with horizontal autoscaling.
- Read/write separation (CQRS where needed) for inquiry-heavy flows.
- Cache-aside for hot flight/ticket queries.
- Event partitioning strategy by business key (bookingId, employeeId, ticketId).

### Resilience
- Active-active deployment across availability zones.
- Circuit breakers, retries with backoff, idempotent consumers.
- Outbox pattern for transactional event publishing.
- DLQ handling with replay tooling and operational runbooks.
- RTO/RPO defined per domain (e.g., Identity/Sales stricter than analytics).

### Performance
- API p95 latency targets by domain (e.g., <300ms for search, <500ms for inquiry).
- End-to-end booking completion SLO with observable workflow stages.
- Capacity plans tied to seasonal peaks and campaign surges.

### Security & compliance
- Compliance-ready controls for **PCI DSS** (if payment in scope), **GDPR/PII**, and enterprise IAM policy.
- Data minimization, retention policies, and right-to-erasure workflows.
- Immutable audit logs and segregation-of-duties in CI/CD.
- Continuous compliance scanning and evidence collection.

### Operability
- SLO-driven monitoring and error budgets.
- Golden signals + domain KPIs (login success rate, booking success, print completion rate).
- Standardized runbooks, incident taxonomy, and on-call escalation paths.

### Maintainability
- Backward-compatible API/event versioning policy.
- Contract testing between producer/consumer services.
- Domain ownership model with clear team boundaries and platform guardrails.

