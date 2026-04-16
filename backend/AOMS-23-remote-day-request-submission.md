# AOMS-23 — Remote Day Request Submission (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-23 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api |
| **Perspective** | Backend |

---

## Story

As an employee, I want to submit a remote day request for a specific date so that my manager is informed and my attendance record reflects that I am working remotely.

---

## Background (Backend Context)

Remote day request submission is the primary write operation in the Remote/OOO domain. It has two non-trivial responsibilities:

1. **Policy enforcement** — before persisting, the service resolves the effective `RemoteDayPolicy` for the employee's manager (via `RemoteDayPolicyResolver` from AOMS-27) and checks whether the weekly limit has been reached. Depending on `enforcement_mode`, the service either blocks (`HARD_BLOCK`) or allows with a warning response (`SOFT_WARNING`).

2. **Manager routing** — the manager who receives the notification is determined by the employee's `User.manager_id`. If the manager is currently OOO and has a delegate active (AOMS-28), the notification is routed to the delegate instead.

On approval (AOMS-25), this request triggers a REMOTE status pre-stamp on the attendance record. That stamp is written by AOMS-25 — this ticket only creates the `RemoteRequest` in `PENDING` status.

**Messaging:** SQS — `oms-notifications-queue`. **Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `RemoteRequest` entity: `id` (UUID), `user_id`, `location_id`, `manager_id` (FK → User — resolved at submission time, not re-read on approval), `request_date` (Date), `notes` (String, nullable), `status` (Enum: `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`), `reviewed_by` (UUID, nullable), `reviewed_at` (Timestamp, nullable), `rejection_reason` (String, nullable), `recurring_schedule_id` (UUID FK → RecurringRemoteSchedule, nullable), `created_at`, `updated_at`, `deleted_at`.
- [ ] Flyway migration creates `remote_requests` table. Unique constraint on `(user_id, request_date, location_id)` WHERE `status IN ('PENDING', 'APPROVED')` — prevents duplicate active requests for the same date.
- [ ] `POST /api/v1/remote-requests` — submits a remote day request.
  - Request: `{ "requestDate": "2026-04-07", "notes": "Working from home" }`.
  - `requestDate` must be a future date — return 400 for past or today.
  - Check for existing PENDING or APPROVED request on the same date for the same user — return 409 `{ "code": "DUPLICATE_REQUEST" }`.
  - Resolve effective policy via `RemoteDayPolicyResolver.resolve(managerId, locationId)`.
  - Count approved remote days in the current ISO week for the employee: `SELECT COUNT(*) FROM remote_requests WHERE user_id = ? AND WEEK(request_date) = WEEK(:requestDate) AND status = 'APPROVED'`.
  - If count >= `policy.max_remote_days_per_week`:
    - `HARD_BLOCK`: return HTTP 400 `{ "code": "REMOTE_LIMIT_REACHED", "message": "Weekly remote day limit of {N} has been reached." }`.
    - `SOFT_WARNING`: persist the request and return HTTP 201 with `{ "data": {...}, "warning": "Weekly remote day limit of {N} has been reached. Request submitted but may be declined." }`.
  - If no policy exists (null): proceed without limit check.
  - Persist `RemoteRequest` with `status = PENDING`, `manager_id` resolved from `User.manager_id`.
  - Resolve effective approver: if manager has an active `ApprovalDelegate` (AOMS-28), notify the delegate; otherwise notify the manager. Store the approver's `user_id` in a transient field for the SQS message — do not change `manager_id` on the record.
  - Publish SQS message to `oms-notifications-queue`: `{ "type": "REMOTE_REQUEST_SUBMITTED", "approverId": "uuid", "requesterId": "uuid", "requestDate": "...", "locationId": "..." }`.
  - Write to `AuditLog`.
- [ ] Returns the created `RemoteRequest` DTO in the response.
- [ ] Unit tests: future date only, duplicate rejection (409), HARD_BLOCK enforcement, SOFT_WARNING passes with warning, no-policy passes freely, delegate routing (SQS message targets delegate, not manager).
- [ ] Integration test: submit request; verify DB record; verify SQS message published with correct `approverId`.

---

## Implementation Notes

```
POST /api/v1/remote-requests
    ↓
RemoteRequestService.submit(userId, locationId, requestDate, notes)
    ├── Validate: requestDate > today
    ├── Check duplicate: no PENDING/APPROVED for same user+date
    ├── Resolve manager: User.manager_id
    ├── Resolve policy: RemoteDayPolicyResolver.resolve(managerId, locationId)
    ├── Count weekly approved remotes for user
    ├── Apply HARD_BLOCK or SOFT_WARNING (or skip if no policy)
    ├── RemoteRequestRepository.save(request)
    ├── Resolve effective approver (delegate check — ApprovalDelegateRepository)
    ├── SqsPublisher.send("oms-notifications-queue", { type: REMOTE_REQUEST_SUBMITTED, approverId, ... })
    └── AuditLogService.log(...)
```

- The weekly count query must use the same ISO week definition that the policy configuration uses. Use `YEARWEEK(request_date, 1)` in MySQL or `DATE_TRUNC('week', request_date)` in PostgreSQL — confirm with the DB choice from AOMS-9.
- `manager_id` on the `RemoteRequest` is frozen at submission time — it reflects who was the employee's manager when the request was submitted, even if the manager changes later.
- The delegate resolution is read-only here — it affects only which `approverId` goes into the SQS message. AOMS-28 owns the `ApprovalDelegate` entity.

---

## API Contract

```
POST /api/v1/remote-requests
Request: { "requestDate": "2026-04-07", "notes": "Client site visit" }

Response 201 (no limit issue):
{ "success": true, "data": { "requestId": "uuid", "requestDate": "2026-04-07", "status": "PENDING", ... }, "error": null }

Response 201 (SOFT_WARNING):
{ "success": true, "data": { ... }, "warning": "Weekly remote day limit of 3 reached. Request submitted.", "error": null }

Response 400 (HARD_BLOCK):
{ "success": false, "data": null, "error": { "code": "REMOTE_LIMIT_REACHED", "message": "Weekly remote day limit of 3 has been reached." } }

Response 409 (duplicate):
{ "success": false, "data": null, "error": { "code": "DUPLICATE_REQUEST", "message": "An active request already exists for this date." } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] HARD_BLOCK and SOFT_WARNING paths confirmed in tests
- [ ] Delegate routing confirmed in test (SQS message targets delegate when active)
- [ ] Audit log entry on every submission
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-27** must be complete (`RemoteDayPolicyResolver` bean must exist).
- **AOMS-28** scaffolded to the point that `ApprovalDelegateRepository` is queryable (delegate check in submission). Can be stubbed to return null if AOMS-28 is not yet complete.
- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- `User.manager_id` populated from Arms HR sync.
- SQS queue `oms-notifications-queue` provisioned.
