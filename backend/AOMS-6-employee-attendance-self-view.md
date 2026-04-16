# AOMS-6 — Employee Attendance Self-View (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-6 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, api |
| **Perspective** | Backend |

---

## Story

As an employee, I want to view my own attendance history so that I can verify my records and understand my attendance status on any given day.

---

## Background (Backend Context)

This ticket delivers the REST API that backs the React self-view screen. The employee sees their own records only — no access to any other employee's data. The API must enforce the `employment_start_date` boundary, support date-range and status filtering, and return paginated results. All authorisation is enforced server-side.

**API Gateway:** Spring Cloud Gateway (dev) / AWS or Kong (prod) routes `/api/v1/attendance/**` to this service. The interceptor (ATT-P02) ensures `userId` in the request matches the authenticated user.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `GET /api/v1/attendance/my` returns a paginated list of `AttendanceRecord` records for the authenticated employee.
- [ ] Response shape per record: `{ id, recordDate, status, firstBadgeIn, lastBadgeOut, totalDurationMinutes, isLate, minutesLate, isOverridden }`.
- [ ] Query only returns records from `employment_start_date` to today — no earlier records are returned even if they exist.
- [ ] Supports query params: `from` (date), `to` (date), `status` (Enum, multi-value), `page`, `size`, `sort`.
- [ ] Default sort: `recordDate DESC`.
- [ ] An employee cannot query another employee's records — the `userId` is always sourced from the session context, not from a query parameter.
- [ ] Empty result set (no records yet) returns HTTP 200 with empty `data.content` array — not 404.
- [ ] `GET /api/v1/attendance/my/{recordId}` returns a single record detail including: all fields above plus `workSessionId`, `leaveRequestId`, `remoteRequestId`.
- [ ] Returns 404 if `recordId` does not belong to the authenticated user.
- [ ] All responses use the standard `ApiResponse<T>` envelope.
- [ ] Response does not include raw `BadgeEvent` data — only derived `WorkSession` summary fields.
- [ ] Unit tests: date boundary enforcement, status filter, ownership check (other user's record returns 404).
- [ ] Integration test: authenticated employee fetches own records, verifies pagination and filtering.

---

## Implementation Notes

```
GET /api/v1/attendance/my
    ↓
AttendanceController.getMyAttendance(principal, filters, pageable)
    ↓
AttendanceService.getForEmployee(userId, locationId, filters, pageable)
    ↓
AttendanceRecordRepository.findByUserIdAndLocationIdAndRecordDateBetween(...)
```

- The `userId` is **always** extracted from `SecurityContextHolder` — never trusted from a query parameter.
- `employment_start_date` check: add it as a lower bound on the `recordDate` filter: `recordDate >= user.employment_start_date`.
- `WorkSession` fields (`firstBadgeIn`, `lastBadgeOut`, `totalDurationMinutes`, `isLate`, `minutesLate`) are resolved via a JOIN on `work_sessions.id = attendance_records.work_session_id`. If `work_session_id` is null, these fields return null in the response.

---

## API Contract

```
GET /api/v1/attendance/my?from=2026-01-01&to=2026-03-31&status=LATE,ABSENT&page=0&size=20

Response 200:
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid",
        "recordDate": "2026-03-01",
        "status": "LATE",
        "firstBadgeIn": "2026-03-01T09:22:00Z",
        "lastBadgeOut": "2026-03-01T17:45:00Z",
        "totalDurationMinutes": 503,
        "isLate": true,
        "minutesLate": 22,
        "isOverridden": false
      }
    ],
    "page": 0, "size": 20, "totalElements": 45, "totalPages": 3
  },
  "error": null
}
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Role + location interceptor applied; employee cannot access other employees' records
- [ ] Swagger annotations complete (`@Operation`, `@ApiResponse`, `@Parameter`)
- [ ] No raw `BadgeEvent` data exposed
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete (AttendanceRecords must exist).
- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
