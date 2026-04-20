# AOMS-32 — Visitor Registration: Pre-register and Walk-in (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-32 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | workplace, visitor, api |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As an employee, I want to pre-register an expected visitor so security has advance notice. As a Facilities Admin, I want to register walk-in visitors who arrive without pre-registration.

---

## Acceptance Criteria

- [ ] `POST /api/v1/visitors/pre-register` — employee pre-registers an expected visitor.
  - Request: `{ visitorEmail, visitorName, hostUserId, locationId, expectedDate, purposeOfVisit }`.
  - Creates or reuses a `VisitorProfile` (keyed on `visitorEmail`).
  - Creates a `ParentVisit` with `status = EXPECTED`.
  - Returns `{ visitId, visitorProfileId, status }`.
- [ ] `POST /api/v1/visitors/walk-in` — Facilities Admin registers a walk-in visitor.
  - Same request shape as pre-register; `hostUserId` required.
  - Creates `ParentVisit` with `status = CHECKED_IN`; creates `VisitRecord` with `check_in_at = now()`.
  - Accessible to `FACILITIES_ADMIN` only — returns 403 for other roles.
- [ ] `VisitorProfile` is reused across visits: same `visitorEmail` always maps to the same `VisitorProfile.id`. New visit creates a new `ParentVisit` referencing the existing profile.
- [ ] `GET /api/v1/visits/history` — employee views own pre-registered visitor history (own `host_user_id` only), paginated, filterable by `from_date` / `to_date`.
- [ ] `hostUserId` must be an active employee at `locationId` — return 400 if not.
- [ ] `expectedDate` must not be in the past — return 400.
- [ ] Unit tests: reuse existing visitor profile; walk-in by non-FACILITIES_ADMIN → 403; past expected date → 400.
- [ ] Integration test: pre-register visitor; verify `VisitorProfile` created; pre-register same visitor again → same `visitorProfileId` returned.

---

## API Contract

```
POST /api/v1/visitors/pre-register
Request:  { "visitorEmail": "jane@acme.com", "visitorName": "Jane Doe",
            "hostUserId": "uuid", "locationId": "uuid",
            "expectedDate": "2026-06-15", "purposeOfVisit": "Client meeting" }
Response 201: { "success": true, "data": { "visitId": "uuid", "visitorProfileId": "uuid", "status": "EXPECTED" } }

POST /api/v1/visitors/walk-in
Request:  { "visitorEmail": "john@vendor.com", "visitorName": "John Smith",
            "hostUserId": "uuid", "locationId": "uuid", "purposeOfVisit": "Delivery" }
Response 201: { "success": true, "data": { "visitId": "uuid", "visitorProfileId": "uuid",
                                            "status": "CHECKED_IN", "checkInAt": "2026-06-15T09:32:00Z" } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] `VisitorProfile` reuse verified (same email → same profileId)
- [ ] Role guard: walk-in only by `FACILITIES_ADMIN`
- [ ] Audit log entry on `VISITOR_REGISTERED` and `VISITOR_WALK_IN`
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green

---

## Dependencies

- `AOMS-33` (check-in/out) depends on `ParentVisit` created here.
- `AOMS-34` (agreement templates) must exist for check-in to validate agreement version.
