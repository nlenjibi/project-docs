# AOMS-15 — Floor Plan Visualisation (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-15 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | seating, api |
| **Perspective** | Backend |

---

## Story

As an employee, I want to view an interactive floor plan for my office so that I can see seat availability and navigate the office layout.

---

## Background (Backend Context)

This ticket delivers the read API that powers the React floor plan component. The API must return the full floor plan structure with real-time per-seat availability for a given date — all in a single response to avoid multiple round trips from the frontend.

Availability is computed in real-time from `SeatBooking` records — no pre-computed cache. This is intentional (AD-015): at the expected scale (hundreds of seats per location) the query is fast enough with proper indexing.

The `seat_visibility_mode` from `LocationConfig` controls how much occupant information is returned. The backend enforces this — the React client does not strip data itself.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `GET /api/v1/locations/{locationId}/floor-plan?date=2026-03-14` returns the full floor plan for the location on the given date.
- [ ] Response structure:
  ```
  floors[] →
    zones[] →
      seats[] → { seatId, seatNumber, seatType, status, xPosition, yPosition,
                  availability (AVAILABLE / BOOKED / UNAVAILABLE / MAINTENANCE),
                  occupantInfo (see visibility mode rules below) }
  ```
- [ ] A seat's `availability` is derived in real-time:
  - `AVAILABLE` if no `SeatBooking` with `status = CONFIRMED` exists for that `seat_id` and `date`.
  - `BOOKED` if a CONFIRMED booking exists.
  - `UNAVAILABLE` if `Seat.status = UNAVAILABLE`.
  - `MAINTENANCE` if `Seat.status = MAINTENANCE`.
- [ ] `occupantInfo` is populated according to `LocationConfig.seat_visibility_mode`:
  - `FULL`: return `{ userId, name }` of the booking holder for all BOOKED seats.
  - `TEAM_ONLY`: return `{ userId, name }` only if the booking holder is in the same team (shares the same `manager_id`) as the requesting user; otherwise return `null`.
  - `AVAILABILITY_ONLY`: always return `null` for `occupantInfo`.
- [ ] The requesting employee's own permanent seat is flagged: `isMyPermanentSeat: true`.
- [ ] `date` defaults to today if not provided; must not be more than `LocationConfig.hot_desk_booking_window_days` days in the future (return 400 beyond this).
- [ ] Only active floors (`is_active = true`) and active zones (`is_active = true`) are included in the response. Inactive seats are excluded.
- [ ] Employee can only request the floor plan for locations where they hold a role.
- [ ] Response is performant for 500+ seats — the availability join is a single bulk query, not a per-seat loop.
- [ ] Unit tests: `FULL` mode returns occupant names, `AVAILABILITY_ONLY` strips all names, `TEAM_ONLY` strips names for out-of-team users, inactive floor excluded.
- [ ] Integration test: seed floors/zones/seats with mixed bookings; verify availability response matches expected seat states.

---

## Implementation Notes

```
GET /api/v1/locations/{locationId}/floor-plan?date=2026-03-14
    ↓
FloorPlanService.getFloorPlan(locationId, date, requestingUserId)
    ├── FloorRepository.findActiveByLocation(locationId)  → floors + zones + seats (JOIN)
    ├── SeatBookingRepository.findConfirmedByLocationAndDate(locationId, date)
    │     → Map<seatId, BookingDto>  (one bulk query — NOT per seat)
    ├── Apply availability: for each seat, check if seatId in booking map
    ├── Apply visibility mode (from LocationConfig)
    │     ├── TEAM_ONLY: resolve requesting user's team_members set first
    └── Map to FloorPlanResponseDto
```

- The bulk availability query: `SELECT seat_id, user_id FROM seat_bookings WHERE location_id = :locationId AND booking_date = :date AND status = 'CONFIRMED' AND deleted_at IS NULL`. This returns a map of `seatId → bookingUserId` in one query.
- For `TEAM_ONLY` mode: resolve the requesting user's `manager_id`; then fetch all users with the same `manager_id` at the location; filter occupant info to only those users.
- Use a flat `JOIN` query to fetch `Floor → Zone → Seat` in one database round trip; assemble the nested structure in Java.

---

## API Contract

```
GET /api/v1/locations/{locationId}/floor-plan?date=2026-03-14

Response 200:
{
  "success": true,
  "data": {
    "locationId": "uuid",
    "date": "2026-03-14",
    "floors": [
      {
        "floorId": "uuid", "name": "Floor 2", "floorNumber": 2,
        "zones": [
          {
            "zoneId": "uuid", "name": "Zone A",
            "seats": [
              {
                "seatId": "uuid",
                "seatNumber": "2A-14",
                "seatType": "HOT_DESK",
                "xPosition": 120,
                "yPosition": 340,
                "availability": "BOOKED",
                "isMyPermanentSeat": false,
                "occupantInfo": { "userId": "uuid", "name": "John Smith" }
              }
            ]
          }
        ]
      }
    ]
  },
  "error": null
}
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All three visibility modes tested
- [ ] Availability derived from a single bulk query (no per-seat loop)
- [ ] Inactive floors/zones/seats excluded
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-14** must be complete (Floor/Zone/Seat hierarchy must exist).
- **AOMS-16** must be at least partially scaffolded so `SeatBooking` table exists for the availability join.
- **AOMS-9** must be complete for `seat_visibility_mode` to be readable from `LocationConfig`.
