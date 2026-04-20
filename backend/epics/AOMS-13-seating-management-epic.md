# AOMS-13 — Seating Management (Epic)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-13 |
| **Type** | Epic |
| **Priority** | 🔴 Highest |
| **PI Objective** | PI-OBJ-02 |
| **Labels** | seating, epic |
| **Service** | `seating-service` (Spring Boot / Java) |

---

## Epic Summary

Enable employees to discover, book, and manage office seats across all locations. Facilities admins manage the physical seat hierarchy; employees self-serve hot-desk bookings within a configurable booking window. The system handles real-time availability, booking cancellations, automatic no-show release, manager block reservations, permanent seat assignments, and configurable occupant visibility — all surfaced through a floor plan visualisation API consumed by the React frontend.

---

## Business Goals

- Give employees a real-time, visual floor plan to find and book available seats without admin involvement.
- Automate no-show seat release so the desk pool is not locked by unchecked-in bookings.
- Allow managers to reserve seat blocks for their teams without restricting individual employee choice within the block.
- Support permanent seat assignments for employees with fixed workstations, clearly visible on the floor plan.
- Give admins control over how much occupant information is shown per office (full names, team-only, or availability only) to meet different privacy requirements.

---

## PI Objective — PI-OBJ-02

> *Seat booking flow operational end-to-end: hierarchy manageable by admins, hot-desk booking and cancellation live for employees, floor plan visualisation API serving real-time availability, no-show auto-release running nightly.*

PI-OBJ-02 is marked **complete at end of Sprint 2**.

---

## Stories

### Sprint 1

| ID | Title | Points |
|---|---|---|
| AOMS-14 | Seat Hierarchy Setup (Floor / Zone / Seat) | 3 |
| AOMS-16 | Hot-Desk Booking | 3 |
| AOMS-17 | Booking Cancellation | 2 |

### Sprint 2

| ID | Title | Points |
|---|---|---|
| AOMS-15 | Floor Plan Visualisation API | 3 |
| AOMS-18 | No-Show Auto-Release | 3 |
| AOMS-19 | Manager Block Reservation | 3 |
| AOMS-20 | Permanent Seat Assignment | 2 |
| AOMS-21 | Seat Visibility Configuration | 2 |

**Total: 8 stories · 21 story points**

---

## Data Model Overview

```
Location
  └── Floor (is_active, floor_number)
        └── Zone (is_active)
              └── Seat (seat_type: HOT_DESK | PERMANENT,
                        status: AVAILABLE | BOOKED | UNAVAILABLE | MAINTENANCE,
                        assigned_user_id — set for PERMANENT seats,
                        x_position, y_position)

SeatBooking (seat_id, user_id, booking_date, status: CONFIRMED | CANCELLED | NO_SHOW)
BlockReservation (manager_id, zone_id, start_date, end_date, seat_ids[])
LocationConfig (seat_visibility_mode: FULL | TEAM_ONLY | AVAILABILITY_ONLY,
                hot_desk_booking_window_days)
```

---

## Data Flow

```
Admin creates hierarchy (AOMS-14)
    ↓
Employee views floor plan (AOMS-15)
    ↓ (real-time availability from SeatBooking — single bulk query)
Employee books seat (AOMS-16)
    ↓
Employee cancels booking (AOMS-17)    OR    No-show auto-release job (AOMS-18)
                                                ↓ (cross-references badge events from attendance-service)
Manager block-reserves seats (AOMS-19)
Admin assigns permanent seat to employee (AOMS-20)
Admin sets seat visibility mode (AOMS-21)
    ↓ (visibility mode applied in floor plan API response — AOMS-15)
```

---

## Key Constraints

- **Real-time availability only** — no pre-computed availability cache. At the expected scale (hundreds of seats per location) a single bulk query is fast enough with proper indexing (AD-015).
- **Single bulk query for floor plan** — `AOMS-15` availability must not use a per-seat loop. Single `SELECT seat_id, user_id FROM seat_bookings WHERE location_id = ? AND booking_date = ? AND status = 'CONFIRMED'` returns the full map in one query.
- **PERMANENT seats are not bookable** via the hot-desk flow regardless of `assigned_user_id` state. The booking endpoint rejects any attempt to book a `PERMANENT` type seat.
- **Seat visibility is backend-enforced** — the React client does not strip `occupantInfo`. The backend applies visibility rules before sending the response.
- **Booking window** — configurable per location via `LocationConfig.hot_desk_booking_window_days`. The floor plan API returns 400 for dates beyond this window.
- **No-show job** reads badge events from the attendance pipeline — this is the first cross-service data dependency in PI 1.

---

## Acceptance at Epic Level

- [ ] Admin can create and manage full Floor → Zone → Seat hierarchy per location
- [ ] Employee can book any available hot-desk seat within the booking window from the floor plan
- [ ] Booking cancellation releases seat immediately (up to 2 hours before booking date)
- [ ] No-show auto-release job running nightly; unchecked-in bookings released at configurable cutoff
- [ ] Manager can reserve a seat block; team members see the block on the floor plan
- [ ] Permanent seats assigned by admin; employee sees `isMyPermanentSeat: true` on floor plan
- [ ] Seat visibility mode configurable per location; three modes verified end-to-end
- [ ] Floor plan API performant for 500+ seats (availability via single bulk query confirmed in load test)
- [ ] PI-OBJ-02 signed off by Product Owner at Sprint 2 demo

---

## Dependencies

- `ATT-P02` (Role + Location Interceptor) applied to all endpoints
- `ATT-P03` (Audit Log) wired to all state-changing operations (assign, unassign, type conversion)
- `AOMS-2` (badge event ingestion — attendance epic) must produce badge data that `AOMS-18` no-show job can query
- `ATT-P07` (SES email) for booking confirmation notifications (ATT-P07 covers `seat.booking.confirmed` event template)
