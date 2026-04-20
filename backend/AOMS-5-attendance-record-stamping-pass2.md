# AOMS-5 — Attendance Record Stamping — Pass 2: Leave, Remote & Holidays (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-5-attendance-record-stamping-pass2.md](attendance-service/AOMS-5-attendance-record-stamping-pass2.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-5 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job |
| **Perspective** | Backend |

---

## Story

As a system, I want to overlay approved leave, remote day, and public holiday records onto attendance data so that employees without badge events are not incorrectly marked as absent.

---

## Background (Backend Context)

Pass 2 runs immediately after Pass 1 in the nightly pipeline. It corrects `AttendanceRecord` statuses for employees who have no badge event because they were on approved leave, working remotely, or the office was closed for a public holiday. Without Pass 2, all employees on approved leave would be stamped `ABSENT` — which is incorrect.

Pass 2 only touches records with no existing `WorkSession` link. If a `WorkSession` was resolved in Pass 1, the record is authoritative and is not overridden by Pass 2 (e.g. an employee who worked remotely but still badged in keeps the badge-based status).

**Messaging note:** Pass 2 queries the `ooo_requests`, `remote_requests`, and `public_holidays` tables directly (co-located in this service's database, populated by the Remote/OOO service). No SQS dependency for the nightly job itself — SQS is used for notifications only.

**Pipeline:** Jenkins CI — lint → unit test → integration test → Docker build.

---

## Acceptance Criteria (Backend)

- [ ] `AttendancePass2Service.overlay(locationId, date)` runs after Pass 1 as part of the same nightly job orchestration.
- [ ] For each `user_id` at the location with no `AttendanceRecord.work_session_id` for the given date:
  - Query `ooo_requests` for an approved OOO covering that user and date → set `status = ON_LEAVE`; set `leave_request_id`.
  - Query `remote_requests` for an approved remote day for that user and date → set `status = REMOTE`; set `remote_request_id`.
  - Query `public_holidays` for the location and date → set `status = PUBLIC_HOLIDAY`.
  - Priority order when multiple match: `ON_LEAVE` > `PUBLIC_HOLIDAY` > `REMOTE` (OOO takes precedence).
- [ ] Pass 2 **only** updates records where `work_session_id IS NULL` — badge-based statuses from Pass 1 are never overwritten.
- [ ] `is_overridden = true` records are also skipped by Pass 2.
- [ ] After both passes, any `user_id` at the location with no `AttendanceRecord` for the given date (and on or after `employment_start_date`) is stamped `ABSENT` by a third micro-step within Pass 2.
- [ ] The ABSENT stamping micro-step generates `AttendanceRecord` rows with `work_session_id = null`, `leave_request_id = null`, `remote_request_id = null`.
- [ ] Pass 2 run is logged in `attendance_stamp_log` alongside Pass 1.
- [ ] Unit tests: ON_LEAVE overlay, REMOTE overlay, PUBLIC_HOLIDAY overlay, priority conflict (ON_LEAVE wins), ABSENT stamping, badge-based record not touched by Pass 2.
- [ ] Integration test: seed approved OOO, remote, and public holiday records; run both passes; assert correct final statuses across all scenarios.

---

## Implementation Notes

```
AttendancePass2Service.overlay(locationId, date)
    ├── Get all active users at location
    ├── For each user with no work_session_id record:
    │     ├── Check OOORequest (approved, date in range) → ON_LEAVE
    │     ├── Check PublicHoliday (location, date) → PUBLIC_HOLIDAY
    │     ├── Check RemoteRequest (approved, date) → REMOTE
    │     └── AttendanceRecordRepository.upsert() [skip if work_session_id set or is_overridden]
    └── ABSENT sweep: stamp remaining users with no record
```

- Fetch all relevant `OOORequest`, `RemoteRequest`, and `PublicHoliday` records for the location and date in three bulk queries — do not query per user to avoid N+1.
- The ABSENT sweep: `SELECT DISTINCT user_id FROM users WHERE location_id = ? AND employment_start_date <= ? AND deleted_at IS NULL` minus the set of users who already have an `AttendanceRecord` for that date → insert `ABSENT` records.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Priority conflict scenario tested
- [ ] Badge-based record protection confirmed
- [ ] ABSENT sweep tested
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** must be complete (Pass 1 must run first).
- `ooo_requests` and `remote_requests` tables must exist and be populated by the Remote/OOO service (marked as cross-service dependency; stub with test data for Sprint 1 integration tests).
- `public_holidays` table and CRUD (AOMS-11) must be available.
