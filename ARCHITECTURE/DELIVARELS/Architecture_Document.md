# Architecture Document: Target Technical Architecture

## 1. Executive Summary
The target architecture transitions the airline platform from monolithic COBOL/CICS transaction meshes to a domain-driven, service-oriented ecosystem. The design preserves business continuity through coexistence while progressively extracting capabilities from legacy programs.

## 2. Technical Architecture

### 2.1 Architectural Style
- Domain-aligned microservices with API-first contracts.
- Event-driven integration for cross-domain workflows.
- Incremental strangler pattern for low-risk decomposition.

### 2.2 Core Domains and Services
- **Identity & Access Service**: authentication, authorization, audit.
- **Flight Discovery Service**: schedule and availability queries.
- **Ticket Inquiry Service**: ticket/passenger lookup and pagination.
- **Sales Orchestration Service**: booking and confirmation state machine.
- **Passenger/Workforce Ingestion Service**: JSON/XML and workforce onboarding flows.
- **Document Fulfillment Service**: ticket/receipt rendering and print-dispatch abstraction.

### 2.3 Integration and Interface Layer
- API Gateway for north-south traffic and policy enforcement.
- Event bus for domain events (`BookingConfirmed`, `TicketIssued`, `ReceiptRequested`).
- Legacy adapter layer for CICS transaction bridging and DB2 interoperability.

### 2.4 Data Architecture
- Domain-owned schemas for new services.
- Controlled replication and projection patterns during coexistence.
- Migration waves prioritized by read-heavy flows before write-critical paths.

### 2.5 Security and Compliance
- Replace sequential credential-file validation with managed identity stack.
- Enforce token-based authorization across services.
- Centralize security telemetry and immutable audit trails.

### 2.6 Observability and Operations
- Distributed tracing across legacy-modern calls.
- SLO-driven monitoring and incident response.
- CI/CD with quality gates (static analysis, contract tests, integration tests).

## 3. Key Benefits
- Reduced systemic coupling and higher maintainability.
- Improved reliability through service isolation and retryable integration patterns.
- Measurable security uplift and stronger compliance readiness.

## 4. Lessons Learned
- Domain boundaries must be validated against runtime dependency maps, not just org charts.
- Ticket inquiry and sales orchestration are top-value but high-risk candidates; phase carefully.
- Legacy naming and contract drift (`SRCHFLI/SRCHFLY`, unresolved dependencies) must be addressed early.

## 5. Conclusion
The proposed architecture delivers a pragmatic path from legacy stability to digital agility by balancing immediate risk controls with long-term platform modernization goals.
