# AOMS-46 — Asset Fault Reporting and Maintenance Records (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-46 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 2 |
| **Phase** | Phase 2 |
| **Labels** | inventory, assets, maintenance, api |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As an employee, I want to report a fault on my assigned asset so Facilities Admin is alerted and can schedule maintenance before the issue worsens.

---

## Acceptance Criteria

- [ ] `POST /api/v1/fault-reports` — employee submits a fault report.
  - Request: `{ assetId, description, severity: LOW|MEDIUM|HIGH|CRITICAL }`.
  - Validates: reporter must have an active `AssetAssignment` for this asset OR be `FACILITIES_ADMIN`/`HR`.
  - Creates `FaultReport` with `status = OPEN`.
  - If severity is `HIGH` or `CRITICAL`: sets `Asset.status = UNDER_MAINTENANCE`. Asset cannot be assigned until fault is resolved.
  - Publishes `oms.inventory.fault.reported` → notification-service notifies Facilities Admin (in-app only).
- [ ] `PATCH /api/v1/fault-reports/{id}/resolve` — Facilities Admin resolves the fault.
  - Accessible to `FACILITIES_ADMIN`.
  - Sets `FaultReport.status = RESOLVED` and `resolved_at = now()`.
  - If no other open `HIGH|CRITICAL` faults remain for this asset: sets `Asset.status = AVAILABLE`.
- [ ] `GET /api/v1/fault-reports?assetId={id}` — fault history for an asset. Accessible to `HR`, `FACILITIES_ADMIN`.
- [ ] `POST /api/v1/maintenance-records` — Facilities Admin logs a maintenance event.
  - Request: `{ assetId, maintenanceType: SCHEDULED|AD_HOC, performedAt, notes, nextMaintenanceDate }`.
  - Creates `MaintenanceRecord`. Publishes `oms.inventory.maintenance.logged` → audit-service.
- [ ] `GET /api/v1/assets/{id}/audit-log` — chronological list of assignments, fault reports, and maintenance records for an asset. Accessible to `HR`, `FACILITIES_ADMIN`.
- [ ] Unit tests: HIGH fault → asset set to `UNDER_MAINTENANCE`; resolve last HIGH fault → asset back to `AVAILABLE`; LOW fault does not change asset status; non-assigned employee reports fault → 403.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] `UNDER_MAINTENANCE` status blocks assignment (validated in AOMS-44 guard)
- [ ] Multi-fault scenario: two HIGH faults; resolve one; asset remains `UNDER_MAINTENANCE`; resolve both; asset becomes `AVAILABLE`
- [ ] `oms.inventory.fault.reported` event published; Facilities Admin notified
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-44** — asset status transitions defined there; `UNDER_MAINTENANCE` guard enforced in assignment flow.
