# AOMS-25 — Manager Request Approval (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-25 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api, manager |
| **Perspective** | Backend |

---

## Story

As a manager, I want to approve or reject remote day and OOO requests from my direct reports so that I can manage my team's availability effectively.

---

## Background (Backend Context)

The approval action has a downstream side effect that is the critical integration point between the Remote/OOO domain and the Attendance domain: **on approval, attendance records must be pre-stamped**. Specifically:

- Approving a `RemoteRequest` → pre-stamp the corresponding `AttendanceRecord` as `REMOTE` for that date.
- Approving an `OOORequest` → pre-stamp `AttendanceRecord` as `ON_LEAVE` for each date in the approved range.

This pre-stamping is a direct service-layer call (not a nightly job action) so that the employee's calendar and attendance views reflect the approval immediately, without waiting for the next nightly run. If an `AttendanceRecord` doesn't yet exist for a future date, it is created. If one already exists (e.g. the nightly job ran and stamped `ABSENT`), it is updated — unless `is_overridden = true`.

The approver may be the original manager OR an active delegate (AOMS-28). Both are valid actors for approval/rejection.

**Messaging:** SQS — `oms-notifications-queue`. **Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] **`PATCH /api/v1/remote-requests/{requestId}/approve`** — approves a remote day request.
  - Accessible to: the `manager_id` on the request, or an active `ApprovalDelegate` for that manager.
  - Request status must be `PENDING` — return 409 if already approved/rejected/cancelled.
  - If approving would cause the team's remote count on that date to exceed `policy.team_overlap_threshold_percent`, return a soft-warning in the response but **do not block** (manager can proceed).
  - Set `status = APPROVED`, `reviewed_by = actorId`, `reviewed_at = now()`.
  - Pre-stamp attendance: call `AttendancePreStampService.stampRemote(userId, requestDate, requestId, locationId)` — upserts `AttendanceRecord` with `status = REMOTE`, `remote_request_id = requestId` (skip if `is_overridden = true`).
  - Publish SQS `{ "type": "REMOTE_REQUEST_APPROVED", "requesterId": "uuid", "requestDate": "...", "locationId": "..." }`.
  - Write to `AuditLog`.

- [ ] **`PATCH /api/v1/remote-requests/{requestId}/reject`** — rejects a remote day request.
  - Request: `{ "reason": "Team already at remote capacity this week" }`.
  - `reason` is mandatory — return 400 if absent or blank.
  - Accessible to same actors as approve.
  - Request status must be `PENDING`.
  - Set `status = REJECTED`, `reviewed_by`, `reviewed_at`, `rejection_reason`.
  - Publish SQS `{ "type": "REMOTE_REQUEST_REJECTED", "requesterId": "uuid", "reason": "...", "locationId": "..." }`.
  - Write to `AuditLog`.

- [ ] **`PATCH /api/v1/ooo-requests/{requestId}/approve`** — approves an OOO request.
  - Same access control as remote approval.
  - Pre-stamp attendance for **all dates** in `[startDate, endDate]` inclusive: call `AttendancePreStampService.stampLeave(userId, startDate, endDate, requestId, locationId)` — upserts `AttendanceRecord` with `status = ON_LEAVE`, `leave_request_id = requestId` for each date. Skip public holidays (they retain `PUBLIC_HOLIDAY` status). Skip dates where `is_overridden = true`.
  - Publish SQS `{ "type": "OOO_REQUEST_APPROVED", "requesterId": "uuid", "startDate": "...", "endDate": "...", "locationId": "..." }`.
  - Write to `AuditLog`.

- [ ] **`PATCH /api/v1/ooo-requests/{requestId}/reject`** — rejects an OOO request. Same pattern as remote rejection.

- [ ] **`GET /api/v1/approvals/pending`** — returns all PENDING requests (both remote and OOO) that the authenticated user is authorised to approve.
  - For Manager: all PENDING requests where `manager_id = authenticatedUserId` plus any where the authenticated user is an active delegate for another manager.
  - For HR: all PENDING requests at their location (HR sees all).
  - For Super Admin: all PENDING at any location.
  - Sorted by `created_at ASC` (oldest first).
  - Response includes `requestType: "REMOTE" | "OOO"` discriminator field.

- [ ] Unit tests: approve remote → attendance pre-stamp created; approve OOO range → attendance records created for each date; rejection requires reason; delegate can approve; non-authorised manager cannot approve another manager's team (403); overlap soft-warning on approval.
- [ ] Integration test: submit remote request; manager approves; verify `AttendanceRecord` for that date has `status = REMOTE`.

---

## Implementation Notes

```
AttendancePreStampService (new shared service):

stampRemote(userId, date, remoteRequestId, locationId):
    AttendanceRecordRepository.upsert(
        user_id=userId, record_date=date, location_id=locationId,
        status=REMOTE, remote_request_id=remoteRequestId
    ) WHERE is_overridden = false

stampLeave(userId, startDate, endDate, oooRequestId, locationId):
    For each date in [startDate, endDate]:
        Skip if PublicHoliday exists for (locationId, date)
        AttendanceRecordRepository.upsert(
            status=ON_LEAVE, leave_request_id=oooRequestId
        ) WHERE is_overridden = false
```

- The upsert for attendance pre-stamping uses `INSERT ... ON CONFLICT (user_id, record_date, location_id) DO UPDATE SET status = ..., remote_request_id = ... WHERE is_overridden = false`.
- The overlap soft-warning check on approval: count APPROVED remote requests at the same location on the same date; compute as a % of the team size. If > `policy.team_overlap_threshold_percent` → include `"warning"` in the approval response but still approve.
- The approver authorisation check: `actor.id == request.managerId OR ApprovalDelegateRepository.findActive(managerId=request.managerId, delegateId=actor.id)` not null.

---

## API Contract

```
PATCH /api/v1/remote-requests/{requestId}/approve
Response 200 (clean): { "success": true, "data": { "requestId": "...", "status": "APPROVED" } }
Response 200 (soft warning): { "success": true, "data": { "status": "APPROVED" }, "warning": "Team overlap threshold of 50% exceeded on this date." }

PATCH /api/v1/remote-requests/{requestId}/reject
Request: { "reason": "Team at remote capacity" }
Response 200: { "success": true, "data": { "requestId": "...", "status": "REJECTED", "rejectionReason": "..." } }

PATCH /api/v1/ooo-requests/{requestId}/approve
Response 200: { "success": true, "data": { "requestId": "...", "status": "APPROVED", "datesStamped": 5 } }

GET /api/v1/approvals/pending?page=0&size=20
Response 200: { "success": true, "data": { "content": [{ "requestId", "requestType", "requesterName", "date(s)", "status": "PENDING", "submittedAt" }, ...], "totalElements": 12, ... } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Attendance pre-stamp confirmed for both REMOTE and ON_LEAVE in integration tests
- [ ] `is_overridden = true` records not overwritten — confirmed in test
- [ ] Delegate approval path confirmed in unit test
- [ ] Non-authorised actor returns 403
- [ ] Rejection reason mandatory — 400 if missing
- [ ] Overlap soft-warning returns correctly without blocking
- [ ] Audit log entries for all approve and reject actions
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-23** and **AOMS-24** must be complete (`RemoteRequest` and `OOORequest` entities).
- **AOMS-27** must be complete (policy for overlap threshold check).
- **AOMS-28** scaffolded — `ApprovalDelegateRepository` queryable.
- **AOMS-4** / **AOMS-5** must be complete — `AttendanceRecord` entity and upsert pattern established.
- **ATT-P03** must be complete.
