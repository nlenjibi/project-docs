# AOMS-34 — Agreement Template Management (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-34 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 2 |
| **Phase** | Phase 2 |
| **Labels** | workplace, visitor, agreement, admin |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As a Facilities Admin, I want to manage versioned NDA / access agreement templates so that visitors always sign the latest terms and returning visitors are required to re-sign when the terms change.

---

## Acceptance Criteria

- [ ] `POST /api/v1/agreement-templates` — creates a new agreement template version.
  - Accessible to `FACILITIES_ADMIN`, `SUPER_ADMIN` only.
  - Request: `{ locationId, title, content (HTML or plain text), effectiveFrom (date) }`.
  - On creation: auto-increments `version` integer per location (1, 2, 3…). Sets `is_active = false` until explicitly activated.
  - Returns `{ templateId, version, effectiveFrom }`.
- [ ] `PATCH /api/v1/agreement-templates/{templateId}/activate` — activates a template version.
  - Accessible to `FACILITIES_ADMIN`, `SUPER_ADMIN`.
  - Deactivates all other templates at the same `locationId`. Only one active template per location at any time.
  - Once activated, the template cannot be edited (content is immutable after activation). Return 409 on edit attempt of an active template.
  - All future check-ins at this location will compare against this version.
- [ ] `GET /api/v1/agreement-templates/active?locationId={id}` — returns the currently active template.
  - Accessible to any authenticated role (needed by check-in flow).
- [ ] `GET /api/v1/agreement-templates?locationId={id}` — returns full version history for a location, paginated. Accessible to `FACILITIES_ADMIN`, `HR`, `SUPER_ADMIN`.
- [ ] Unit tests: activate template → previous template deactivated; edit active template → 409; no active template at location → check-in returns 500 with `NO_ACTIVE_AGREEMENT` error.
- [ ] Integration test: create two versions at same location; activate v2; call active endpoint → returns v2; activate v1 → returns v1; v2 now inactive.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Only one active template per location enforced at DB constraint level (partial unique index on `is_active = true` per `location_id`)
- [ ] Active template content immutable — edit blocked at service layer
- [ ] Audit log entry on `AGREEMENT_TEMPLATE_ACTIVATED`
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-33** (check-in) consumes `GET /api/v1/agreement-templates/active` to enforce version check.
- Must be deployed before `AOMS-33` can complete check-in flows.
