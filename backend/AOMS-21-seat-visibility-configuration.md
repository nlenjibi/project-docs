# AOMS-21 — Seat Visibility Configuration (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-21 |
| **Epic** | Seating Management — AOMS-13 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 1 |
| **Sprint** | Sprint 2 |
| **Labels** | seating, config, admin |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want to configure the seat visibility mode for my office so that I can control how much occupancy information employees can see on the floor plan.

---

## Background (Backend Context)

`seat_visibility_mode` is a field on `LocationConfig` (already introduced in AOMS-9). This ticket is a thin configuration ticket — the core visibility enforcement logic is implemented in the floor plan API (AOMS-15) which already reads `LocationConfig.seat_visibility_mode` and applies the appropriate filtering rules.

This ticket's scope is: the API to **update** the visibility mode, validation, audit logging, and the confirmation that the change takes effect immediately (no cache to flush).

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `PATCH /api/v1/locations/{locationId}/config/seat-visibility` — updates `seat_visibility_mode`.
  - Request: `{ "seatVisibilityMode": "TEAM_ONLY" }`.
  - Valid values: `FULL`, `TEAM_ONLY`, `AVAILABILITY_ONLY` — return 400 for any other value.
  - Accessible to Facilities Admin and Super Admin only — return 403 for all other roles.
  - Facilities Admin can only update config for their own location — return 403 for other locations.
  - Super Admin can update any location.
  - Applies immediately: the next call to `GET /api/v1/locations/{locationId}/floor-plan` reflects the new mode with no additional action required.
  - Writes the previous `seat_visibility_mode` to `location_config_history` (same history mechanism as AOMS-9).
  - Writes to `AuditLog`.
- [ ] `GET /api/v1/locations/{locationId}/config` (already exists from AOMS-9) must include `seatVisibilityMode` in the response — verify this field is exposed in the existing config endpoint.
- [ ] The floor plan API (AOMS-15) reads `seat_visibility_mode` from `LocationConfig` on every request — there is no caching layer to invalidate. Change takes effect immediately because the floor plan service fetches `LocationConfig` fresh per request.
- [ ] Unit tests: valid mode update, invalid mode value (400), Facilities Admin at wrong location (403).
- [ ] Integration test: set mode to `AVAILABILITY_ONLY`; call floor plan API; verify no `occupantInfo` is returned for any booked seat.

---

## Implementation Notes

```
PATCH /api/v1/locations/{locationId}/config/seat-visibility
    ↓
@RequiresRole({FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership
    ↓
LocationConfigService.updateSeatVisibility(locationId, newMode, actorId)
    ├── Load current LocationConfig
    ├── Store previous mode in location_config_history (JSONB: { "seatVisibilityMode": "FULL" })
    ├── config.seatVisibilityMode = newMode
    ├── LocationConfigRepository.save()
    └── AuditLogService.log(action="SEAT_VISIBILITY_UPDATED",
                            previousState={ seatVisibilityMode: oldMode },
                            newState={ seatVisibilityMode: newMode })
```

- This can reuse `LocationConfigService.update()` from AOMS-9 — the seat visibility patch is just a specific partial update of `LocationConfig`. If the general config PATCH already supports this field, this ticket simply verifies that the field is exposed, validated, and audited — and adds a dedicated endpoint alias for discoverability.
- The floor plan API (AOMS-15) must **not** cache `LocationConfig` — it must fetch it on every request for the "immediate effect" guarantee to hold. Verify this is the case in AOMS-15 implementation.

---

## API Contract

```
PATCH /api/v1/locations/{locationId}/config/seat-visibility
Request:  { "seatVisibilityMode": "TEAM_ONLY" }
Response 200:
{
  "success": true,
  "data": {
    "locationId": "uuid",
    "seatVisibilityMode": "TEAM_ONLY",
    "updatedAt": "2026-03-18T11:05:00Z"
  },
  "error": null
}

Response 400:
{ "success": false, "error": { "code": "INVALID_VISIBILITY_MODE", "message": "Valid values: FULL, TEAM_ONLY, AVAILABILITY_ONLY." } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; ≥ 80% coverage on new code
- [ ] Integration test: mode change → floor plan immediately reflects new mode
- [ ] Config history record created on update
- [ ] Audit log entry written on update
- [ ] Role and location access enforced
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-9** must be complete (`LocationConfig` and its PATCH infrastructure exist).
- **AOMS-15** must be complete (floor plan API reads and applies `seat_visibility_mode`).
- **ATT-P03** must be complete (AuditLog infrastructure).
