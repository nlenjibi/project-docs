# AOMS-10 — Super Admin Attendance Override (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-10-super-admin-attendance-override.md](attendance-service/AOMS-10-super-admin-attendance-override.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-10 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, admin, audit |
| **Perspective** | Backend |

---

## Story

As a super admin, I want to manually override an employee's attendance record so that genuine errors in the system can be corrected.

---

## Background (Backend Context)

Override is a sensitive operation — it is the only mechanism that allows a human to change a system-computed status. The design prioritises auditability above all else: both the original and new status are stored on the record, the override reason is mandatory, and the change is written to the `AuditLog`. Once overridden, a record cannot be silently re-stamped by the nightly job (AOMS-4 and AOMS-5 both respect the `is_overridden` flag).

A revert mechanism is also required: a Super Admin must be able to undo an override and restore the original system-computed status.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `PATCH /api/v1/attendance/{recordId}/override` updates the status of an `AttendanceRecord`.
  - Request body: `{ "newStatus": "PRESENT", "reason": "Badge reader fault on 2026-03-12" }`.
  - `reason` is mandatory — return HTTP 400 if absent or blank.
  - `newStatus` must be a valid `AttendanceStatus` enum value — return HTTP 400 otherwise.
  - On success, sets: `status = newStatus`, `is_overridden = true`, `override_reason = reason`, `original_status = previous status`, `overridden_at = now()`, `overridden_by = actorId`.
- [ ] Only Super Admin role can call this endpoint — return 403 for any other role.
- [ ] Super Admin can override records at **any** location (Super Admin has global scope).
- [ ] A record with `is_overridden = true` cannot be overridden again without first reverting — return HTTP 409 with `{ "code": "ALREADY_OVERRIDDEN", "message": "Record is already overridden. Revert before overriding again." }`.
- [ ] `PATCH /api/v1/attendance/{recordId}/revert` reverts an overridden record to its `original_status`.
  - Request body: `{ "reason": "Override applied in error" }`.
  - `reason` is mandatory.
  - Sets: `status = original_status`, `is_overridden = false`, `override_reason = null`, `original_status = null`, `overridden_by = null`, `overridden_at = null`.
  - Returns 409 if record is not currently overridden.
- [ ] Both override and revert write an `AuditLog` entry including `previous_state` and `new_state` (full record JSONB snapshots).
- [ ] Overridden records are marked with `is_overridden = true` — all read APIs (AOMS-6, AOMS-7, AOMS-8) must surface this field; React uses it to render the visual indicator.
- [ ] The nightly Pass 1 and Pass 2 jobs skip records where `is_overridden = true` — this is already enforced by AOMS-4 and AOMS-5.
- [ ] Unit tests: successful override, attempt to override already-overridden record (409), revert, revert on non-overridden record (409), missing reason (400).
- [ ] Integration test: Super Admin overrides a record; verifies audit log; verifies nightly job skips the overridden record on re-run.

---

## Implementation Notes

```
PATCH /api/v1/attendance/{recordId}/override
    ↓
@RequiresRole(SUPER_ADMIN) — no location check (Super Admin is global)
    ↓
AttendanceOverrideService.override(recordId, newStatus, reason, actorId)
    ├── AttendanceRecordRepository.findById(recordId) — 404 if not found
    ├── Guard: is_overridden = true → throw AlreadyOverriddenException (409)
    ├── Store originalStatus = record.status
    ├── Update record fields
    ├── AttendanceRecordRepository.save()
    └── AuditLogService.log(action="ATTENDANCE_OVERRIDDEN", previousState, newState)
```

- Store `original_status` on the entity: `original_status` (Enum, nullable) — populated on override, cleared on revert.
- Add `overridden_at` (Timestamp, nullable) and `overridden_by` (UUID FK → User, nullable) columns to `attendance_records` via Flyway migration.
- The `AuditLog` `previous_state` and `new_state` should capture the full `AttendanceRecord` as JSONB, not just the changed fields — this gives the audit reviewer the complete picture.

---

## API Contract

```
PATCH /api/v1/attendance/{recordId}/override
Request:  { "newStatus": "PRESENT", "reason": "Badge reader fault on 2026-03-12" }
Response 200: { "success": true, "data": { ...updatedRecord }, "error": null }
Response 400: { "success": false, "error": { "code": "MISSING_REASON", "message": "..." } }
Response 409: { "success": false, "error": { "code": "ALREADY_OVERRIDDEN", "message": "..." } }

PATCH /api/v1/attendance/{recordId}/revert
Request:  { "reason": "Override applied in error" }
Response 200: { "success": true, "data": { ...revertedRecord }, "error": null }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Super Admin role enforced; no other role can call override or revert
- [ ] Audit log entries confirmed for both override and revert
- [ ] `is_overridden` field returned on all attendance read API responses
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete (override protection must be implemented there first).
- **ATT-P03** must be complete (AuditLog infrastructure required).
