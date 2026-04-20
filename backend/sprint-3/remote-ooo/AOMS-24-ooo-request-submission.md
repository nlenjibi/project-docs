# AOMS-24 — OOO Request Submission (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-24 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api |
| **Perspective** | Backend |

---

## Story

As an employee, I want to submit an out-of-office request for a date range so that my manager can approve my absence and my attendance records are updated accordingly.

---

## Background (Backend Context)

OOO requests cover multi-day absence (annual leave, sick leave, etc.). Unlike `RemoteRequest` (single date), an `OOORequest` spans a `start_date` to `end_date` range. The overlap check must consider the full range — a new OOO request cannot overlap with any date in an existing approved OOO range for the same employee.

OOO requests are not subject to the remote day policy limit — they have their own leave type logic. On approval (handled in AOMS-25), attendance records for all dates in the approved range are pre-stamped as `ON_LEAVE` by the service.

This ticket also feeds into AOMS-28 (delegation) — when a manager submits their own OOO, the submission flow prompts for delegate nomination. The prompt itself is a UI concern (React), but the backend must return a flag in the response indicating that delegation is required when the submitter is a Manager role.

**Messaging:** SQS — `oms-notifications-queue`. **Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `OOORequest` entity: `id` (UUID), `user_id`, `location_id`, `manager_id` (FK → User, resolved at submission), `leave_type` (Enum: `ANNUAL_LEAVE`, `SICK_LEAVE`, `PERSONAL_DAY`, `BUSINESS_TRAVEL`, `OTHER`), `start_date` (Date), `end_date` (Date), `notes` (String, nullable), `status` (Enum: `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`), `reviewed_by` (UUID, nullable), `reviewed_at` (Timestamp, nullable), `rejection_reason` (String, nullable), `created_at`, `updated_at`, `deleted_at`.
- [ ] Flyway migration creates `ooo_requests` table.
- [ ] `POST /api/v1/ooo-requests` — submits an OOO request.
  - Request: `{ "leaveType": "ANNUAL_LEAVE", "startDate": "2026-04-14", "endDate": "2026-04-18", "notes": "Annual holiday" }`.
  - `startDate` must be a future date — return 400 for past or today.
  - `endDate` must be >= `startDate` — return 400 otherwise.
  - Overlap check: query for any existing PENDING or APPROVED `OOORequest` for `user_id` where `(startDate <= existing.end_date AND endDate >= existing.start_date)` — return 409 `{ "code": "OVERLAPPING_REQUEST", "overlappingRequestId": "uuid" }` if found.
  - Persist `OOORequest` with `status = PENDING`, `manager_id` resolved from `User.manager_id`.
  - Resolve effective approver (delegate check via `ApprovalDelegateRepository` — same pattern as AOMS-23).
  - Publish SQS message: `{ "type": "OOO_REQUEST_SUBMITTED", "approverId": "uuid", "requesterId": "uuid", "leaveType": "...", "startDate": "...", "endDate": "...", "locationId": "..." }`.
  - If the submitter holds a `MANAGER` role at the location, include `"requiresDelegation": true` in the response `data` to signal to the React UI that delegate nomination is recommended (leads into AOMS-28 flow).
  - Write to `AuditLog`.
- [ ] `GET /api/v1/ooo-requests/my` — paginated list of the authenticated employee's own OOO requests, filterable by `status`, `leaveType`, `from`, `to`. Includes `startDate`, `endDate`, `leaveType`, `status`, `reviewedBy`, `rejectionReason`.
- [ ] Employee cannot view another employee's OOO requests via this endpoint — `user_id` always sourced from session.
- [ ] Unit tests: future start date validation, end before start (400), overlap detection (409), requiresDelegation flag set for Manager role submitter, delegate routing in SQS.
- [ ] Integration test: submit OOO; verify DB record; verify overlap check rejects a conflicting second request.

---

## Implementation Notes

```
POST /api/v1/ooo-requests
    ↓
OOORequestService.submit(userId, locationId, leaveType, startDate, endDate, notes)
    ├── Validate: startDate > today, endDate >= startDate
    ├── Overlap check (single query with date range intersection)
    ├── Persist OOORequest (status = PENDING)
    ├── Resolve effective approver (delegate check)
    ├── Determine requiresDelegation: requestingUser has MANAGER role at location
    ├── SqsPublisher.send("oms-notifications-queue", { type: OOO_REQUEST_SUBMITTED, ... })
    └── AuditLogService.log(...)
```

- Overlap query (PostgreSQL): `SELECT id FROM ooo_requests WHERE user_id = :userId AND status IN ('PENDING', 'APPROVED') AND start_date <= :endDate AND end_date >= :startDate AND deleted_at IS NULL LIMIT 1`.
- `requiresDelegation` is returned in the response only when true — it does not affect DB state. AOMS-28 handles the actual delegation creation.
- The `manager_id` on the `OOORequest` is frozen at submission time.

---

## API Contract

```
POST /api/v1/ooo-requests
Request: { "leaveType": "ANNUAL_LEAVE", "startDate": "2026-04-14", "endDate": "2026-04-18", "notes": "Annual holiday" }

Response 201 (employee submitter):
{ "success": true, "data": { "requestId": "uuid", "leaveType": "ANNUAL_LEAVE", "startDate": "2026-04-14", "endDate": "2026-04-18", "status": "PENDING" }, "error": null }

Response 201 (manager submitter):
{ "success": true, "data": { ..., "requiresDelegation": true }, "error": null }

Response 409 (overlap):
{ "success": false, "data": null, "error": { "code": "OVERLAPPING_REQUEST", "message": "An approved or pending request already covers dates in this range.", "overlappingRequestId": "uuid" } }

GET /api/v1/ooo-requests/my?status=PENDING&leaveType=ANNUAL_LEAVE&page=0&size=20
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Overlap detection confirmed with date range intersection query
- [ ] `requiresDelegation` flag returned for Manager role submitters
- [ ] Audit log entry on every submission
- [ ] Employee cannot read another employee's requests
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- `User.manager_id` and role data populated from Arms HR sync.
- **AOMS-28** scaffolded to the point that `ApprovalDelegateRepository` is queryable (can be stubbed to return null).
- SQS queue `oms-notifications-queue` provisioned.
