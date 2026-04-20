# AOMS-7 — Manager Team Attendance View (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-7 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, api, manager, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a manager, I want to view the attendance records of my direct reports so that I can monitor team attendance patterns and identify concerns.

---

## Background (Service Context)

The manager view is scoped to direct reports at the manager's location. The `manager_id` field on the local `users` read-model (populated from the Arms HR sync via `auth-service` → `oms.user` SQS events) defines the direct-report relationship. The API must enforce: (1) the requesting user holds `MANAGER` role at the given location, (2) any `employee_id` filter must be a direct report of the requesting manager, (3) no data crosses location boundaries.

CSV export is generated server-side using `StreamingResponse` — rows are streamed from the async DB cursor directly to the response; the full result set is never buffered in memory.

---

## Acceptance Criteria

### Role Check
- [ ] All endpoints in this ticket require `MANAGER` role at `location_id`. Use a FastAPI dependency `require_role(Role.MANAGER)` that reads `current_user.roles` and raises HTTP 403 if not satisfied.
- [ ] `location_id` is extracted from the route path or query param, then cross-checked against `current_user.location_ids`.

### Team List Endpoint
- [ ] `GET /api/v1/attendance/team` returns paginated `AttendanceRecordResponse` (same schema as AOMS-6) for all direct reports of the authenticated manager at their location.
- [ ] Query params: `employee_id` (UUID, optional), `from_date` (date, optional), `to_date` (date, optional), `status` (list[AttendanceStatus], optional), `page`, `size`, `order`.
- [ ] Response per record includes: `employee_id`, `employee_name`, `record_date`, `status`, `first_badge_in`, `last_badge_out`, `total_duration_minutes`, `is_late`, `minutes_late`, `is_overridden`.
- [ ] If `employee_id` filter is provided: verify it is in the direct-report set before querying. Return HTTP 403 if the queried employee is not a direct report.
- [ ] Manager cannot see records for employees at other locations.

### Calendar Snapshot Endpoint
- [ ] `GET /api/v1/attendance/team/calendar?date=2026-03-14` returns one record per direct report showing their status for that date. Used by the React calendar grid.
- [ ] Response: `{ "date": "2026-03-14", "records": [{ "employeeId", "employeeName", "status", "isOverridden" }, ...] }`.
- [ ] Employees with no record for that date are included with `status: null`.

### CSV Export Endpoint
- [ ] `GET /api/v1/attendance/team/export` streams a CSV response.
- [ ] Same filter params as team list endpoint.
- [ ] Max 90-day window: if `to_date - from_date > 90 days`, return HTTP 400 with `{ "code": "EXPORT_WINDOW_TOO_LARGE", "message": "Export window cannot exceed 90 days." }`.
- [ ] Response headers: `Content-Type: text/csv; charset=utf-8`, `Content-Disposition: attachment; filename="team-attendance-{from_date}-{to_date}.csv"`.
- [ ] CSV columns: `Employee Name`, `Employee ID`, `Date`, `Status`, `First Badge In`, `Last Badge Out`, `Duration (mins)`, `Minutes Late`, `Overridden`.
- [ ] Streamed via `StreamingResponse` — no full in-memory buffer.

### Tests
- [ ] Unit tests: direct-report set enforcement (non-direct-report `employee_id` → 403); location boundary (records from another location not returned); 90-day export cap (91-day window → 400); calendar snapshot includes employees with null status.
- [ ] Integration test: seed 3 direct reports + 1 non-direct-report for a manager; manager queries team; verify only 3 direct reports returned; verify non-direct-report query returns 403; verify CSV row count matches filtered records.

---

## Implementation Notes

```python
router = APIRouter(prefix="/api/v1/attendance", tags=["attendance"])

@router.get("/team", response_model=ApiResponse[PaginatedResponse[TeamAttendanceRecordResponse]])
async def get_team_attendance(
    employee_id: UUID | None = Query(None),
    from_date: date | None = Query(None),
    to_date: date | None = Query(None),
    status: list[AttendanceStatus] = Query(default=[]),
    page: int = Query(0, ge=0),
    size: int = Query(20, ge=1, le=100),
    current_user: UserContext = Depends(get_current_user),
    _: None = Depends(require_role(Role.MANAGER)),
    db: AsyncSession = Depends(get_db),
):
    direct_report_ids = await user_repo.get_direct_report_ids(
        manager_id=current_user.user_id,
        location_id=current_user.primary_location_id,
        db=db,
    )
    if employee_id and employee_id not in direct_report_ids:
        raise HTTPException(status_code=403, detail="Employee is not a direct report.")

    target_ids = [employee_id] if employee_id else list(direct_report_ids)
    records, total = await record_repo.find_for_users(target_ids, from_date, to_date, status, page, size, db)
    return ApiResponse.ok(PaginatedResponse.of(records, total, page, size))
```

CSV streaming:
```python
@router.get("/team/export")
async def export_team_attendance(
    from_date: date = Query(...),
    to_date: date = Query(...),
    current_user: UserContext = Depends(get_current_user),
    _: None = Depends(require_role(Role.MANAGER)),
    db: AsyncSession = Depends(get_db),
):
    if (to_date - from_date).days > 90:
        raise HTTPException(status_code=400, detail={"code": "EXPORT_WINDOW_TOO_LARGE"})

    direct_report_ids = await user_repo.get_direct_report_ids(current_user.user_id, current_user.primary_location_id, db)

    async def csv_generator():
        yield "Employee Name,Employee ID,Date,Status,First Badge In,Last Badge Out,Duration (mins),Minutes Late,Overridden\n"
        async for record in record_repo.stream_for_users(direct_report_ids, from_date, to_date, db):
            yield f"{record.employee_name},{record.user_id},{record.record_date},{record.status}," \
                  f"{record.first_badge_in or ''},{record.last_badge_out or ''}," \
                  f"{record.total_duration_minutes or ''},{record.minutes_late or ''},{record.is_overridden}\n"

    return StreamingResponse(
        csv_generator(),
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename=team-attendance-{from_date}-{to_date}.csv"},
    )
```

DB streaming uses `stream_results=True` on the async session to avoid loading all rows at once:
```python
result = await db.stream(stmt)  # AsyncResult supports async iteration
async for row in result:
    yield row
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Direct-report scoping enforced; non-direct-report query returns 403 (tested)
- [ ] Location boundary enforced; cross-location records never returned (tested)
- [ ] CSV export streams — no full in-memory load (verified in integration test with large dataset)
- [ ] 90-day export window cap enforced
- [ ] OpenAPI docs complete for all three endpoints
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete.
- `users` read-model in `attendance_db` must include `manager_id` and `primary_location_id` (populated via `oms.user.created`/`updated` SQS consumer from AOMS-5 setup).
