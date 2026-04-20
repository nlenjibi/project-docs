# AOMS-18 — No-Show Auto-Release (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-18 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 2 |
| **Labels** | seating, batch-job, scheduled |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want the system to automatically release unattended bookings at the configured cutoff time so that seats are not wasted when employees do not show up.

---

## Background (Backend Context)

The no-show auto-release job is a daily scheduled job per location. It runs at `LocationConfig.no_show_release_time` and identifies CONFIRMED bookings for today where the booking holder has no badge-in event at that location. Those bookings are set to `RELEASED`, a `NoShowRecord` is created, and a daily summary SQS notification is sent to the Facilities Admin.

The job depends on `BadgeEvent` records from the badge sync (AOMS-2) already being available for today. In a multi-location setup, each location runs its own job instance at its configured release time.

**Messaging:** SQS. Queue: `oms-notifications-queue` (daily summary). No real-time per-release notification.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `NoShowRecord` entity: `id` (UUID), `seat_booking_id` (FK → SeatBooking), `user_id`, `location_id`, `booking_date` (Date), `auto_released_at` (Timestamp), `created_at`. No `deleted_at` — immutable once created.
- [ ] Flyway migration creates `no_show_records` table with unique constraint on `seat_booking_id`.
- [ ] `NoShowAutoReleaseJob` is a `@Scheduled` Spring component. Cron expression configured via `NO_SHOW_RELEASE_CRON` env var (default `0 10 * * *` — 10am daily).
- [ ] Per-location execution: the job iterates all active `Location` records and runs the release logic for each. The effective release time per location is read from `LocationConfig.no_show_release_time`; a location's job skips if the current time (in the location's timezone) has not yet reached its configured release time.
- [ ] Release logic per location:
  1. Find all `SeatBooking` records where: `booking_date = today`, `status = CONFIRMED`, `location_id = current location`, `deleted_at IS NULL`.
  2. For each booking, check if a `BadgeEvent` with `event_type = BADGE_IN` exists for `booking.user_id` at `location_id` on today's date with `occurred_at >= today midnight`.
  3. If no badge-in exists: update `SeatBooking.status = RELEASED`, set `auto_released_at = now()`. Create `NoShowRecord`. Do NOT update `SeatBooking.deleted_at` — RELEASED is a terminal status, not a deletion.
- [ ] Released seats immediately return to the available pool — the availability query in AOMS-15 queries for `status = CONFIRMED` only, so RELEASED bookings are automatically excluded.
- [ ] After processing all no-shows for a location, publish **one** SQS message to `oms-notifications-queue`: `{ "type": "NO_SHOW_DAILY_SUMMARY", "locationId": "...", "date": "...", "releasedCount": N, "releasedBookings": [{ "userId", "seatNumber", "bookingDate" }, ...] }`.
- [ ] Job execution logged to `no_show_job_log`: `location_id`, `run_date`, `started_at`, `completed_at`, `bookings_checked`, `bookings_released`, `status`.
- [ ] Job is idempotent: if re-run for the same date and location, already-RELEASED bookings are skipped. `NoShowRecord` unique constraint on `seat_booking_id` prevents duplicates.
- [ ] Unit tests: no badge-in → released, badge-in exists → not released, already-released booking skipped, idempotent re-run, per-location timezone cutoff check.
- [ ] Integration test: seed CONFIRMED bookings with and without badge-in events; run job; verify correct bookings are released and `NoShowRecord` created; verify non-attended booking was NOT released (badge-in present).

---

## Implementation Notes

```
NoShowAutoReleaseJob (@Scheduled cron)
    ↓
For each active Location:
    ├── Check: has LocationConfig.no_show_release_time been reached for this location? (timezone-aware)
    ├── SeatBookingRepository.findConfirmedBookingsForToday(locationId, today)
    ├── BadgeEventRepository.findBadgeInsForToday(locationId, today) → Set<userId>
    ├── For each booking where booking.userId NOT IN badgeInUserIds:
    │     ├── booking.status = RELEASED, booking.auto_released_at = now()
    │     ├── SeatBookingRepository.save(booking)
    │     └── NoShowRecordRepository.save(NoShowRecord)
    ├── SqsPublisher.send("oms-notifications-queue", dailySummaryMessage)
    └── NoShowJobLogRepository.save(result)
```

- Fetch all badge-in user IDs for the location and date in a **single bulk query** — do not check per booking. The set of users with badge-ins is typically small; the set of bookings is at most hundreds.
- Use `@Transactional` per-booking or per-batch to ensure a booking update and its `NoShowRecord` are created atomically.
- The SQS daily summary message is sent even when `releasedCount = 0` — the notification service can decide not to send an email in the zero case.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Idempotency confirmed in unit tests
- [ ] Timezone-aware release time check confirmed
- [ ] Badge-in bulk query confirmed (no N+1)
- [ ] SQS daily summary published per location
- [ ] Job log persisted for every run
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-16** must be complete (`SeatBooking` entity).
- **AOMS-2** must be complete (`BadgeEvent` records available for today).
- `LocationConfig.no_show_release_time` and `Location.timezone` seeded.
- SQS queue `oms-notifications-queue` provisioned.
- **AOMS-12** (No-Show Reporting) depends on this ticket.
