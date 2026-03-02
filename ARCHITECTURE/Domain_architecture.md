# DDD Domain Architecture for Legacy Login, Sales, and Back-Office Flows

## Domain List

### Core Domains
1. **Identity & Access**
   - Authenticates employees and determines operational access based on department.
   - Business critical because every online transaction depends on successful login and authorization.

2. **Sales & Booking Operations**
   - Orchestrates customer sales flow across flight lookup, ticket inquiry, and sell transaction steps.
   - Revenue-generating core capability.

3. **Ticketing & Fulfillment**
   - Retrieves ticket/passenger details and drives receipt/ticket print fulfillment.
   - Critical for customer confirmation and operational completion.

### Supporting Domains
1. **Flight Discovery**
   - Provides searchable flight availability and flight detail retrieval.
   - Enables sales decisions but is not, by itself, the transactional completion domain.

2. **Customer & Passenger Records**
   - Maintains passenger master/transactional identity records used by inquiry and sales.
   - Supports downstream ticketing and sales contexts.

3. **Workforce Provisioning (HR Ingestion)**
   - Onboards employee records and credentials from external files (JSON/AS-400 feeds).
   - Supports the Identity domain.

4. **Document & Print Operations**
   - Manages spool/print integration and document output lifecycle.
   - Supports ticketing completion and operational reporting.

### Generic Domains
1. **File Ingestion & Parsing Utilities**
   - JSON/XML parsing, segmented file handling, and mapping helpers.

2. **Cryptographic Utility (Legacy)**
   - Shared encryption/verification routines currently used by auth/provisioning.
   - Should evolve toward platform IAM and standard hash/policy services.

3. **Reference Data Management**
   - Reusable access to low-volatility entities (e.g., airport/department metadata).

---

## Bounded Context Map

### Contexts and Relationships
1. **Identity Context**
   - Owns employee authentication, credential policy enforcement, and access claims.
   - Upstream for all online channels.

2. **Sales Orchestration Context**
   - Owns booking workflow state transitions (search -> select -> sell -> confirm).
   - Consumes identity claims from Identity Context.
   - Consumes flight/ticket read models from Flight and Ticket contexts.

3. **Flight Catalog Context**
   - Owns flight schedule/availability query models.
   - Publishes read APIs/events consumed by Sales and Ticket Inquiry contexts.

4. **Ticket Inquiry Context**
   - Owns ticket-centric read models and pagination/search behavior.
   - Integrates passenger and flight projections for inquiry use cases.

5. **Passenger Management Context**
   - Owns passenger entity lifecycle and passenger identifiers.
   - Supplies canonical passenger data to Ticket and Sales contexts.

6. **Workforce Management Context**
   - Owns employee onboarding and employee master updates.
   - Publishes employee lifecycle events to Identity Context (e.g., hire, update, deactivate).

7. **Document Fulfillment Context**
   - Owns generation and dispatch of printable outputs and queue status.
   - Consumes ticket/sales completion events.

### Context Interaction Pattern (Target)
- **Identity -> Sales/Ticket/Flight**: token/claim-based authorization contract.
- **Flight Catalog -> Sales/Ticket Inquiry**: query API + cached read model.
- **Passenger Management -> Ticket Inquiry/Sales**: passenger profile service and reference ID contract.
- **Sales Orchestration -> Ticket Inquiry/Document Fulfillment**: domain events (`BookingConfirmed`, `TicketIssued`, `ReceiptRequested`).
- **Workforce Management -> Identity**: asynchronous employee lifecycle events.

---

## Domain Responsibilities

### Identity & Access
- Validate credentials and enforce authentication policy.
- Resolve department/role to authorization entitlements.
- Publish session/claim artifacts for downstream contexts.
- Record security audit events (without credential leakage).

### Sales & Booking Operations
- Manage booking state machine and transaction integrity.
- Coordinate inventory checks, passenger assignment, and sale confirmation.
- Emit business events for fulfillment and reporting.

### Ticketing & Fulfillment
- Serve ticket search/inquiry use cases with pagination.
- Bind ticket details to passengers and flight references.
- Trigger and track document fulfillment requests.

### Flight Discovery
- Provide route/date/flight-number search and filtering.
- Expose consistent flight read models for channels.

### Customer & Passenger Records
- Maintain canonical passenger profile and identifiers.
- Validate passenger data quality and deduplication rules.

### Workforce Provisioning
- Ingest employee data from batch sources.
- Validate and normalize employee records before persistence.
- Trigger credential bootstrap/update workflows.

### Document & Print Operations
- Format ticket/receipt payloads for output channels.
- Submit/monitor print queue jobs with retry semantics.

### Generic Utility Domains
- Standardized ingestion/parsing frameworks.
- Shared security primitives (to be replaced by managed IAM crypto policy).
- Reference data serving abstractions.

---

## Data Ownership Model

### Authoritative Ownership by Context
- **Identity Context**: credentials, authentication state, access claims, security events.
- **Workforce Management Context**: employee master profile (employment attributes, department assignment).
- **Passenger Management Context**: passenger master data and identity keys.
- **Flight Catalog Context**: flight schedules, availability projections, route metadata.
- **Ticket Inquiry Context**: ticket-centric query projections and inquiry indexes.
- **Sales Orchestration Context**: booking transaction state, reservation lifecycle, sale outcomes.
- **Document Fulfillment Context**: print job state, document dispatch audit.

### Shared Data Principles
- No cross-context direct table ownership; interaction via service contracts/events.
- Read-only projections allowed in consuming contexts.
- Identity claims, passenger IDs, flight IDs, booking IDs, and ticket IDs are canonical integration keys.
- Employee department should be mastered in Workforce and consumed by Identity via event replication.

### Legacy-to-Target Ownership Corrections
- Move credential logic out of file-based stores into Identity-owned credential repository.
- Keep employee demographic/HR attributes out of Identity (owned by Workforce).
- Separate ticket inquiry projections from sales write transactions for scalability.

---

## Suggested Service Decomposition

1. **Identity Service**
   - Capabilities: authenticate, authorize, token/claims issuance, credential policy.
   - Boundary: owns authentication workflow and security logs.

2. **Workforce Service**
   - Capabilities: employee onboarding/updates, department assignment, lifecycle events.
   - Boundary: system of record for employee master data.

3. **Flight Catalog Service**
   - Capabilities: flight search/read endpoints and availability views.
   - Boundary: read-optimized flight domain service.

4. **Passenger Service**
   - Capabilities: passenger CRUD, validation, identity resolution.
   - Boundary: passenger canonical registry.

5. **Sales Service**
   - Capabilities: booking orchestration, reservation commits, sale completion.
   - Boundary: write-intensive transactional workflow engine.

6. **Ticket Inquiry Service**
   - Capabilities: ticket/passenger/flight joined inquiry, pagination, status views.
   - Boundary: query model service (CQRS-style read side).

7. **Fulfillment Service**
   - Capabilities: receipt/ticket document generation, queue submission, job tracking.
   - Boundary: output-channel abstraction and operational retries.

8. **Ingestion Gateway Service**
   - Capabilities: file intake (JSON/XML/legacy), schema validation, transformation orchestration.
   - Boundary: anti-corruption layer for external data feeds.

### Recommended Rollout Sequence
1. Stand up Identity + Workforce boundaries first to remove credential risk concentration.
2. Split read services (Flight Catalog, Ticket Inquiry) from sales write flow.
3. Isolate Fulfillment to decouple spool/print operational concerns.
4. Introduce Ingestion Gateway as anti-corruption layer for batch integrations.
5. Incrementally migrate Sales orchestration after read and identity contracts are stable.
