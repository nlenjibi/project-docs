# AOMS-12 — No-Show Reporting for Admin (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-12-no-show-reporting.md](attendance-service/AOMS-12-no-show-reporting.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-12 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, reporting, admin |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want to view a report of seat booking no-shows so that I can identify patterns and take action where necessary.

---

## Background (Backend Context)

`NoShowRecord` entities are created by the no-show auto-release job (AOMS-18). This ticket delivers the query API that surfaces those records as a filterable, exportable report for Facilities Admin. The report is read-only — no mutations. Manager and Employee roles must not see this data.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `GET /api/v1/seat-bookings/no-shows` returns a paginated list of `NoShowRecord` entries for the requesting user's location.
- [ ] Accessible to Facilities Admin and Super Admin only — return 403 for all other roles.
- [ ] Filterable by: `from` (date), `to` (date), `employeeId`, `departmentId`.
- [ ] Response per record: `{ noShowRecordId, employeeId, employeeName, department, bookingDate, seatReference (floor + zone + seat number), autoReleasedAt }`.
- [ ] An aggregate field is surfaced per employee in the response: `noShowCountInPeriod` — the number of no-shows for that employee within the filtered date range.
- [ ] `GET /api/v1/seat-bookings/no-shows/export` streams a CSV download with the same filter params.
  - CSV columns: `Employee Name`, `Employee ID`, `Department`, `Booking Date`, `Seat Reference`, `Auto Released At`, `No-Show Count (Period)`.
  - Max 90-day window per export.
  - Response headers: `Content-Type: text/csv`, `Content-Disposition: attachment; filename="no-show-report-{from}-{to}.csv"`.
- [ ] Manager and Employee roles cannot access either endpoint — return 403.
- [ ] Unit tests: role guard (Manager → 403, Employee → 403, Facilities Admin → 200), department filter, no-show count aggregation.
- [ ] Integration test: seed `NoShowRecord` data; Facilities Admin fetches report; verifies counts and CSV row count.

---

## Implementation Notes

```
GET /api/v1/seat-bookings/no-shows
    ↓
@RequiresRole({FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership check
    ↓
NoShowReportService.getReport(locationId, filters, pageable)
    ├── NoShowRecordRepository.findByLocationAndFilters(...)
    ├── For each record: join to seat (floor/zone/seat) for seatReference
    ├── Compute noShowCountInPeriod per employee (subquery or GROUP BY)
    └── Map to NoShowReportResponseDto
```

- `seatReference` is a derived string: `"{floorName} / {zoneName} / Seat {seatNumber}"` — computed at the service layer by joining through `NoShowRecord → SeatBooking → Seat → Zone → Floor`.
- `noShowCountInPeriod` can be computed as a subquery in the main repository query (avoid N+1): `COUNT(*) WHERE user_id = ? AND booking_date BETWEEN :from AND :to GROUP BY user_id`.
- CSV export: stream via `StreamingResponseBody`.

---

## API Contract

```
GET /api/v1/seat-bookings/no-shows?from=2026-03-01&to=2026-03-31&page=0&size=20

Response 200:
{
  "success": true,
  "data": {
    "content": [
      {
        "noShowRecordId": "uuid",
        "employeeId": "uuid",
        "employeeName": "Jane Doe",
        "department": "Engineering",
        "bookingDate": "2026-03-12",
        "seatReference": "Floor 2 / Zone A / Seat 14",
        "autoReleasedAt": "2026-03-12T10:30:00Z",
        "noShowCountInPeriod": 3
      }
    ],
    "page": 0, "size": 20, "totalElements": 12, "totalPages": 1
  },
  "error": null
}

GET /api/v1/seat-bookings/no-shows/export?from=2026-03-01&to=2026-03-31
→ 200 CSV stream
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Role guard confirmed: Manager and Employee return 403
- [ ] `noShowCountInPeriod` computed via DB aggregation (not in-app loop)
- [ ] CSV streams; no in-memory buffer
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-18** (No-Show Auto-Release) must be complete — `NoShowRecord` records must exist.
- **ATT-P01**, **ATT-P02** must be complete.
- Seat hierarchy (`Floor`, `Zone`, `Seat`) entities must exist (AOMS-14).
