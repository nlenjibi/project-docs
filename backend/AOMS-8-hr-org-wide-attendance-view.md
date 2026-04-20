# AOMS-8 — HR Org-Wide Attendance View (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-8-hr-org-wide-attendance-view.md](attendance-service/AOMS-8-hr-org-wide-attendance-view.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-8 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, api, hr |
| **Perspective** | Backend |

---

## Story

As an HR team member, I want to view attendance records across all employees at my location so that I can monitor org-wide attendance and generate reports.

---

## Background (Backend Context)

The HR view is the broadest read scope in the attendance domain — all employees at the HR user's location. The key distinction from the manager view (AOMS-7) is that HR sees everyone, not just direct reports. However, HR is still **location-scoped** — they cannot see records for other locations. Only Super Admin crosses location boundaries.

The aggregate statistics endpoint (`/summary`) is a separate query — it is used by the React dashboard header row showing totals per day and should be cheap to compute.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `GET /api/v1/attendance/location/{locationId}` returns paginated `AttendanceRecord` records for all active employees at the given location.
- [ ] Requesting user must hold HR or Super Admin role at `locationId` — return 403 otherwise.
- [ ] Filterable by: `departmentId`, `managerId` (show all reports under a manager subtree), `employeeId`, `from` (date), `to` (date), `status` (multi-value).
- [ ] Response per record includes: `employeeId`, `employeeName`, `department`, `recordDate`, `status`, `firstBadgeIn`, `lastBadgeOut`, `totalDurationMinutes`, `isLate`, `minutesLate`, `isOverridden`.
- [ ] `GET /api/v1/attendance/location/{locationId}/summary?date=2026-03-14` returns aggregate counts for a single date: `{ date, totalPresent, totalLate, totalAbsent, totalRemote, totalOnLeave, totalPublicHoliday, totalInsufficientHours }`.
- [ ] `GET /api/v1/attendance/location/{locationId}/export` streams a CSV download.
  - Filterable by same params as the list endpoint.
  - Max 90-day date window per export.
  - Response headers: `Content-Type: text/csv`, `Content-Disposition: attachment; filename="attendance-{locationId}-{from}-{to}.csv"`.
  - CSV columns: `Employee Name`, `Employee ID`, `Department`, `Date`, `Status`, `First Badge In`, `Last Badge Out`, `Duration (mins)`, `Minutes Late`, `Overridden`.
- [ ] HR user cannot access records for `locationId` where they do not hold an HR role. Super Admin may access any `locationId`.
- [ ] Unit tests: location boundary (HR at Accra cannot fetch Lagos records), department filter, summary aggregation, export row count.
- [ ] Integration test: HR user fetches all records for location; verifies totals match seeded data.

---

## Implementation Notes

```
GET /api/v1/attendance/location/{locationId}
    ↓
@RequiresRole({HR, SUPER_ADMIN}) + location ownership check
    ↓
AttendanceService.getLocationAttendance(locationId, filters, pageable)
    ↓
AttendanceRecordRepository.findByLocationAndFilters(locationId, filters, pageable)
```

- The `departmentId` filter joins to `users.department`; the `managerId` filter resolves the full reporting subtree (recursive CTE on `users.manager_id`) — be cautious with deep hierarchies; add a max depth of 5 for safety.
- Summary query: a single GROUP BY query on `attendance_records` for the given `locationId` and `date` — do not compute in Java.
- CSV export: stream via `StreamingResponseBody`; do not buffer the full result set in memory.

---

## API Contract

```
GET /api/v1/attendance/location/{locationId}?from=&to=&status=ABSENT&department=Engineering&page=0&size=20

GET /api/v1/attendance/location/{locationId}/summary?date=2026-03-14
Response:
{
  "success": true,
  "data": {
    "date": "2026-03-14",
    "totalPresent": 42, "totalLate": 5, "totalAbsent": 3,
    "totalRemote": 8, "totalOnLeave": 2, "totalPublicHoliday": 0,
    "totalInsufficientHours": 1
  },
  "error": null
}

GET /api/v1/attendance/location/{locationId}/export?from=2026-03-01&to=2026-03-31
→ 200 CSV stream
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Location boundary enforced; HR at one location cannot read another
- [ ] Summary query is a single DB aggregation — no in-app looping
- [ ] CSV export streams; no full in-memory buffer
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied (HR, Super Admin)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete.
- **ATT-P01**, **ATT-P02** must be complete.
- `User.department` field populated from Arms HR sync.
