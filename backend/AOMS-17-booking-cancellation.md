# AOMS-17 — Booking Cancellation (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-17 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 1 |
| **Labels** | seating, api, booking |
| **Perspective** | Backend |

---

## Story

As an employee, I want to cancel my hot-desk booking so that the seat is released for someone else if my plans change.

---

## Background (Backend Context)

Cancellation must enforce the 2-hour cutoff window — bookings cannot be cancelled less than `LocationConfig.booking_cancellation_cutoff_hours` hours before the start of the booking day (i.e. before midnight of the booking date, minus the cutoff). The seat returns to the available pool immediately on cancellation — the availability query in AOMS-15 already handles this by checking for CONFIRMED bookings only.

An SQS notification message is published on cancellation so the React UI can display a confirmation email. Cancellation is also written to the `AuditLog`.

**Messaging:** SQS. Queue: `oms-notifications-queue`.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `DELETE /api/v1/seat-bookings/{bookingId}` cancels the booking.
  - Employee can only cancel their **own** CONFIRMED bookings — return 403 if `booking.user_id != requestingUserId`.
  - Booking must have `status = CONFIRMED` — return 409 if status is already `CANCELLED` or `RELEASED`.
  - Cutoff check: cancellation is blocked if `now()` is within `LocationConfig.booking_cancellation_cutoff_hours` hours of `booking_date` midnight. Return HTTP 400 with `{ "code": "CANCELLATION_CUTOFF_PASSED", "message": "Cancellation is no longer allowed within {X} hours of the booking date." }`.
  - On success: set `status = CANCELLED`, `cancelled_at = now()`.
  - Seat availability is restored immediately — the unique partial index on CONFIRMED bookings means no explicit seat status update is needed; the next availability query will find no CONFIRMED booking for that seat/date.
  - Publish SQS message to `oms-notifications-queue`: `{ "type": "SEAT_BOOKING_CANCELLED", "userId": "...", "seatId": "...", "bookingDate": "...", "locationId": "..." }`.
  - Write to `AuditLog`.
- [ ] Facilities Admin and Super Admin can cancel **any** employee's booking at their location without the cutoff restriction (admin override). Request must include an optional `reason` field.
- [ ] `GET /api/v1/seat-bookings/{bookingId}` returns booking detail — employee sees own booking; Facilities Admin sees any booking at their location.
- [ ] Unit tests: own booking cancelled successfully, cutoff enforcement, already-cancelled (409), cancel another user's booking (403), Facilities Admin bypass of cutoff.
- [ ] Integration test: employee cancels booking; verify `status = CANCELLED`; verify seat appears as available in the floor plan query.

---

## Implementation Notes

```
DELETE /api/v1/seat-bookings/{bookingId}
    ↓
SeatBookingCancellationService.cancel(bookingId, requestingUser, reason)
    ├── SeatBookingRepository.findById(bookingId) — 404 if not found
    ├── Ownership check: booking.userId == requestingUserId OR role is FACILITIES_ADMIN/SUPER_ADMIN
    ├── Status check: CONFIRMED only
    ├── Cutoff check: skip for FACILITIES_ADMIN / SUPER_ADMIN
    │     └── now() < booking_date midnight - cutoff_hours → allowed; else → 400
    ├── Update: status = CANCELLED, cancelled_at = now()
    ├── SeatBookingRepository.save()
    ├── SqsPublisher.send("oms-notifications-queue", { type: SEAT_BOOKING_CANCELLED, ... })
    └── AuditLogService.log(...)
```

- Cutoff computation: `booking_date` is a `LocalDate`. The cutoff deadline is `LocalDateTime.of(bookingDate, LocalTime.MIDNIGHT).minus(cutoffHours, HOURS)` in the location's timezone. Compare against `ZonedDateTime.now(locationTimezone)`.
- The location timezone is read from `LocationConfig` (or `Location.timezone`) — do not compute cutoffs in UTC.
- SQS publish is best-effort and happens outside the transaction. Log and swallow `SqsException` — the cancellation must succeed even if the notification fails.

---

## API Contract

```
DELETE /api/v1/seat-bookings/{bookingId}
Response 200:
{
  "success": true,
  "data": { "bookingId": "uuid", "status": "CANCELLED", "cancelledAt": "2026-03-18T09:10:00Z" },
  "error": null
}

Response 400 (cutoff):
{ "success": false, "data": null, "error": { "code": "CANCELLATION_CUTOFF_PASSED", "message": "Cancellation not allowed within 2 hours of the booking date." } }

Response 403 (not owner):
{ "success": false, "data": null, "error": { "code": "FORBIDDEN", "message": "..." } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Timezone-aware cutoff computation confirmed in unit tests
- [ ] Facilities Admin bypass confirmed in unit tests
- [ ] SQS notification published; failure is swallowed and logged
- [ ] Audit log entry on every cancellation
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-16** must be complete (`SeatBooking` entity and booking flow must exist).
- `LocationConfig.booking_cancellation_cutoff_hours` and `Location.timezone` must be seeded.
- **ATT-P03** must be complete.
