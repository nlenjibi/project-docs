# AOMS-47 — Asset Lifecycle Events and Audit Trail (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-47 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 2 |
| **Phase** | Phase 2 |
| **Labels** | inventory, assets, audit, lifecycle |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As HR or a Facilities Admin, I want to view the complete chronological history of any asset so I can trace its ownership chain, maintenance history, and fault incidents for compliance purposes.

---

## Acceptance Criteria

- [ ] `GET /api/v1/assets/{assetId}/audit-log` — returns the full chronological timeline for a single asset, newest first.
  - Accessible to `HR`, `FACILITIES_ADMIN`, `SUPER_ADMIN`.
  - Each entry includes: `eventType` (one of: `REGISTERED`, `ASSIGNED`, `ACKNOWLEDGED`, `RETURNED`, `FAULT_REPORTED`, `FAULT_RESOLVED`, `MAINTENANCE_LOGGED`, `RETIRED`), `timestamp`, `actorId`, `actorName`, `details` (JSON object with event-specific fields).
  - Derived from the combination of `AssetAssignment` records, `FaultReport` records, `MaintenanceRecord` records, and the original `Asset.created_at`.
- [ ] Every state-changing operation across AOMS-44, AOMS-45, AOMS-46 publishes `oms.audit.event` to `audit-service` with `entityType = Asset` and `entityId = assetId`. This covers: `ASSET_REGISTERED`, `ASSET_ASSIGNED`, `ASSET_ACKNOWLEDGED`, `ASSET_RETURNED`, `ASSET_RETIRED`, `FAULT_REPORTED`, `FAULT_RESOLVED`, `MAINTENANCE_LOGGED`.
- [ ] `GET /api/v1/assets/{assetId}/audit-log` builds its timeline from `inventory_db` local records only — it does NOT query `audit-service`. `audit-service` receives events for cross-system audit compliance; the local read is for operational use.
- [ ] `GET /api/v1/assets?locationId={id}&status=RETIRED` — returns retired asset register for record-keeping. Retired assets remain in the register permanently (never hard-deleted).
- [ ] Unit tests: asset with full lifecycle (registered → assigned → returned → fault → maintenance → retired) → timeline has 7 entries in correct chronological order.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] All 8 `oms.audit.event` messages confirmed published for asset lifecycle operations
- [ ] Timeline endpoint returns entries from local DB (no cross-service call at query time)
- [ ] Retired assets never hard-deleted — confirmed in soft-delete implementation
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-44**, **AOMS-45**, **AOMS-46** — all asset state-changing operations must publish `oms.audit.event`.
- `ATT-P03` (Audit Log) — `audit-service` must be consuming `oms.audit.event` for cross-system compliance.
