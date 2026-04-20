# AOMS-20 — Permanent Seat Assignment (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-20 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | seating, api, admin |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want to assign permanent seats to specific employees so that employees with fixed workstations have their seat reflected in the system.

---

## Background (Backend Context)

Permanent seat assignment is a configuration operation — it links a `PERMANENT` type seat to a specific `User`. A PERMANENT seat cannot be booked by any other employee via the hot-desk flow. When a permanent seat is unassigned, it can optionally be converted to `HOT_DESK` type, making it available for general booking.

The assignment and unassignment must be written to the `AuditLog`. The employee can see their permanent seat highlighted in the React floor plan — the backend surfaces `isMyPermanentSeat: true` on the floor plan response (already handled in AOMS-15 using `Seat.assigned_user_id`).

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `POST /api/v1/seats/{seatId}/permanent-assignment` — assigns a PERMANENT seat to an employee.
  - Request: `{ "userId": "uuid" }`.
  - Accessible to Facilities Admin and Super Admin.
  - `seatId` must be of type `PERMANENT` — return 400 if `seat_type = HOT_DESK`.
  - `seatId` must not already have an `assigned_user_id` — return 409 if already assigned.
  - `userId` must be an active employee at the same `location_id` as the seat — return 400 if not.
  - Sets `Seat.assigned_user_id = userId` and `Seat.status = AVAILABLE`.
  - Writes to `AuditLog`.
- [ ] `DELETE /api/v1/seats/{seatId}/permanent-assignment` — unassigns a PERMANENT seat.
  - Accessible to Facilities Admin and Super Admin.
  - Returns 400 if the seat has no current assignment.
  - Clears `Seat.assigned_user_id = null`.
  - Does not change `seat_type` — that is a separate operation.
  - Writes to `AuditLog`.
- [ ] `PATCH /api/v1/seats/{seatId}/type` — converts a seat type.
  - Accessible to Facilities Admin and Super Admin.
  - Request: `{ "seatType": "HOT_DESK" }`.
  - A `PERMANENT` seat with an active `assigned_user_id` cannot be converted to `HOT_DESK` without first being unassigned — return 400 with `{ "code": "SEAT_STILL_ASSIGNED", "message": "Unassign the seat before converting to HOT_DESK." }`.
  - A `HOT_DESK` seat can be converted to `PERMANENT` only if it has no CONFIRMED bookings for future dates — return 400 with `{ "code": "SEAT_HAS_FUTURE_BOOKINGS", "message": "Cancel future bookings before converting to PERMANENT." }`.
  - Writes to `AuditLog`.
- [ ] The hot-desk booking endpoint (AOMS-16) must reject attempts to book a `PERMANENT` seat — this validation already exists in AOMS-16 but must be verified against the `assigned_user_id` field: any seat with `seat_type = PERMANENT` is unbookable via the general booking flow regardless of `assigned_user_id` being set.
- [ ] Unit tests: assign to non-PERMANENT seat (400), assign to already-assigned seat (409), unassign non-assigned seat (400), type conversion blocked by future bookings, type conversion blocked by existing assignment.
- [ ] Integration test: assign seat to employee; verify employee's floor plan response shows `isMyPermanentSeat: true`; verify another employee cannot book the seat.

---

## Implementation Notes

```
POST /api/v1/seats/{seatId}/permanent-assignment
    ↓
@RequiresRole({FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership
    ↓
PermanentSeatAssignmentService.assign(seatId, userId, actorId)
    ├── Load seat; validate seat_type = PERMANENT
    ├── Guard: assigned_user_id IS NOT NULL → 409
    ├── Validate: userId active employee at seat.location_id
    ├── seat.assigned_user_id = userId
    ├── SeatRepository.save(seat)
    └── AuditLogService.log(action="PERMANENT_SEAT_ASSIGNED", ...)
```

- The `assigned_user_id` column already exists on `Seat` from AOMS-14. This ticket adds the management APIs and business rules around it.
- For the type conversion check (PERMANENT → HOT_DESK), query: `SELECT COUNT(*) FROM seat_bookings WHERE seat_id = ? AND booking_date >= today AND status = 'CONFIRMED'`. If > 0 → return 400.

---

## API Contract

```
POST /api/v1/seats/{seatId}/permanent-assignment
Request:  { "userId": "uuid" }
Response 200: { "success": true, "data": { "seatId": "uuid", "seatNumber": "2A-14", "assignedUserId": "uuid", "assignedUserName": "Jane Doe" } }

DELETE /api/v1/seats/{seatId}/permanent-assignment
Response 200: { "success": true, "data": { "seatId": "uuid", "assignedUserId": null } }

PATCH /api/v1/seats/{seatId}/type
Request:  { "seatType": "HOT_DESK" }
Response 200: { "success": true, "data": { "seatId": "uuid", "seatType": "HOT_DESK" } }
Response 400: { "success": false, "error": { "code": "SEAT_STILL_ASSIGNED", "message": "..." } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All guard conditions covered in unit tests
- [ ] `isMyPermanentSeat` confirmed in floor plan response (AOMS-15 integration)
- [ ] Audit log entries for assign, unassign, type conversion
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-14** must be complete (`Seat.assigned_user_id` column added in that migration).
- **AOMS-15** must be complete (floor plan highlights permanent seat).
- **AOMS-16** must be complete (booking rejects PERMANENT seats).
