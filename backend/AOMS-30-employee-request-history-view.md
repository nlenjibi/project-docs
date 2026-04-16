# AOMS-30 — Employee Request History View (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-30 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api |
| **Perspective** | Backend |

---

## Story

As an employee, I want to view the history and status of all my remote day and OOO requests so that I can track what has been approved, rejected, or is still pending.

---

## Background (Backend Context)

This ticket delivers the read API that backs the React request history view, plus the self-service cancellation flow for two scenarios: cancelling a `PENDING` request (before manager action) and cancelling an approved future request. The two cancellation paths have different business rules.

Cancelling a `PENDING` request is straightforward — the employee withdraws before the manager reviews it. The manager should receive a notification that the request was withdrawn.

Cancelling an `APPROVED` future request is more nuanced: the attendance pre-stamp must be reversed (the `AttendanceRecord` status reverted from `REMOTE` / `ON_LEAVE` to `ABSENT` or removed), and the manager is notified. The employee cannot "un-cancel" — they must submit a new request.

**Messaging:** SQS — `oms-notifications-queue`. **Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] **`GET /api/v1/remote-requests/my`** — returns the authenticated employee's own `RemoteRequest` records, paginated.
  - Filterable by: `status` (multi-value), `from` (date), `to` (date).
  - Response per record: `{ requestId, requestDate, status, notes, submittedAt, reviewedBy (name), reviewedAt, rejectionReason, isRecurring (bool), recurringScheduleId }`.
  - Employee cannot view another employee's records — `user_id` always from session.

- [ ] **`GET /api/v1/ooo-requests/my`** — same pattern for `OOORequest` records.
  - Response per record: `{ requestId, leaveType, startDate, endDate, status, notes, submittedAt, reviewedBy (name), reviewedAt, rejectionReason }`.

- [ ] **`GET /api/v1/requests/my`** — unified list combining both `RemoteRequest` and `OOORequest` records for the authenticated employee, sorted by `submittedAt DESC`.
  - Each item has a `requestType: "REMOTE" | "OOO"` discriminator.
  - Filterable by: `requestType`, `status`, `from`, `to`.
  - Paginated.

- [ ] **`DELETE /api/v1/remote-requests/{requestId}`** — cancels a PENDING remote day request.
  - Employee can only cancel their own request.
  - Status must be `PENDING` — return 409 if `APPROVED`, `REJECTED`, or already `CANCELLED`.
  - Sets `status = CANCELLED`, `cancelled_at = now()`.
  - Publish SQS `{ "type": "REMOTE_REQUEST_CANCELLED_BY_EMPLOYEE", "managerId": "uuid", "requesterId": "uuid", "requestDate": "..." }` — manager is notified of the withdrawal.
  - Write to `AuditLog`.

- [ ] **`DELETE /api/v1/ooo-requests/{requestId}`** — cancels a PENDING OOO request. Same pattern as above.

- [ ] **`POST /api/v1/remote-requests/{requestId}/cancel-approved`** — cancels an already-APPROVED future remote day request.
  - `requestDate` must be in the future — return 400 if `requestDate <= today`.
  - Status must be `APPROVED`.
  - Employee can only cancel their own.
  - Sets `status = CANCELLED`.
  - Reverse attendance pre-stamp: call `AttendancePreStampService.revertStamp(userId, requestDate, locationId)` — sets `AttendanceRecord.status = ABSENT`, clears `remote_request_id`. Skip if `AttendanceRecord.is_overridden = true`.
  - Publish SQS `{ "type": "REMOTE_REQUEST_CANCELLED_APPROVED", "managerId": "uuid", "requesterId": "uuid", "requestDate": "..." }`.
  - Write to `AuditLog`.

- [ ] **`POST /api/v1/ooo-requests/{requestId}/cancel-approved`** — cancels an already-APPROVED future OOO request.
  - `startDate` must be in the future — return 400 if `startDate <= today` (past dates cannot be un-approved).
  - Reverse all attendance pre-stamps for the OOO date range: `AttendancePreStampService.revertLeaveStamps(userId, startDate, endDate, locationId)` — sets `AttendanceRecord.status = ABSENT` for each date, clears `leave_request_id`. Skip overridden records.
  - Publish SQS notification.
  - Write to `AuditLog`.

- [ ] Unit tests: own records only, combined list sorted correctly, cancel PENDING (200), cancel APPROVED future → attendance reverted, cancel APPROVED past date (400), cancel another employee's request (403).
- [ ] Integration test: employee submits and approves remote request; calls `cancel-approved`; verifies `AttendanceRecord` reverted to `ABSENT`; verifies SQS message published.

---

## Implementation Notes

```
AttendancePreStampService (extends from AOMS-25):

revertStamp(userId, date, locationId):
    UPDATE attendance_records
    SET status = 'ABSENT', remote_request_id = NULL, updated_at = NOW()
    WHERE user_id = :userId AND record_date = :date AND location_id = :locationId
      AND is_overridden = false

revertLeaveStamps(userId, startDate, endDate, locationId):
    UPDATE attendance_records
    SET status = 'ABSENT', leave_request_id = NULL, updated_at = NOW()
    WHERE user_id = :userId AND record_date BETWEEN :startDate AND :endDate
      AND location_id = :locationId AND is_overridden = false
```

- `revertStamp` and `revertLeaveStamps` set status to `ABSENT` rather than deleting the record — the record was pre-created on approval and should remain visible in history as `ABSENT`.
- The unified `GET /api/v1/requests/my` endpoint can be implemented as a UNION query or as two parallel service calls assembled in the controller. For simplicity, two parallel queries assembled in Java is preferred — avoids a complex UNION with heterogeneous columns.
- Recurring schedule child instances cancelled via the parent `DELETE /api/v1/recurring-remote-schedules/{id}` (AOMS-26) — this endpoint only handles non-recurring or single-instance cancellations. For a recurring instance, direct the employee to use `DELETE /api/v1/remote-requests/{id}/cancel-instance` from AOMS-26.

---

## API Contract

```
GET /api/v1/requests/my?requestType=REMOTE&status=PENDING,APPROVED&page=0&size=20
Response 200:
{
  "success": true,
  "data": {
    "content": [
      {
        "requestId": "uuid",
        "requestType": "REMOTE",
        "requestDate": "2026-04-07",
        "status": "APPROVED",
        "submittedAt": "2026-03-20T10:00:00Z",
        "reviewedByName": "John Manager",
        "isRecurring": false
      },
      {
        "requestId": "uuid",
        "requestType": "OOO",
        "leaveType": "ANNUAL_LEAVE",
        "startDate": "2026-05-01",
        "endDate": "2026-05-05",
        "status": "PENDING",
        "submittedAt": "2026-03-21T09:30:00Z"
      }
    ],
    "page": 0, "size": 20, "totalElements": 14, "totalPages": 1
  }
}

DELETE /api/v1/remote-requests/{requestId}
Response 200: { "success": true, "data": { "requestId": "...", "status": "CANCELLED" } }
Response 409: { "success": false, "error": { "code": "CANNOT_CANCEL_APPROVED", "message": "Use /cancel-approved to cancel an approved request." } }

POST /api/v1/remote-requests/{requestId}/cancel-approved
Response 200: { "success": true, "data": { "requestId": "...", "status": "CANCELLED", "attendanceReverted": true } }
Response 400: { "success": false, "error": { "code": "PAST_DATE_CANCEL", "message": "Cannot cancel an approved request for a past date." } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Own-records-only enforcement confirmed (another employee's request → 403)
- [ ] Attendance revert confirmed for both REMOTE and OOO cancellations
- [ ] `is_overridden = true` records not reverted — confirmed in test
- [ ] Past-date cancel guard confirmed (400 returned)
- [ ] SQS manager notification on all cancellations
- [ ] Audit log entries for all cancel actions
- [ ] Swagger annotations complete on all endpoints
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-23** and **AOMS-24** must be complete (entities exist).
- **AOMS-25** must be complete (`AttendancePreStampService` exists and has revert methods added here).
- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- SQS queue `oms-notifications-queue` provisioned.
