# Legacy System Dependency & Interaction Model

## Role and Objective
**Role:** System Integration Architect  
**Objective:** Build a logical dependency and interaction model of the legacy COBOL/CICS/DB2 system.

## Input Inventory

### Programs / Modules in Scope
- `CICS/LOGIN/LOGIN-COB` (`PROGRAM-ID LOGIN`)
- `CICS/LOGIN/CRYPTO-VERIFICATION` (`PROGRAM-ID CRYPTVE`)
- `CICS/SALES-MAP/SRCHFLY-COB` (`PROGRAM-ID SRCHFLY`)
- `CICS/SALES-MAP/SRCHTKT-COB` (`PROGRAM-ID SRCHTKT`)
- `CICS/SALES-MAP/SELL1-COB` (`PROGRAM-ID SELLCOB1`)
- `CICS/SALES-MAP/PRINT-TICKET-COB` (`PROGRAM-ID PRINTCI`)
- `CICS/SALES-MAP/RECEIPT-COB` (`PROGRAM-ID PRINTPA`)
- `COB-PROG/EMPLO-INSERT/EMPLO-MAIN-INSERT` (`PROGRAM-ID EMPINSRT`)
- `COB-PROG/EMPLO-INSERT/EMPLO-LECTURE-JSON` (`PROGRAM-ID SUINSRT`)
- `COB-PROG/EMPLO-INSERT/EMPLO-CRYPTO-PASS` (`PROGRAM-ID CRYPTPGM`)
- `COB-PROG/PASSENGER-INSERT/PASSENGER-INSERT-MAINPROG` (`PROGRAM-ID PASSENG`)
- `COB-PROG/PASSENGER-INSERT/PASSENGER-SUXML-SUBPROG` (`PROGRAM-ID SUXML`)
- `AS-400/Insert/INSERTCSV` (`PROGRAM-ID ISRTJSON`)
- `AS-400/Insert/encryptpgm` (`PROGRAM-ID CRYPTPGM`)

### Database Tables (from DB2 DDL)
- `AIRPORT`, `AIRPLANE`, `FLIGHT`, `PASSENGERS`, `TICKET`, `EMPLO`, `DEPT`, `BUY`, `CREW`, `SHIFT`

### Interfaces
- CICS BMS map interfaces (`LOGINMAP`, `SELL1/SELL2`, `SRCHFLI/SRCHTKT` map contracts)
- CICS pseudo-conversation and COMMAREA transfer
- CICS spool subsystem (`SPOOLOPEN`, `SPOOLWRITE`, `SPOOLCLOSE`)
- Sequential credential files (`PASSDOC`/`DISK-FILEPASS`)
- Batch file interfaces: JSON (`EMPLOYEE-LIST.json`), XML split files (`PASSENGER1..8.xml`)

---

## Dependency Graph (textual)

### Program → Program Dependencies
```text
LOGIN
  -> CALL CRYPTVE
  -> XCTL SRCHFLI

SRCHFLY
  -> XCTL LOGIN
  -> XCTL SRCHTKT
  -> XCTL SELLCOB1

SRCHTKT
  -> XCTL LOGIN
  -> XCTL SRCHFLY
  -> XCTL SELLCOB1
  -> SPOOL subsystem path

SELLCOB1
  -> XCTL LOGIN
  -> XCTL SRCHFLY
  -> XCTL SRCHTKT
  -> XCTL SELLCOB2 (target not present in current repo)

EMPINSRT
  -> CALL SUINSRT

SUINSRT
  -> CALL CRYPTPGM

PASSENG
  -> CALL SUXML

ISRTJSON
  -> CALL CRYPTPGM
```

### Program → Database Dependencies
```text
LOGIN    -> EMPLO
SRCHFLY  -> FLIGHT
SRCHTKT  -> TICKET + PASSENGERS + FLIGHT + AIRPORT
SELLCOB1 -> PASSENGERS + FLIGHT
EMPINSRT -> EMPLO (insert)
PASSENG  -> PASSENGERS (insert)
ISRTJSON -> EMPLOYEE (AS/400 naming variant)
```

### Program → External System Dependencies
```text
LOGIN / SRCHFLY / SRCHTKT / SELLCOB1
  -> CICS BMS maps + COMMAREA + TRANSID loops

SRCHTKT
  -> CICS spool and downstream print path

CRYPTVE / CRYPTPGM
  -> Sequential password file interface

EMPINSRT / SUINSRT
  -> JSON ingestion source

PASSENG / SUXML
  -> XML ingestion source (segmented files)
```

---

## Critical Flow Paths
1. **Authentication & Routing**  
   `LOGIN -> EMPLO lookup -> CRYPTVE -> XCTL department entry`  
   Any break here blocks all online users.

2. **Sales Navigation Core**  
   `SRCHFLY <-> SRCHTKT <-> SELLCOB1`  
   Bidirectional XCTL chain forms a runtime-critical navigation mesh.

3. **Ticket Search & Print**  
   `SRCHTKT -> DB2 joins/cursors -> CICS spool -> print modules`  
   Depends on DB integrity and spool/JCL availability.

4. **Employee Provisioning**  
   `EMPINSRT -> SUINSRT -> CRYPTPGM -> INSERT EMPLO`  
   Data onboarding and credential generation are tightly intertwined.

5. **Passenger Provisioning**  
   `PASSENG -> SUXML -> INSERT PASSENGERS`  
   Sensitive to XML transfer segmentation and parser assumptions.

---

## Coupling Analysis

### Strong Coupling
- **LOGIN ↔ CRYPTVE ↔ password file contract:** shared encryption assumptions and fixed record layout.
- **Sales transaction programs:** hardcoded XCTL targets and COMMAREA conventions.
- **Embedded SQL ↔ DCLGEN/table shape:** compile-time dependency to schema and host variable structure.
- **SRCHTKT UI-state + query intermixing:** pagination and data retrieval are implemented in the same unit.

### Medium Coupling
- **Batch main ↔ parser subprogram pairs:** explicit interfaces but still procedural and synchronous.
- **Print programs ↔ operations/JCL model:** operationally coupled, functionally separable.
- **AS/400 insert flow ↔ crypto routine:** reusable call boundary exists but with hardcoded naming/contracts.

### Weak Coupling
- **DDL artifacts ↔ runtime programs:** mostly deployment/maintenance-time dependency.
- **Image/map documentation files ↔ executable logic:** low runtime dependency.

---

## Candidate Microservice Boundaries
1. **Identity & Access Service**
   - Login, credential validation, role/department resolution.
   - Source: `LOGIN`, `CRYPTVE`, employee auth data in `EMPLO`.

2. **Flight Discovery Service**
   - Flight search/read APIs by route, date, flight number.
   - Source: `SRCHFLY`.

3. **Ticket Inquiry Service**
   - Ticket + passenger + flight lookup and paginated read models.
   - Source: `SRCHTKT`.

4. **Sales Transaction Service**
   - Sale orchestration, booking workflow, pricing/confirmation.
   - Source: `SELLCOB1` (+ unresolved `SELLCOB2`).

5. **Ingestion Service (HR + Passenger)**
   - File intake, validation, mapping, and insert orchestration.
   - Source: `EMPINSRT`, `SUINSRT`, `PASSENG`, `SUXML`, `ISRTJSON`.

6. **Document & Print Service**
   - Receipt/ticket render + queue submission.
   - Source: `SRCHTKT` spool logic, `PRINTCI`, `PRINTPA`.

---

## Refactoring Opportunities
1. **Decouple navigation from business logic** by replacing direct XCTL mesh with a routing abstraction.
2. **Extract shared SQL access layer** from SRCHFLY/SRCHTKT into reusable query subprograms.
3. **Replace file-based credential validation** with a managed identity store and policy enforcement.
4. **Standardize canonical naming** (`EMPLO` vs `EMPLOYEE`, `PASSENG` vs `PASSENGERS`) via adapter mapping.
5. **Split SRCHTKT by concern** (query, paging state, print trigger).
6. **Unify ingestion contracts** across JSON/XML/AS400 loaders.
7. **Stabilize print integration** with queue commands and retry/monitoring semantics.
8. **Add dependency contract checks** for XCTL targets, map names, and SQL object existence.

---

## Priority Modernization Sequence
1. Resolve naming and target-program drift (`SRCHFLI/SRCHFLY`, `SELLCOB2` dependency).
2. Isolate authentication and remove sequential password-file dependency.
3. Decompose SRCHTKT and sales orchestration hotspots.
4. Introduce ingestion and print boundaries as early strangler candidates.
5. Expose read-only services first (flight/ticket inquiry), then move write flows.
