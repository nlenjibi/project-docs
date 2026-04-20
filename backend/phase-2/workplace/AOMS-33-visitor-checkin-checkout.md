# AOMS-33 — Visitor Check-in / Check-out + Agreement Version Enforcement (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-33 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | workplace, visitor, agreement, api |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As a Facilities Admin, I want to check visitors in and out so that we know who is on-site at any moment, and so that every visitor has signed the current NDA version before entering.

---

## Acceptance Criteria

- [ ] `PATCH /api/v1/visits/{visitId}/check-in` — checks in a pre-registered visitor.
  - Accessible to `FACILITIES_ADMIN` only.
  - Validates `ParentVisit.status = EXPECTED` — return 409 if already `CHECKED_IN` or `COMPLETED`.
  - Retrieves the active `AgreementTemplate` version. Compares against `VisitorProfile.last_signed_version`.
    - If versions differ OR `last_signed_version` is null: return `requiresSignature: true` with `templateId` and `templateContent`. Check-in does NOT complete until signature is submitted.
    - If versions match: check-in completes immediately.
  - On check-in completion: creates `VisitRecord` with `check_in_at = now()`, `agreement_template_version_signed = activeVersion`. Updates `ParentVisit.status = CHECKED_IN`. Updates `VisitorProfile.last_signed_version`.
  - Publishes `oms.workplace.visit.checked_in` → notification-service notifies host employee (in-app + email).
- [ ] `PATCH /api/v1/visits/{visitId}/sign-agreement` — submits visitor agreement signature.
  - Body: `{ templateId, signatureData (base64 PNG or acknowledgement boolean) }`.
  - Called immediately before check-in when `requiresSignature: true` is returned.
  - After signature recorded: triggers check-in completion.
- [ ] `PATCH /api/v1/visits/{visitId}/check-out` — checks out a visitor.
  - Accessible to `FACILITIES_ADMIN` only.
  - Validates `ParentVisit.status = CHECKED_IN` — return 409 if not checked in.
  - Sets `VisitRecord.check_out_at = now()`. Updates `ParentVisit.status = COMPLETED`.
  - Publishes `oms.workplace.visit.checked_out` → audit-service.
- [ ] `GET /api/v1/visits/location/{locationId}/active` — returns all currently checked-in visitors (status = `CHECKED_IN`) at the location. Accessible to `FACILITIES_ADMIN`, `HR`.
- [ ] Unit tests: check-in with stale agreement version → `requiresSignature: true`; check-in when already checked in → 409; check-out when not checked in → 409; fresh visitor (no prior visit) → requires signature.
- [ ] Integration test: pre-register visitor (AOMS-32) → check-in → verify `VisitRecord` created with agreement version; update agreement template → check-in same visitor → `requiresSignature: true`.

---

## API Contract

```
PATCH /api/v1/visits/{visitId}/check-in
Response 200 (signature needed): { "success": true, "data": { "requiresSignature": true,
  "templateId": "uuid", "templateContent": "NDA text..." } }
Response 200 (check-in complete): { "success": true, "data": { "visitId": "uuid",
  "checkInAt": "2026-06-15T09:35:00Z", "agreementVersionSigned": 3 } }

PATCH /api/v1/visits/{visitId}/check-out
Response 200: { "success": true, "data": { "visitId": "uuid", "checkOutAt": "2026-06-15T17:10:00Z" } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Agreement version check working: stale version blocks completion; fresh version allows immediate entry
- [ ] `VisitorProfile.last_signed_version` updated on every successful check-in
- [ ] `oms.workplace.visit.checked_in` event published and verified by integration test
- [ ] Audit log entries for `VISITOR_CHECKED_IN` and `VISITOR_CHECKED_OUT`
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-32** — `ParentVisit` must exist before check-in can proceed.
- **AOMS-34** — active `AgreementTemplate` must exist for version comparison.
- **ATT-P10** (In-App Notifications) — host employee notification on check-in.
