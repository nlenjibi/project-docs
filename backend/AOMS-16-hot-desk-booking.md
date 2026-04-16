# AOMS-16 — Hot-Desk Booking (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-16 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | seating, api, booking |
| **Perspective** | Backend |

---

## Story

As an employee, I want to book an available hot-desk seat for a specific date so that I have a confirmed workspace when I come into the office.

---

## Background (Backend Context)

Hot-desk booking is the highest-traffic write operation in the Seating domain. The core constraint is **no double-bookings**: two concurrent requests for the same seat on the same date must not both succeed. This is enforced at the database level via a unique partial index — not only in application code. Optimistic or pessimistic locking is needed for the read-check-write sequence.

An SQS message is published on successful booking to trigger the notification service (in-app + email confirmation to the employee). The booking is also written to the `AuditLog`.

**Messaging:** SQS (not Kafka). Queue: `oms-notifications-queue`.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `SeatBooking` entity: `id` (UUID), `seat_id`, `user_id`, `location_id`, `booking_date` (Date), `status` (Enum: `CONFIRMED`, `CANCELLED`, `RELEASED`), `booked_at` (Timestamp), `cancelled_at` (Timestamp, nullable), `cancellation_reason` (String, nullable), `auto_released_at` (Timestamp, nullable), `block_reservation_id` (UUID FK → BlockReservation, nullable), `created_at`, `updated_at`, `deleted_at`.
- [ ] Flyway migration creates `seat_bookings` table with: unique partial index on `(seat_id, booking_date)` WHERE `status = 'CONFIRMED'` — this enforces one confirmed booking per seat per day at the DB level.
- [ ] `POST /api/v1/seat-bookings` — creates a booking.
  - Request: `{ "seatId": "uuid", "bookingDate": "2026-03-20" }`.
  - `bookingDate` must be today or in the future.
  - `bookingDate` must not exceed `today + LocationConfig.hot_desk_booking_window_days`.
  - `seat_type` of the target seat must be `HOT_DESK` — return 400 if employee attempts to book a PERMANENT seat.
  - Seat must have `status = AVAILABLE` and no existing CONFIRMED booking for that date.
  - Employee cannot have more than one CONFIRMED booking per date at the same location — return 409.
  - On success: persist `SeatBooking` with `status = CONFIRMED`; publish SQS message to `oms-notifications-queue` with `{ "type": "SEAT_BOOKED", "userId": "...", "seatId": "...", "bookingDate": "...", "locationId": "..." }`.
  - Write to `AuditLog`.
- [ ] Concurrency safety: use a `SELECT ... FOR UPDATE` (pessimistic lock) on the seat for the given date within the booking transaction, or rely on the unique partial index to catch races at the DB level and catch `DataIntegrityViolationException` → return 409 `{ "code": "SEAT_UNAVAILABLE", "message": "Seat is no longer available for that date." }`.
- [ ] `GET /api/v1/seat-bookings/my?from=&to=` — returns the authenticated employee's own bookings, paginated, filtered by optional date range.
- [ ] `GET /api/v1/seat-bookings/my/{bookingId}` — returns a single booking. Returns 404 if booking does not belong to the authenticated user.
- [ ] Unit tests: successful booking, past date rejection, beyond-window rejection, PERMANENT seat rejection, one-per-day duplicate (409), concurrent booking race (409 via unique index violation).
- [ ] Integration test: two concurrent requests for the same seat on the same date — verify exactly one succeeds (HTTP 201) and one returns 409.

---

## Implementation Notes

```
POST /api/v1/seat-bookings
    ↓
@RequiresRole(EMPLOYEE or above) + location ownership check
    ↓
SeatBookingService.book(userId, locationId, seatId, bookingDate)
    ├── Validate date window (LocationConfig)
    ├── Validate seat is HOT_DESK and AVAILABLE
    ├── Check no existing CONFIRMED booking for user on that date/location
    ├── [DB transaction begins]
    │     ├── SELECT seat status + existing booking FOR UPDATE (or rely on unique index)
    │     ├── SeatBookingRepository.save(booking)
    │     └── [catch DataIntegrityViolationException → throw SeatUnavailableException]
    ├── [DB transaction commits]
    ├── SqsPublisher.send("oms-notifications-queue", { type: SEAT_BOOKED, ... })
    └── AuditLogService.log(...)
```

- The unique partial index approach is simpler than row-level locking and equally safe for this use case. Use `DataIntegrityViolationException` catch as the concurrency guard.
- SQS publish happens **after** the transaction commits — if the SQS publish fails, the booking is still persisted and the employee can see their booking in `GET /my`. The notification failure is logged and can be retried by the notification service's DLQ.
- `LocationConfig.hot_desk_booking_window_days` is fetched once at the start of the request, not cached in the service.

---

## API Contract

```
POST /api/v1/seat-bookings
Request: { "seatId": "uuid", "bookingDate": "2026-03-20" }
Response 201:
{
  "success": true,
  "data": {
    "bookingId": "uuid",
    "seatId": "uuid",
    "seatNumber": "2A-14",
    "floorName": "Floor 2",
    "bookingDate": "2026-03-20",
    "status": "CONFIRMED",
    "bookedAt": "2026-03-18T10:22:00Z"
  },
  "error": null
}

Response 409 (seat taken):
{ "success": false, "data": null, "error": { "code": "SEAT_UNAVAILABLE", "message": "..." } }

Response 409 (already have booking that day):
{ "success": false, "data": null, "error": { "code": "DUPLICATE_BOOKING", "message": "..." } }

GET /api/v1/seat-bookings/my?from=2026-03-01&to=2026-03-31&page=0&size=20
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Concurrency race condition covered in integration test
- [ ] Unique partial index on `(seat_id, booking_date) WHERE status = 'CONFIRMED'` confirmed in migration
- [ ] SQS notification published post-commit
- [ ] Audit log entry written on every confirmed booking
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-14** must be complete (Seat hierarchy must exist).
- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- SQS queue `oms-notifications-queue` provisioned.
- `LocationConfig.hot_desk_booking_window_days` seeded.
