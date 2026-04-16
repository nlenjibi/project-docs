# AOMS-4 — Attendance Record Stamping — Pass 1: Badge Events (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-4 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job |
| **Perspective** | Backend |

---

## Story

As a system, I want to create AttendanceRecords from resolved WorkSessions so that each employee has an official attendance status for each working day.

---

## Background (Backend Context)

Pass 1 is the first of a two-pass nightly stamping process. It runs immediately after `WorkSessionResolutionJob` (AOMS-3) and translates `WorkSession` records into final `AttendanceRecord` statuses based on configurable thresholds. Pass 2 (AOMS-5) overlays leave, remote, and holiday statuses on top — Pass 1 only concerns itself with badge-based statuses.

The pipeline is: **AOMS-2 (ingest) → AOMS-3 (resolve sessions) → AOMS-4 (Pass 1 stamp) → AOMS-5 (Pass 2 overlay)**.

**Pipeline:** Jenkins CI — lint → unit test → integration test → Docker build.

---

## Acceptance Criteria (Backend)

- [ ] `AttendanceRecord` entity created with fields: `id` (UUID), `user_id`, `location_id`, `record_date` (Date), `status` (Enum: `PRESENT`, `LATE`, `ABSENT`, `INSUFFICIENT_HOURS`, `REMOTE`, `ON_LEAVE`, `PUBLIC_HOLIDAY`), `work_session_id` (FK → WorkSession, nullable), `leave_request_id` (UUID, nullable), `remote_request_id` (UUID, nullable), `is_overridden` (Boolean, default false), `override_reason` (String, nullable), `original_status` (Enum, nullable), `created_at`, `updated_at`. Soft-delete with `deleted_at`.
- [ ] Flyway migration creates `attendance_records` table with unique constraint on `(user_id, record_date, location_id)`.
- [ ] Pass 1 logic (`AttendancePass1Service`):
  - For each `WorkSession` resolved in the current night's run:
    - If `total_duration_minutes >= LocationConfig.min_presence_duration_minutes` AND `is_late = false` → `status = PRESENT`.
    - If `total_duration_minutes >= min_presence_duration_minutes` AND `is_late = true` → `status = LATE`.
    - If `total_duration_minutes < min_presence_duration_minutes` (or `total_duration_minutes = null`) → `status = INSUFFICIENT_HOURS`.
  - `work_session_id` is set on the `AttendanceRecord`.
- [ ] Records are only created for dates **on or after** the employee's `User.employment_start_date`. Dates before this are silently skipped.
- [ ] Exactly one `AttendanceRecord` per `(user_id, record_date, location_id)` — the unique constraint enforces this; the service uses upsert logic.
- [ ] Re-running Pass 1 for the same date updates the existing record (idempotent upsert) — does not create duplicates.
- [ ] `AttendanceRecord` records with `is_overridden = true` are **never** updated by the nightly job — overridden records are protected.
- [ ] Pass 1 run is logged in `attendance_stamp_log`: location, date, records processed, records created, records updated, errors.
- [ ] Unit tests: PRESENT, LATE, INSUFFICIENT_HOURS paths; employment start date boundary; idempotent re-run; override protection.
- [ ] Integration test: seed `WorkSession` records; run Pass 1; assert correct `AttendanceRecord` statuses.

---

## Implementation Notes

```
AttendancePass1Service.stamp(locationId, date)
    ├── WorkSessionRepository.findByLocationAndDate(locationId, date)
    ├── For each session:
    │     ├── Check employment_start_date boundary
    │     ├── Apply status rules (PRESENT / LATE / INSUFFICIENT_HOURS)
    │     └── AttendanceRecordRepository.upsert() [skip if is_overridden]
    └── Log result
```

- Thresholds come from `LocationConfig` — fetched once per location per run.
- The upsert query: `INSERT INTO attendance_records ... ON CONFLICT (user_id, record_date, location_id) DO UPDATE SET ... WHERE is_overridden = false`.
- `employment_start_date` is read from the `User` record — do not query Arms HR directly in this job; use the locally synced `User` table.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Override protection confirmed in unit tests
- [ ] Employment start date boundary confirmed in unit tests
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-2** and **AOMS-3** must be complete.
- `User.employment_start_date` populated (requires Arms HR sync — AOMS-4 sprint context).
- `LocationConfig` must include `min_presence_duration_minutes`.
