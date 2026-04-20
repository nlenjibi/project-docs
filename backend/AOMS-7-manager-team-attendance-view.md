# AOMS-7 — Manager Team Attendance View (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-7-manager-team-attendance-view.md](attendance-service/AOMS-7-manager-team-attendance-view.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-7 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, api, manager |
| **Perspective** | Backend |

---

## Story

As a manager, I want to view the attendance records of my direct reports so that I can monitor team attendance patterns and identify concerns.

---

## Background (Backend Context)

The manager view is scoped strictly to direct reports at the manager's location. The `manager_id` field on the `User` entity (populated from the Arms HR sync) defines the direct report relationship. The API must enforce: (1) the requesting user holds the MANAGER role at the given location, (2) the queried employees are direct reports of the requesting manager, (3) no data leaks across locations.

The CSV export is generated server-side and streamed as a file download — the React frontend does not assemble it.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `GET /api/v1/attendance/team` returns paginated `AttendanceRecord` records for all direct reports of the authenticated manager at their location.
- [ ] Filterable by: `employeeId` (must be a direct report), `from` (date), `to` (date), `status` (Enum, multi-value).
- [ ] Response per record includes: `employeeId`, `employeeName`, `recordDate`, `status`, `firstBadgeIn`, `lastBadgeOut`, `totalDurationMinutes`, `isLate`, `minutesLate`, `isOverridden`.
- [ ] Manager cannot query records for employees who are not their direct reports — returns 403 if attempted.
- [ ] Manager cannot query records for employees at other locations — enforced by location scoping.
- [ ] `GET /api/v1/attendance/team/calendar?date=2026-03-01` returns a snapshot: one record per direct report showing their status for the given date. Used by the React calendar grid view.
- [ ] `GET /api/v1/attendance/team/export?from=&to=&employeeId=` streams a CSV response with headers: `Employee Name`, `Employee ID`, `Date`, `Status`, `First Badge In`, `Last Badge Out`, `Duration (mins)`, `Minutes Late`.
  - Response headers: `Content-Type: text/csv`, `Content-Disposition: attachment; filename="team-attendance-{from}-{to}.csv"`.
  - Export is bounded — max 90-day window per export request to prevent unbounded queries.
- [ ] All requests enforce role = MANAGER at the location in scope.
- [ ] Unit tests: direct-report scoping, location boundary, 90-day export cap, calendar snapshot.
- [ ] Integration test: manager fetches team records, verifies scoping; non-direct-report query returns 403.

---

## Implementation Notes

```
GET /api/v1/attendance/team
    ↓
AttendanceController (MANAGER role required via @RequiresRole)
    ↓
AttendanceService.getTeamAttendance(managerId, locationId, filters, pageable)
    ↓
UserRepository.findDirectReports(managerId, locationId)  → List<UUID>
AttendanceRecordRepository.findByUserIdsAndLocationAndDateRange(...)
```

- Direct reports are resolved via `SELECT id FROM users WHERE manager_id = :managerId AND primary_location_id = :locationId AND deleted_at IS NULL`.
- If `employeeId` filter is provided, verify it is in the direct-report set before querying — do not rely on the DB constraint alone.
- CSV export: use `org.apache.commons.csv` or a simple `BufferedWriter` — do not load all records into memory before writing. Stream rows directly from the `ResultSet` to the response output stream using a `StreamingResponseBody`.

---

## API Contract

```
GET /api/v1/attendance/team?from=2026-03-01&to=2026-03-31&status=ABSENT&page=0&size=20

GET /api/v1/attendance/team/calendar?date=2026-03-14

GET /api/v1/attendance/team/export?from=2026-03-01&to=2026-03-31
→ 200 CSV download
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Direct-report scoping enforced; non-direct-report query returns 403
- [ ] Location boundary enforced
- [ ] CSV export streams (no full in-memory load)
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied (MANAGER role)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete.
- **ATT-P01**, **ATT-P02** must be complete.
- `User.manager_id` populated from Arms HR sync.
