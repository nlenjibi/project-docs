# AOMS-9 — Attendance Configuration per Location (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-9 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, config, admin |
| **Perspective** | Backend |

---

## Story

As a facilities admin, I want to configure attendance policy settings for my office so that the system correctly interprets badge data according to our office's rules.

---

## Background (Backend Context)

`LocationConfig` is the single source of configurable attendance policy per office. Changes to thresholds affect the **next** nightly sync run — they do not retroactively alter existing `AttendanceRecord` or `WorkSession` data. Configuration history is preserved by creating a new `LocationConfigHistory` record on every update rather than overwriting the current values — this provides an audit trail of policy changes and supports future retroactive re-stamping.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `LocationConfig` entity includes attendance-specific fields: `work_start_time` (Time), `lateness_threshold_minutes` (Integer), `min_presence_duration_minutes` (Integer), `no_show_release_time` (Time), `session_gap_threshold_hours` (Integer), `hot_desk_booking_window_days` (Integer), `booking_cancellation_cutoff_hours` (Integer), `seat_visibility_mode` (Enum: `FULL`, `TEAM_ONLY`, `AVAILABILITY_ONLY`).
- [ ] `GET /api/v1/locations/{locationId}/config` returns the current `LocationConfig` for the given location. Accessible to all authenticated roles (read-only for Employee/Manager).
- [ ] `PATCH /api/v1/locations/{locationId}/config` updates one or more attendance config fields. Accessible to Facilities Admin and Super Admin only.
- [ ] Before applying the update, the current state is written to `location_config_history` table: `{ id, location_id, changed_at, changed_by, previous_values (JSONB), new_values (JSONB) }`.
- [ ] Validation rules enforced at the service layer:
  - `lateness_threshold_minutes` must be ≥ 0 and ≤ 120.
  - `min_presence_duration_minutes` must be > 0 and ≤ 480 (8 hours).
  - `session_gap_threshold_hours` must be ≥ 1 and ≤ 12.
  - `work_start_time` must be a valid `HH:mm` time.
  - `no_show_release_time` must be after `work_start_time`.
  - Invalid values return HTTP 400 with field-specific error messages.
- [ ] `GET /api/v1/locations/{locationId}/config/history` returns paginated history of configuration changes. Accessible to Facilities Admin and Super Admin only.
- [ ] All config updates are written to the `AuditLog` via `AuditLogService`.
- [ ] Changes take effect from the **next** nightly sync run — no retroactive re-stamping triggered by this ticket.
- [ ] Unit tests: each validation rule, partial update (only provided fields updated), history creation.
- [ ] Integration test: Facilities Admin updates config; verify history record created; verify current config updated; verify audit log entry written.

---

## Implementation Notes

```
PATCH /api/v1/locations/{locationId}/config
    ↓
@RequiresRole({FACILITIES_ADMIN, SUPER_ADMIN}) + location ownership check
    ↓
LocationConfigService.update(locationId, patchRequest, actorId)
    ├── Validate fields
    ├── Load current config
    ├── Persist current state to location_config_history
    ├── Apply patch to LocationConfig
    ├── LocationConfigRepository.save()
    └── AuditLogService.log(...)
```

- Use a `PATCH` (not `PUT`) — only provided fields are updated; omitted fields retain current values. Use `@JsonInclude(NON_NULL)` on the request DTO.
- `location_config_history` stores `previous_values` and `new_values` as JSONB snapshots of only the fields that changed.
- The nightly jobs (`BadgeSyncJob`, `WorkSessionResolutionJob`, `AttendancePass1Service`) fetch `LocationConfig` once per run at job start — they do not poll for changes mid-run.

---

## API Contract

```
GET /api/v1/locations/{locationId}/config
Response 200:
{
  "success": true,
  "data": {
    "locationId": "uuid",
    "workStartTime": "09:00",
    "latenessThresholdMinutes": 15,
    "minPresenceDurationMinutes": 240,
    "noShowReleaseTime": "10:30",
    "sessionGapThresholdHours": 6,
    "hotDeskBookingWindowDays": 14,
    "bookingCancellationCutoffHours": 2,
    "seatVisibilityMode": "FULL"
  },
  "error": null
}

PATCH /api/v1/locations/{locationId}/config
Request body:
{ "latenessThresholdMinutes": 20, "minPresenceDurationMinutes": 300 }

GET /api/v1/locations/{locationId}/config/history?page=0&size=10
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All validation rules tested
- [ ] History record created on every update
- [ ] Audit log entry written for every config change
- [ ] Role access enforced (Facilities Admin / Super Admin for write; all roles for read)
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- `LocationConfig` entity must be seeded with default values per location before any nightly jobs run.
