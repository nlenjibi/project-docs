# AOMS-6 — Employee Attendance Self-View (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-6 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, api, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As an employee, I want to view my own attendance history so that I can verify my records and understand my attendance status on any given day.

---

## Background (Service Context)

This ticket delivers the read API for the React self-view screen. The employee sees their own records only — `user_id` is always sourced from the validated internal JWT, never from a query parameter. The API enforces the `employment_start_date` lower bound and returns `WorkSession` summary fields via a JOIN — raw `BadgeEvent` data is never exposed.

The internal JWT (injected by AWS API Gateway) is validated via a FastAPI `Depends()` security dependency. The decoded payload provides `user_id`, `roles`, and `location_ids` — no secondary call to `auth-service` is made per request.

---

## Acceptance Criteria

### Pydantic Schemas
- [ ] `AttendanceRecordResponse`: `id` (UUID), `record_date` (date), `status` (str), `first_badge_in` (datetime | None), `last_badge_out` (datetime | None), `total_duration_minutes` (int | None), `is_late` (bool), `minutes_late` (int | None), `is_overridden` (bool). `model_config = ConfigDict(from_attributes=True)`.
- [ ] `AttendanceRecordDetailResponse`: extends above with `work_session_id` (UUID | None), `leave_request_id` (UUID | None), `remote_request_id` (UUID | None).
- [ ] `PaginatedResponse[T]`: generic wrapper — `content: list[T]`, `page: int`, `size: int`, `total_elements: int`, `total_pages: int`.
- [ ] Outer envelope: `ApiResponse[T]` — `success: bool`, `data: T | None`, `error: ErrorDetail | None`.

### List Endpoint
- [ ] `GET /api/v1/attendance/my` returns paginated `AttendanceRecordResponse` list for the authenticated employee.
- [ ] Query parameters: `from_date` (date, optional), `to_date` (date, optional), `status` (list[AttendanceStatus], optional, multi-value via `Query(...)`), `page` (int ≥ 0, default 0), `size` (int 1–100, default 20), `sort` (Literal["record_date"], default "record_date"), `order` (Literal["asc", "desc"], default "desc").
- [ ] Lower bound enforced: `record_date >= user.employment_start_date`. If `from_date` is provided and is earlier than `employment_start_date`, the effective lower bound is `employment_start_date`.
- [ ] `user_id` is always from `current_user.user_id` (decoded JWT) — query parameter override is not accepted.
- [ ] Empty result returns HTTP 200 with `data.content = []` — not 404.
- [ ] `WorkSession` fields (`first_badge_in`, `last_badge_out`, etc.) are populated via a LEFT JOIN on `work_sessions.id = attendance_records.work_session_id`. Fields are `null` when `work_session_id` is null.

### Detail Endpoint
- [ ] `GET /api/v1/attendance/my/{record_id}` returns `AttendanceRecordDetailResponse`.
- [ ] Returns 404 if `record_id` does not exist **or** does not belong to the authenticated `user_id`. Never reveal that a record exists but belongs to someone else.

### Tests
- [ ] Unit tests (mock DB): employment start date lower bound applied; `status` multi-filter applied; wrong `user_id` on detail → 404; unknown `record_id` → 404.
- [ ] Integration test: seed 10 `AttendanceRecord` rows for two users; authenticated as user A; assert only user A's records returned; assert pagination correct; assert `status` filter narrows results.

---

## Implementation Notes

```python
router = APIRouter(prefix="/api/v1/attendance", tags=["attendance"])

@router.get("/my", response_model=ApiResponse[PaginatedResponse[AttendanceRecordResponse]])
async def get_my_attendance(
    from_date: date | None = Query(None),
    to_date: date | None = Query(None),
    status: list[AttendanceStatus] = Query(default=[]),
    page: int = Query(0, ge=0),
    size: int = Query(20, ge=1, le=100),
    order: Literal["asc", "desc"] = Query("desc"),
    current_user: UserContext = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> ApiResponse:
    user = await user_repo.get(current_user.user_id, db)
    effective_from = max(from_date or date.min, user.employment_start_date)
    records, total = await record_repo.find_my(
        user_id=current_user.user_id,
        from_date=effective_from,
        to_date=to_date,
        statuses=status,
        page=page,
        size=size,
        order=order,
        db=db,
    )
    return ApiResponse.ok(PaginatedResponse.of(records, total, page, size))
```

Repository query with LEFT JOIN:
```python
stmt = (
    select(AttendanceRecord, WorkSession)
    .outerjoin(WorkSession, AttendanceRecord.work_session_id == WorkSession.id)
    .where(
        AttendanceRecord.user_id == user_id,
        AttendanceRecord.record_date >= from_date,
        AttendanceRecord.deleted_at.is_(None),
    )
    .order_by(desc(AttendanceRecord.record_date) if order == "desc" else asc(AttendanceRecord.record_date))
    .offset(page * size)
    .limit(size)
)
```

---

## API Contract

```
GET /api/v1/attendance/my?from_date=2026-01-01&to_date=2026-03-31&status=LATE&status=ABSENT&page=0&size=20

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
- [ ] Employee cannot access another employee's records (tested)
- [ ] Raw `BadgeEvent` data not exposed in any response field
- [ ] FastAPI auto-generated OpenAPI docs include query param descriptions and response examples
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete — `AttendanceRecord` rows must exist.
- Internal JWT validation dependency (`get_current_user`) must be implemented (shared module, done in AOMS-2 setup).
