# AOMS-26 — Recurring Remote Schedule (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-26 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 5 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api, recurring |
| **Perspective** | Backend |

---

## Story

As an employee, I want to set up a recurring remote day schedule so that I don't have to submit individual requests each week for a regular arrangement.

---

## Background (Backend Context)

`RecurringRemoteSchedule` is the parent entity. A single manager approval generates all child `RemoteRequest` instances for future occurrences. The generation horizon is configurable (default: 12 weeks ahead) — the system does not generate instances indefinitely upfront. A weekly scheduled job (or a generation-on-demand trigger) extends the instances as the horizon approaches.

The parent/child pattern is standard: cancelling a single instance cancels only that `RemoteRequest`; cancelling the series terminates the parent (`effective_to` set to today) and cancels all future child instances. Past instances are never modified by a series cancellation.

Policy checks (AOMS-27) apply at the time the parent is submitted, not per-instance — the manager approves the arrangement, not each occurrence.

**Messaging:** SQS — `oms-notifications-queue`. **Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `RecurringRemoteSchedule` entity: `id` (UUID), `user_id`, `location_id`, `manager_id`, `day_of_week` (Enum: `MONDAY`–`FRIDAY`), `effective_from` (Date), `effective_to` (Date, nullable — null = open-ended), `status` (Enum: `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`), `notes` (String, nullable), `generation_horizon_weeks` (Integer, default 12), `reviewed_by` (UUID, nullable), `reviewed_at` (Timestamp, nullable), `rejection_reason` (String, nullable), `created_at`, `updated_at`, `deleted_at`.
- [ ] `RemoteRequest.recurring_schedule_id` FK already exists (from AOMS-23 migration). Verify FK is in place.
- [ ] Flyway migration creates `recurring_remote_schedules` table.
- [ ] **`POST /api/v1/recurring-remote-schedules`** — submits a recurring schedule.
  - Request: `{ "dayOfWeek": "WEDNESDAY", "effectiveFrom": "2026-04-01", "effectiveTo": "2026-09-30", "notes": "WFH Wednesdays" }`.
  - `effectiveFrom` must be a future date.
  - `dayOfWeek` must match the `effectiveFrom` date's day of week — return 400 otherwise (prevents ambiguous start).
  - Check for conflicting active recurring schedule on the same `dayOfWeek` for this user — return 409 `{ "code": "CONFLICTING_SCHEDULE" }` if found.
  - Apply the same remote day policy check as AOMS-23 against `max_remote_days_per_week` (counts the day being added as one recurring slot per week).
  - Persist `RecurringRemoteSchedule` with `status = PENDING`.
  - **Do not generate child instances yet** — children are generated only on approval.
  - Notify manager (SQS: `{ "type": "RECURRING_SCHEDULE_SUBMITTED", "approverId": "uuid", "requesterId": "uuid", ... }`).
  - Write to `AuditLog`.

- [ ] **`PATCH /api/v1/recurring-remote-schedules/{scheduleId}/approve`** — approves the schedule.
  - Accessible to the schedule's `manager_id` or active delegate (same logic as AOMS-25).
  - Sets `status = APPROVED`, `reviewed_by`, `reviewed_at`.
  - Calls `RecurringScheduleInstanceGeneratorService.generate(scheduleId)` — creates child `RemoteRequest` instances for each matching weekday from `effectiveFrom` to `MIN(effectiveTo, today + generation_horizon_weeks * 7)`. Each instance has `status = APPROVED` (parent approval implies child approval), `recurring_schedule_id` set.
  - Publish SQS `{ "type": "RECURRING_SCHEDULE_APPROVED", "requesterId": "uuid", ... }`.
  - Write to `AuditLog`.

- [ ] **`DELETE /api/v1/recurring-remote-schedules/{scheduleId}`** — cancels the entire series.
  - Sets `effective_to = today` on the parent, `status = CANCELLED`.
  - Cancels all child `RemoteRequest` records where `request_date > today` and `status = APPROVED` (sets their `status = CANCELLED`).
  - Past instances (where `request_date <= today`) are not modified.
  - Write to `AuditLog`.

- [ ] **`DELETE /api/v1/remote-requests/{requestId}/cancel-instance`** — cancels a single instance.
  - Request must have `recurring_schedule_id` set — return 400 if called on a non-recurring request (use `DELETE /api/v1/remote-requests/{id}` from AOMS-30 for that).
  - Sets the single `RemoteRequest` to `CANCELLED`.
  - Does not affect the parent or other instances.
  - Write to `AuditLog`.

- [ ] **`GET /api/v1/recurring-remote-schedules/my`** — lists the authenticated employee's recurring schedules (active and cancelled), with `instances` count summary.

- [ ] **Horizon extension job**: `RecurringScheduleHorizonJob` runs weekly (cron: `SCHEDULE_HORIZON_CRON` env var). For each `APPROVED` schedule with `effective_to = null` (open-ended), generates new child instances for the next week beyond the current furthest-generated date.

- [ ] Unit tests: approval generates correct child instances (correct weekdays only, skips public holidays), series cancellation leaves past instances intact, single instance cancel, conflicting schedule (409), day-of-week mismatch on effectiveFrom (400).
- [ ] Integration test: submit schedule; approve; verify child `RemoteRequest` instances exist for the correct dates; cancel series; verify future instances cancelled and past intact.

---

## Implementation Notes

```
RecurringScheduleInstanceGeneratorService.generate(scheduleId):
    schedule = RecurringRemoteScheduleRepository.findById(scheduleId)
    horizon = MIN(schedule.effectiveTo, today + schedule.generationHorizonWeeks * 7)
    current = schedule.effectiveFrom
    while current <= horizon:
        if current.dayOfWeek == schedule.dayOfWeek:
            if NOT PublicHoliday(locationId, current):
                RemoteRequestRepository.save(RemoteRequest {
                    userId, locationId, managerId,
                    requestDate = current,
                    status = APPROVED,
                    recurringScheduleId = scheduleId
                })
        current += 1 day
```

- Child instances are generated as `APPROVED` immediately on schedule approval — they do not go through a separate approval flow. The parent approval covers all instances.
- Public holidays within the horizon are skipped — no `RemoteRequest` is generated for a public holiday date.
- The generation is idempotent: uses `INSERT ... ON CONFLICT (user_id, request_date, location_id) WHERE status IN ('PENDING','APPROVED') DO NOTHING` to avoid duplicates on re-run.

---

## API Contract

```
POST /api/v1/recurring-remote-schedules
Request: { "dayOfWeek": "WEDNESDAY", "effectiveFrom": "2026-04-01", "effectiveTo": "2026-09-30" }
Response 201: { "success": true, "data": { "scheduleId": "uuid", "dayOfWeek": "WEDNESDAY", "status": "PENDING" } }

PATCH /api/v1/recurring-remote-schedules/{scheduleId}/approve
Response 200: { "success": true, "data": { "scheduleId": "...", "status": "APPROVED", "instancesGenerated": 26 } }

DELETE /api/v1/recurring-remote-schedules/{scheduleId}
Response 200: { "success": true, "data": { "status": "CANCELLED", "futureInstancesCancelled": 18, "pastInstancesPreserved": 8 } }

DELETE /api/v1/remote-requests/{requestId}/cancel-instance
Response 200: { "success": true, "data": { "requestId": "...", "status": "CANCELLED", "seriesUnaffected": true } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Instance generation produces correct weekday dates — confirmed in test
- [ ] Public holidays skipped during generation — confirmed in test
- [ ] Series cancellation preserves past instances — confirmed in test
- [ ] Single-instance cancel does not affect series — confirmed in test
- [ ] Horizon extension job is idempotent
- [ ] Audit log entries for submit, approve, reject, cancel
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-23** must be complete (`RemoteRequest` entity and `recurring_schedule_id` FK).
- **AOMS-25** must be complete (approval flow — schedule approval uses the same manager/delegate authorisation).
- **AOMS-27** must be complete (policy check on submission).
- **AOMS-11** must be complete (public holidays skipped during instance generation).
