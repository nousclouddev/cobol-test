# Legacy Module Analysis: `LOGIN` + `CRYPTVE`

## Code Summary
The `LOGIN` CICS transaction program implements a pseudo-conversational login flow. On first entry (`EIBCALEN = 0`) it sends the login screen with date/time; on subsequent entry it receives user input, validates identity, and routes users by department. Authentication is split into two steps: (1) read employee metadata (`ADMIDATE`, `DEPTID`) from DB2 `EMPLO`, and (2) call subprogram `CRYPTVE` to verify password against an encrypted password file. Department `7` is routed to `SRCHFLI`; all other departments currently show “map not available yet” messages. 

Main flow (`LOGIN`):
1. Initial call: prepare welcome message + time/date; send map; return with `TRANSID LOGP`.
2. Next call: receive map input (`USERIDI`, `PASSWI`).
3. Query DB2 employee row by `EMPID`.
4. If found, call `CRYPTVE` using password + user + admission date.
5. Process returned status and either:
   - show error message, or
   - route by `DEPTID` (only dept 7 uses `XCTL PROGRAM('SRCHFLI')`).

`CRYPTVE` opens sequential file `PASSDOC`, derives an 8-char transformed password using user id + password + admission date + `RANDOM` seed logic, scans file for matching user, and compares encrypted value.

## Business Logic Extracted
- **Authentication rule**: User must exist in `EMPLO` and password must match `PASSDOC` encrypted record.
- **Department routing rule**:
  - Dept 7 -> operational map/program `SRCHFLI`.
  - Depts 1,2,3,4,5,6,8,9 -> blocked with “not available yet” message.
- **Error/user feedback rule**:
  - Return flag `1` => “PASSWORD OR USERID INCORRECT.”
  - Return flag `2` => system communication error + error code display.
- **Session/pseudo-conversation rule**: uses CICS `RETURN TRANSID('LOGP') COMMAREA` pattern; transaction state persisted in COMMAREA fields (`TRANSACTION`, `USER-ID`).

## Data Access Patterns
- **Database operations**:
  - Single DB2 read (`SELECT ADMIDATE, DEPTID FROM EMPLO WHERE EMPID = :WS-USERID`).
  - No explicit `INSERT/UPDATE/DELETE` in analyzed modules.
- **File operations**:
  - Sequential file `PASSDOC` opened as `INPUT`.
  - Linear scan via repeated `READ PASSFILE` until user found or EOF.
- **Batch/file-like behavior**:
  - Password verification depends on offline-maintained sequential password store, implying external batch/population dependency.

## Dependencies
- CICS APIs: `SEND`, `RECEIVE`, `RETURN`, `XCTL`, `ASKTIME`, `FORMATTIME`.
- DB2 host structures: `INCLUDE EMPLO`, `INCLUDE SQLCA`.
- COBOL subprogram call: dynamic call to `CRYPTVE` (`CALL SUBPROGRAM`).
- External data asset: `PASSDOC` sequential file containing `{USERID, PASS}` records.

## Technical Smells
- **Hard-coded values/messages**:
  - Transaction id (`LOGP`), map names (`LOGON`, `LOGINMP`), target program (`SRCHFLI`), department numeric literals (1..9), and all user-facing messages are inline constants.
- **Weak/fragile crypto design**:
  - Home-grown reversible-looking transform logic instead of standard salted hash.
  - Uses `FUNCTION RANDOM(WS-DATE)` seeded from admission date, which is predictable.
  - Fixed 8-char password handling and byte-level manipulation increase collision/compatibility risks.
- **Data model coupling**:
  - Crypto depends on `ADMIDATE`; changing employee metadata may break login determinism.
- **Scalability concern**:
  - Sequential full scan of password file per login (O(n)) with no index.
- **Control-flow debt**:
  - `GO TO 0400-CLOSE-FILE` used for error exits.
  - Return code semantics overloaded across modules (`WS-FLAG-RETURN`, `LS-FLAG-RETURN`) and comments are partially inconsistent with actual flow.
- **Observability/security leakage**:
  - `DISPLAY WS-CRYPTPASS` and `DISPLAY LS-USERID` can leak sensitive authentication data to logs/console.

## Error Handling Strategy
- `LOGIN` evaluates `SQLCODE` from DB2:
  - `0`: proceed to crypto verification.
  - `100`: treated as invalid credentials path.
  - other: generic system error path.
- `CRYPTVE` checks file status (`FS-PASSFILE`) after open/read/close; sets return flag `2` and passes low-level file status code on failure.
- Error handling is mostly status-code driven with user-facing generic messages and limited structured diagnostics.

## Security Concerns
- No evidence of rate limiting, account lockout, or failed-attempt tracking.
- Potential username enumeration via differentiated backend outcomes combined with deterministic responses.
- Plaintext password is handled in working storage and passed across program boundary.
- Sensitive debug output present (`DISPLAY` on encrypted password/user id).
- Legacy file-based credential storage (`PASSDOC`) increases exposure and operational risk.

## Modernization Implications
1. Replace custom crypto + file store with centralized IAM (e.g., RACF/LDAP/OIDC bridge) and strong one-way password hashes (Argon2/bcrypt/PBKDF2).
2. Consolidate authentication source in DB2/service layer; remove dual-source auth (`EMPLO` + `PASSDOC`).
3. Externalize routing and messages (config/table-driven department-to-function mapping).
4. Introduce structured error taxonomy and audit logging without credential leakage.
5. Refactor pseudo-conversational flow into smaller testable units and remove `GO TO`-based error branches.
6. Add security controls: lockout policy, rate limiting, MFA integration points, and transport/session hardening.
