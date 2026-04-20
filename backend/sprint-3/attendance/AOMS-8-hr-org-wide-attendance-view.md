# AOMS-8 — HR Org-Wide Attendance View (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-8 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, api, hr, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As an HR team member, I want to view attendance records across all employees at my location so that I can monitor org-wide attendance and generate reports.

---

## Background (Service Context)

HR sees all employees at their location — not just direct reports. `SUPER_ADMIN` can access any `location_id`. HR is still location-scoped: an HR user at Accra cannot query Lagos records.

The daily summary endpoint (`/summary`) is used by the React dashboard header — it must be a single SQL `GROUP BY` query, not computed in Python.

The `managerId` filter uses a **recursive CTE** to walk the reporting hierarchy (manager → direct reports → their reports, etc.) up to a max depth of 5 levels, avoiding unbounded recursion on edge-case data.

---

## Acceptance Criteria

### Role Check
- [ ] All endpoints require `HR` or `SUPER_ADMIN` role. Use `require_any_role([Role.HR, Role.SUPER_ADMIN])` dependency.
- [ ] HR user can only access `location_id` values present in their `location_ids` JWT claim.
- [ ] `SUPER_ADMIN` (identified by `location_ids = []` or a special `SUPER_ADMIN` flag in JWT) can access any `location_id`.

### Location List Endpoint
- [ ] `GET /api/v1/attendance/location/{location_id}` returns paginated `AttendanceRecordResponse` for all active employees at the given location.
- [ ] Query params: `department_id` (str, optional), `manager_id` (UUID, optional — resolves full subtree), `employee_id` (UUID, optional), `from_date`, `to_date`, `status` (multi-value), `page`, `size`, `order`.
- [ ] `manager_id` filter resolves the full reporting subtree via recursive CTE (max depth 5).
- [ ] Response per record: `employee_id`, `employee_name`, `department`, `record_date`, `status`, `first_badge_in`, `last_badge_out`, `total_duration_minutes`, `is_late`, `minutes_late`, `is_overridden`.

### Summary Endpoint
- [ ] `GET /api/v1/attendance/location/{location_id}/summary?date=2026-03-14` returns aggregate counts for a single date.
- [ ] Response: `{ "date", "total_present", "total_late", "total_absent", "total_remote", "total_on_leave", "total_public_holiday", "total_insufficient_hours" }`.
- [ ] Computed via a **single SQL GROUP BY query** — not in Python. No N+1.

### CSV Export Endpoint
- [ ] `GET /api/v1/attendance/location/{location_id}/export` streams a CSV.
- [ ] Same filter params as list endpoint. Max 90-day window.
- [ ] CSV columns: `Employee Name`, `Employee ID`, `Department`, `Date`, `Status`, `First Badge In`, `Last Badge Out`, `Duration (mins)`, `Minutes Late`, `Overridden`.
- [ ] Response headers: `Content-Type: text/csv`, `Content-Disposition: attachment; filename="attendance-{location_id}-{from_date}-{to_date}.csv"`.
- [ ] `StreamingResponse` — no in-memory buffer.

### Tests
- [ ] Unit tests: location boundary (HR at location A cannot fetch location B → 403); department filter applied; summary aggregation returns correct counts; 90-day export cap.
- [ ] Integration test: HR user fetches all records for location; verifies total count matches seeded data; verifies summary counts; verifies CSV row count.
- [ ] Integration test: `manager_id` filter with 2-level hierarchy returns all employees in subtree.

---

## Implementation Notes

Recursive CTE for manager subtree (SQLAlchemy Core):
```python
from sqlalchemy import text

async def get_subtree_user_ids(manager_id: UUID, db: AsyncSession, max_depth: int = 5) -> list[UUID]:
    result = await db.execute(
        text("""
            WITH RECURSIVE reports AS (
                SELECT id, manager_id, 1 AS depth
                FROM users
                WHERE manager_id = :manager_id AND deleted_at IS NULL
                UNION ALL
                SELECT u.id, u.manager_id, r.depth + 1
                FROM users u
                JOIN reports r ON u.manager_id = r.id
                WHERE u.deleted_at IS NULL AND r.depth < :max_depth
            )
            SELECT id FROM reports
        """),
        {"manager_id": str(manager_id), "max_depth": max_depth},
    )
    return [row.id for row in result]
```

Summary aggregation query:
```python
stmt = (
    select(
        AttendanceRecord.status,
        func.count(AttendanceRecord.id).label("count"),
    )
    .where(
        AttendanceRecord.location_id == location_id,
        AttendanceRecord.record_date == target_date,
        AttendanceRecord.deleted_at.is_(None),
    )
    .group_by(AttendanceRecord.status)
)
```

---

## API Contract

```
GET /api/v1/attendance/location/{location_id}/summary?date=2026-03-14

Response 200:
{
  "success": true,
  "data": {
    "date": "2026-03-14",
    "totalPresent": 42,
    "totalLate": 5,
    "totalAbsent": 3,
    "totalRemote": 8,
    "totalOnLeave": 2,
    "totalPublicHoliday": 0,
    "totalInsufficientHours": 1
  },
  "error": null
}
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Location boundary enforced; HR at one location cannot read another (tested)
- [ ] Summary query is a single SQL aggregation — verified via query logging in tests
- [ ] Recursive CTE depth cap (5) prevents runaway queries
- [ ] CSV export streams; no full in-memory buffer (tested)
- [ ] 90-day export window cap enforced
- [ ] OpenAPI docs complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete.
- `users` read-model must include `department`, `manager_id`, `primary_location_id`.
