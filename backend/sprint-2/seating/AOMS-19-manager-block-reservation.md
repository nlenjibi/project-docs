# AOMS-19 — Manager Block Reservation (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-19 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 5 |
| **Sprint** | Sprint 2 |
| **Labels** | seating, api, manager |
| **Perspective** | Backend |

---

## Story

As a manager, I want to reserve a block of seats for my team on a specific date so that my team can sit together on collaboration days.

---

## Background (Backend Context)

A `BlockReservation` is a manager-level action that reserves a number of seats in a specific zone for a date. The block does not assign seats to individual team members — it holds them in a semi-reserved state so team members can individually book from within the block. Seats not individually booked by the no-show cutoff time are auto-released by the existing no-show job (AOMS-18).

This ticket introduces the `BlockReservation` entity and the management APIs. It also modifies the booking API (AOMS-16) behaviour slightly: when a booking is created for a seat within a block, the `block_reservation_id` is linked on the `SeatBooking`.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `BlockReservation` entity: `id` (UUID), `manager_id` (FK → User), `location_id`, `zone_id`, `reservation_date` (Date), `requested_seats` (Integer), `reserved_seat_ids` (JSONB array of UUIDs), `status` (Enum: `ACTIVE`, `CANCELLED`, `EXPIRED`), `created_at`, `updated_at`, `deleted_at`.
- [ ] Flyway migration creates `block_reservations` table.
- [ ] `POST /api/v1/block-reservations` — creates a block reservation.
  - Request: `{ "zoneId": "uuid", "reservationDate": "2026-03-20", "requestedSeats": 5 }`.
  - Accessible to Manager, Facilities Admin, Super Admin.
  - `reservationDate` must be within the booking window.
  - The service selects `requestedSeats` available `HOT_DESK` seats from the specified zone (lowest seat number first for determinism) that have no CONFIRMED booking for that date.
  - If fewer available seats exist than `requestedSeats`, return 400 with `{ "code": "INSUFFICIENT_AVAILABLE_SEATS", "availableCount": N }`.
  - Creates a `SeatBooking` with `status = CONFIRMED` and `block_reservation_id` set for each reserved seat. The `user_id` on these block-holder bookings is set to the manager's ID (acts as a placeholder).
  - Persists `BlockReservation` with `reserved_seat_ids` populated.
  - Writes to `AuditLog`.
- [ ] Block-reserved seats appear as `BOOKED` on the floor plan (AOMS-15) — the placeholder CONFIRMED bookings ensure this automatically.
- [ ] Team members booking a seat within the block: when a team member calls `POST /api/v1/seat-bookings` for a seat in the block, the booking service detects an existing block-holder booking for that seat/date; it **replaces** the placeholder booking with the employee's booking (updates `user_id` on the existing `SeatBooking` record; preserves `block_reservation_id`). This avoids a double-booking constraint violation.
- [ ] `GET /api/v1/block-reservations/{reservationId}` — returns reservation detail including `reservedSeatIds` and the list of team members who have booked seats within the block.
- [ ] `DELETE /api/v1/block-reservations/{reservationId}` — cancels the block reservation.
  - Accessible to the creating Manager, Facilities Admin, Super Admin.
  - Sets `BlockReservation.status = CANCELLED`.
  - For each `SeatBooking` with `block_reservation_id = reservationId` that still has `user_id = manager_id` (i.e. not yet claimed by a team member), sets `status = CANCELLED`.
  - Team member bookings within the block are preserved — they are not cancelled.
  - Writes to `AuditLog`.
- [ ] `GET /api/v1/block-reservations?locationId=&date=&managerId=` — lists block reservations. Manager sees own reservations; Facilities Admin / Super Admin sees all at location.
- [ ] Unit tests: successful block creation, insufficient seats (400), team member claiming a seat within block, cancellation preserves individual bookings.
- [ ] Integration test: create block for 3 seats; have one team member book within block; cancel block; verify individual booking preserved and 2 placeholder bookings cancelled.

---

## Implementation Notes

```
POST /api/v1/block-reservations
    ↓
@RequiresRole({MANAGER, FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership
    ↓
BlockReservationService.create(managerId, locationId, zoneId, reservationDate, requestedSeats)
    ├── Validate date window
    ├── [Transaction]
    │     ├── Find available HOT_DESK seats in zone (no CONFIRMED booking for date)
    │     │     SELECT * FROM seats WHERE zone_id = ? AND seat_type = 'HOT_DESK'
    │     │       AND id NOT IN (SELECT seat_id FROM seat_bookings
    │     │                      WHERE booking_date = ? AND status = 'CONFIRMED')
    │     │     ORDER BY seat_number LIMIT requestedSeats FOR UPDATE
    │     ├── Guard: count < requestedSeats → 400
    │     ├── Create SeatBooking (status=CONFIRMED, user_id=managerId, block_reservation_id=TBD) per seat
    │     ├── Create BlockReservation (reserved_seat_ids = list of selected seat UUIDs)
    │     └── Update each SeatBooking.block_reservation_id = blockReservation.id
    ├── AuditLogService.log(...)
    └── Return BlockReservationResponseDto
```

- The `SELECT ... FOR UPDATE` on available seats prevents a concurrent individual booking from stealing seats mid-transaction.
- "Team member claiming" logic in AOMS-16's booking service: when `POST /api/v1/seat-bookings` is called for a seat with an existing CONFIRMED booking owned by the manager (block holder), update `user_id` on that `SeatBooking` rather than inserting a new one. Check: `existing.block_reservation_id IS NOT NULL AND existing.user_id = blockReservation.managerId`.

---

## API Contract

```
POST /api/v1/block-reservations
Request: { "zoneId": "uuid", "reservationDate": "2026-03-20", "requestedSeats": 5 }
Response 201:
{
  "success": true,
  "data": {
    "reservationId": "uuid",
    "zoneId": "uuid", "zoneName": "Zone A",
    "reservationDate": "2026-03-20",
    "requestedSeats": 5,
    "reservedSeats": [ { "seatId": "uuid", "seatNumber": "2A-10" }, ... ],
    "status": "ACTIVE"
  }
}

Response 400 (insufficient):
{ "success": false, "error": { "code": "INSUFFICIENT_AVAILABLE_SEATS", "availableCount": 3 } }

DELETE /api/v1/block-reservations/{reservationId}
Response 200: { "success": true, "data": { "status": "CANCELLED", "placeholderBookingsCancelled": 4, "individualBookingsPreserved": 1 } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Concurrency-safe seat selection (`FOR UPDATE`) confirmed
- [ ] Team-member claim flow confirmed in integration test
- [ ] Cancellation preserves individual bookings — confirmed in test
- [ ] Audit log entries for create and cancel
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-16** must be complete (SeatBooking entity and booking flow; block-claim logic modifies AOMS-16 behaviour).
- **AOMS-14** must be complete (Zone and Seat hierarchy).
