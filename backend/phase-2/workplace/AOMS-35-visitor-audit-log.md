# AOMS-35 — Visitor Location Audit Log View (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-35 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 2 |
| **Phase** | Phase 2 |
| **Labels** | workplace, visitor, reporting, api |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As HR or a Facilities Admin, I want to view a complete, searchable log of all visits at my location so I can respond to security incidents, compliance audits, and access investigations.

---

## Acceptance Criteria

- [ ] `GET /api/v1/visits/location/{locationId}` — returns paginated list of all `VisitRecord`s at the location.
  - Accessible to `HR`, `FACILITIES_ADMIN`, `SUPER_ADMIN` only.
  - Query parameters: `from_date`, `to_date`, `status` (`EXPECTED | CHECKED_IN | COMPLETED`), `visitorEmail`, `hostUserId`, `page`, `size`.
  - Response per record: `visitId`, `visitorName`, `visitorEmail`, `hostUserName`, `purposeOfVisit`, `checkInAt`, `checkOutAt`, `agreementVersionSigned`, `status`.
  - `SUPER_ADMIN` can query any location; `HR` and `FACILITIES_ADMIN` restricted to their assigned `locationId`.
- [ ] `GET /api/v1/visits/{visitId}` — returns full detail of a single visit including agreement version signed. Accessible to host employee (own visits), `HR`, `FACILITIES_ADMIN`, `SUPER_ADMIN`.
- [ ] `GET /api/v1/visits/location/{locationId}/active` — currently checked-in visitors (status = `CHECKED_IN`). Accessible to `FACILITIES_ADMIN`, `HR`. No pagination — returns all active visitors.
- [ ] Empty result → HTTP 200 with `data.content = []`, not 404.
- [ ] Unit tests: employee cannot access `location/{locationId}` endpoint → 403; HR at wrong location → 403; SUPER_ADMIN can access any location.
- [ ] Integration test: seed 10 visits for two locations; HR at location A → sees only location A visits; SUPER_ADMIN → can query both.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Location scoping enforced — cross-location data never returned to non-SUPER_ADMIN
- [ ] Swagger annotations complete with all filter parameters described
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-32** and **AOMS-33** — visit records must exist.
