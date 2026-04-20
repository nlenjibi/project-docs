# AOMS-11 — Public Holiday Management (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-11-public-holiday-management.md](attendance-service/AOMS-11-public-holiday-management.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-11 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, config, admin |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want to configure public holidays for my office location so that employees are not marked absent on days the office is closed.

---

## Background (Backend Context)

`PublicHoliday` records are consumed by the nightly attendance Pass 2 job (AOMS-5) to overlay `PUBLIC_HOLIDAY` status on employee records for closed days. This means public holidays must be available in the database **before** the nightly sync runs for those dates — admins are expected to configure them in advance.

Deleting a past holiday is a destructive act that creates incorrect historical records — it requires a re-stamp of all affected `AttendanceRecord` rows for that date. This re-stamp is a Super Admin-only operation and is clearly separated from the normal delete flow.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `PublicHoliday` entity: `id` (UUID), `location_id`, `holiday_date` (Date), `name` (String), `created_at`, `updated_at`, `created_by`. No `deleted_at` — soft delete is not used; hard deletes are allowed but trigger a re-stamp check.
- [ ] Unique constraint on `(location_id, holiday_date)` — one holiday record per location per date.
- [ ] `POST /api/v1/locations/{locationId}/public-holidays` — creates a holiday. Accessible to Facilities Admin and Super Admin.
  - Request: `{ "holidayDate": "2026-12-25", "name": "Christmas Day" }`.
  - `holidayDate` must not be in the past (unless actor is Super Admin).
  - Returns 409 if a holiday already exists for that date at that location.
- [ ] `GET /api/v1/locations/{locationId}/public-holidays` — returns full list of holidays for the location, sorted ascending by date. Accessible to all authenticated roles (read-only).
  - Supports optional `year` query param: `?year=2026`.
- [ ] `PUT /api/v1/locations/{locationId}/public-holidays/{holidayId}` — updates the `name` or `holidayDate` of a future holiday. Accessible to Facilities Admin and Super Admin. Past holidays cannot be edited (return 400).
- [ ] `DELETE /api/v1/locations/{locationId}/public-holidays/{holidayId}` — accessible to Facilities Admin and Super Admin.
  - If the holiday date is in the **future**: delete immediately with no re-stamp required.
  - If the holiday date is in the **past**: only Super Admin may delete; the delete is executed but a re-stamp job is enqueued via SQS (`oms-attendance-restamp-queue`) with `{ "locationId": "...", "date": "2026-01-01" }`. The re-stamp job re-runs Pass 2 for that date.
- [ ] All create, update, and delete operations are written to the `AuditLog`.
- [ ] Unit tests: create holiday, duplicate date (409), past date guard, delete future holiday, delete past holiday triggers SQS message.
- [ ] Integration test: create a holiday; run nightly Pass 2 for that date; verify employees get `PUBLIC_HOLIDAY` status.

---

## Implementation Notes

```
POST /api/v1/locations/{locationId}/public-holidays
    ↓
@RequiresRole({FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership check
    ↓
PublicHolidayService.create(locationId, holidayDate, name, actorId)
    ├── Validate: no duplicate date at location
    ├── Validate: not in past (unless Super Admin)
    ├── PublicHolidayRepository.save()
    └── AuditLogService.log(...)

DELETE /api/v1/locations/{locationId}/public-holidays/{holidayId}
    ↓
PublicHolidayService.delete(holidayId, actorId)
    ├── If past date: guard Super Admin role
    ├── PublicHolidayRepository.delete()
    ├── If past date: SqsPublisher.send("oms-attendance-restamp-queue", { locationId, date })
    └── AuditLogService.log(...)
```

- The re-stamp SQS consumer (a separate `AttendanceRestampListener`) calls `AttendancePass2Service.overlay(locationId, date)` for the affected date. This listener is scoped to this ticket — implement it here.
- `oms-attendance-restamp-queue` URL configured via `ATTENDANCE_RESTAMP_SQS_QUEUE_URL` env var.

---

## API Contract

```
POST /api/v1/locations/{locationId}/public-holidays
Request:  { "holidayDate": "2026-12-25", "name": "Christmas Day" }
Response 201: { "success": true, "data": { "id": "uuid", "holidayDate": "2026-12-25", "name": "Christmas Day" } }

GET /api/v1/locations/{locationId}/public-holidays?year=2026
Response 200: { "success": true, "data": [ { "id", "holidayDate", "name" }, ... ] }

DELETE /api/v1/locations/{locationId}/public-holidays/{holidayId}
Response 200: { "success": true, "data": { "restampQueued": true } }  ← past date
Response 200: { "success": true, "data": { "restampQueued": false } } ← future date
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Past-date delete guard enforced (Facilities Admin blocked; Super Admin proceeds with re-stamp)
- [ ] Re-stamp SQS message published and listener implemented
- [ ] Audit log entries for all mutations
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-5** must be complete (Pass 2 uses `PublicHoliday` table).
- **ATT-P03** must be complete.
- SQS queue `oms-attendance-restamp-queue` provisioned.
