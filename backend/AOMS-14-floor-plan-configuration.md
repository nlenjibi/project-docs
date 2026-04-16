# AOMS-14 — Floor Plan Configuration (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-14 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | seating, config, admin |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want to configure the floor plan for my office so that employees can see an accurate representation of available seating.

---

## Background (Backend Context)

This ticket creates the `Floor → Zone → Seat` hierarchy — the foundational data model for the entire Seating domain. Everything else in seating (booking, availability, no-show release, block reservations) depends on this hierarchy being in place.

Entities use soft deletes — deactivating a floor or zone does not delete child records but marks the parent as inactive, which the availability query must respect. Permanent seat assignment (user-linked) is managed in AOMS-20 — this ticket only establishes the hierarchy itself.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `Floor` entity: `id` (UUID), `location_id`, `name` (String), `floor_number` (Integer), `is_active` (Boolean, default true), `created_at`, `updated_at`, `deleted_at`.
- [ ] `Zone` entity: `id` (UUID), `floor_id`, `location_id`, `name` (String), `is_active` (Boolean, default true), `created_at`, `updated_at`, `deleted_at`.
- [ ] `Seat` entity: `id` (UUID), `zone_id`, `floor_id`, `location_id`, `seat_number` (String), `seat_type` (Enum: `PERMANENT`, `HOT_DESK`), `status` (Enum: `AVAILABLE`, `UNAVAILABLE`, `MAINTENANCE`), `x_position` (Integer), `y_position` (Integer), `assigned_user_id` (UUID FK → User, nullable — for PERMANENT seats), `is_active` (Boolean, default true), `created_at`, `updated_at`, `deleted_at`.
- [ ] Flyway migrations create all three tables with appropriate indexes and foreign keys.
- [ ] **Floor CRUD** (Facilities Admin / Super Admin):
  - `POST /api/v1/locations/{locationId}/floors` — create a floor.
  - `GET /api/v1/locations/{locationId}/floors` — list all floors (including inactive).
  - `PUT /api/v1/locations/{locationId}/floors/{floorId}` — update name or floor_number.
  - `PATCH /api/v1/locations/{locationId}/floors/{floorId}/deactivate` — sets `is_active = false`. Deactivating a floor cascades deactivation to all child Zones and Seats.
- [ ] **Zone CRUD** (Facilities Admin / Super Admin):
  - `POST /api/v1/locations/{locationId}/floors/{floorId}/zones` — create a zone.
  - `PUT /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}` — update zone name.
  - `PATCH /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}/deactivate` — deactivates zone; cascades to child Seats.
- [ ] **Seat CRUD** (Facilities Admin / Super Admin):
  - `POST /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}/seats` — create a seat.
  - `PUT /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}/seats/{seatId}` — update seat fields.
  - `PATCH /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}/seats/{seatId}/status` — update `status` (AVAILABLE / UNAVAILABLE / MAINTENANCE).
- [ ] All mutations write to `AuditLog`.
- [ ] Changes to the floor plan (new seat created, seat deactivated) are immediately reflected in the availability query used by the React floor plan visualisation — no cache to invalidate.
- [ ] Only Facilities Admin and Super Admin can write; all authenticated roles can read.
- [ ] Unit tests: cascade deactivation, duplicate seat number in same zone (400), inactive floor not returned in availability query.
- [ ] Integration test: create floor → zone → seat hierarchy; verify it appears in a read query.

---

## Implementation Notes

```
POST /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}/seats
    ↓
@RequiresRole({FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership check
    ↓
SeatService.createSeat(zoneId, floorId, locationId, request, actorId)
    ├── Validate: seat_number unique within zone
    ├── SeatRepository.save()
    └── AuditLogService.log(...)
```

- Cascade deactivation: when deactivating a `Floor`, set `is_active = false` on all `Zone` records for that floor, and all `Seat` records for those zones — do this in a single service transaction.
- `x_position` and `y_position` are pixel coordinates used by the React floor plan renderer — no server-side validation beyond being non-negative integers.
- Enforce: a `Seat` with `seat_type = PERMANENT` must have `assigned_user_id` set before it can be used. A `Seat` with `seat_type = HOT_DESK` must NOT have `assigned_user_id` set (enforced in service layer).

---

## API Contract (Floor example — Zone and Seat follow the same pattern)

```
POST /api/v1/locations/{locationId}/floors
Request: { "name": "Floor 2", "floorNumber": 2 }
Response 201: { "success": true, "data": { "id": "uuid", "name": "Floor 2", "floorNumber": 2, "isActive": true } }

POST /api/v1/locations/{locationId}/floors/{floorId}/zones/{zoneId}/seats
Request: { "seatNumber": "2A-14", "seatType": "HOT_DESK", "xPosition": 120, "yPosition": 340 }
Response 201: { "success": true, "data": { "id": "uuid", "seatNumber": "2A-14", "seatType": "HOT_DESK", "status": "AVAILABLE" } }

PATCH /api/v1/locations/{locationId}/floors/{floorId}/deactivate
Response 200: { "success": true, "data": { "floorId": "uuid", "cascadedZones": 4, "cascadedSeats": 48 } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Cascade deactivation covered in unit tests
- [ ] Audit log entries for all mutations
- [ ] Read access available to all roles; write access Facilities Admin / Super Admin only
- [ ] Swagger annotations complete on all CRUD endpoints
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- `LocationConfig` seeded per office.
- All seating tickets (AOMS-15 through AOMS-21) depend on this ticket being complete first.
