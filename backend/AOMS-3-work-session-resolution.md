# AOMS-3 — Work Session Resolution (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-3 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job |
| **Perspective** | Backend |

---

## Story

As a system, I want to collapse raw badge events into WorkSession records so that each employee's daily presence is represented as a coherent block of time.

---

## Background (Backend Context)

`WorkSession` is the intermediate layer between raw, immutable `BadgeEvent` records and the business-level `AttendanceRecord`. This separation is a deliberate architectural decision (AD-008): it allows badge-event resolution logic to be tested independently from status-determination logic, and it makes the full trace from raw event → session → record auditable.

This job runs as the second step in the nightly attendance pipeline, immediately after AOMS-2 completes. It processes `BadgeEvent` records written that night and produces one `WorkSession` per employee per work day (accounting for cross-midnight sessions and configurable gap thresholds).

**Pipeline:** Jenkins CI — lint → unit test → integration test → Docker build.

---

## Acceptance Criteria (Backend)

- [ ] `WorkSession` entity created with fields: `id` (UUID), `user_id`, `location_id`, `session_date` (Date — the date of first badge-in), `first_badge_in` (Timestamp), `last_badge_out` (Timestamp, nullable), `total_duration_minutes` (Integer, nullable — computed), `is_late` (Boolean), `minutes_late` (Integer, nullable), `crosses_midnight` (Boolean), `created_at`, `updated_at`. Soft-delete with `deleted_at`.
- [ ] Flyway migration creates `work_sessions` table with unique constraint on `(user_id, session_date, location_id)`.
- [ ] `WorkSessionResolutionService.resolve(userId, locationId, date)` processes all `BadgeEvent` records for the given user/location/date:
  - Groups events chronologically.
  - `first_badge_in` = earliest `BADGE_IN` event.
  - `last_badge_out` = latest `BADGE_OUT` event (null if no `BADGE_OUT` exists).
  - `total_duration_minutes` computed as `last_badge_out - first_badge_in` (null if no `BADGE_OUT`).
  - `crosses_midnight = true` if `last_badge_out` is on a different calendar date than `first_badge_in`.
  - `session_date` = date of `first_badge_in` (the session is owned by its start date).
- [ ] A new session is started when the gap between consecutive events exceeds `LocationConfig.session_gap_threshold_hours`. Each gap-separated block of events produces a separate `WorkSession`.
- [ ] `is_late = true` if `first_badge_in` time exceeds `LocationConfig.work_start_time + LocationConfig.lateness_threshold_minutes`.
- [ ] `minutes_late` is computed only when `is_late = true`; otherwise `null`.
- [ ] Sessions with no `BADGE_OUT` have `last_badge_out = null` and `total_duration_minutes = null` — they are not discarded.
- [ ] Resolution is idempotent: re-running for the same user/date deletes the existing `WorkSession` for that date and re-computes from scratch. This allows re-processing when source data is corrected.
- [ ] `WorkSessionResolutionJob` iterates over all active users with `BadgeEvent` records ingested in the most recent sync run and calls `WorkSessionResolutionService` per user/location/date.
- [ ] Job execution logged to `work_session_resolution_log` with: location, date processed, users processed, sessions created, errors.
- [ ] Unit tests: cross-midnight session, session gap splitting, is_late computation, no BADGE_OUT session, idempotent re-run.
- [ ] Integration test: seed `BadgeEvent` records; run job; assert correct `WorkSession` records created.

---

## Implementation Notes

```
WorkSessionResolutionJob
    ↓
WorkSessionResolutionService.resolve(userId, locationId, date)
    ├── BadgeEventRepository.findByUserAndLocationAndDate()
    ├── Apply gap threshold logic (from LocationConfig)
    ├── Compute fields (first_badge_in, last_badge_out, duration, is_late)
    └── WorkSessionRepository.upsert()
```

- The gap threshold logic: sort all events chronologically; iterate pairwise; when the time gap between event[i] and event[i+1] exceeds `session_gap_threshold_hours`, start a new session group.
- Session ownership rule: the `session_date` is always the calendar date of `first_badge_in` — not the date of the events that occur after midnight.
- LocationConfig is fetched once per location per job run, not per event.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80% on new code (resolution engine logic is critical — aim higher)
- [ ] Cross-midnight and gap-threshold scenarios explicitly covered in unit tests
- [ ] No audit log entry required for this job (it is a system computation, not a user action)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-2** must be complete — `BadgeEvent` records must exist before sessions can be resolved.
- `LocationConfig` entity must include `work_start_time`, `lateness_threshold_minutes`, `session_gap_threshold_hours`.
