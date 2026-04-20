# AOMS-44 — Asset Register and Direct Assignment (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-44 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | inventory, assets, register, api |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As a Facilities Admin, I want to maintain an asset register and directly assign assets to employees so HR and Facilities always know what equipment is allocated and where.

---

## Acceptance Criteria

- [ ] `POST /api/v1/assets` — registers a new asset. Accessible to `FACILITIES_ADMIN`.
  - Request: `{ categoryId, name, serialNumber, locationId, purchaseDate, warrantyExpiryDate }`.
  - Creates `Asset` with `status = AVAILABLE`.
  - `serialNumber` must be unique within `locationId` — return 409 on duplicate.
- [ ] `GET /api/v1/assets?locationId={id}&status={s}&categoryId={c}` — asset register. Accessible to `HR`, `FACILITIES_ADMIN`.
- [ ] `GET /api/v1/assets/{assetId}` — asset detail with current assignment and open fault reports. Accessible to `HR`, `FACILITIES_ADMIN`.
- [ ] `GET /api/v1/assets/my` — own currently assigned assets. Accessible to all employees.
- [ ] `POST /api/v1/asset-assignments` — Facilities Admin directly assigns an asset to an employee.
  - Request: `{ assetId, userId, notes }`.
  - Asset must have `status = AVAILABLE` — return 409 if `ASSIGNED`, `UNDER_MAINTENANCE`, or `RETIRED`.
  - Creates `AssetAssignment` with `status = ASSIGNED`. Sets `Asset.status = ASSIGNED`.
  - Publishes `oms.inventory.asset.assigned` → notification-service notifies employee.
- [ ] `PATCH /api/v1/asset-assignments/{id}/acknowledge` — employee acknowledges receipt.
  - Accessible to the assigned employee only.
  - Sets `AssetAssignment.status = ACKNOWLEDGED` and `acknowledged_at = now()`.
  - Publishes `oms.inventory.asset.acknowledged` → audit-service.
- [ ] `PATCH /api/v1/asset-assignments/{id}/return` — employee or Facilities Admin marks asset as returned.
  - Sets `AssetAssignment.status = RETURNED` and `returned_at = now()`. Sets `Asset.status = AVAILABLE`.
  - Asset becomes available for re-assignment immediately.
- [ ] `PATCH /api/v1/assets/{assetId}/retire` — Facilities Admin retires an asset permanently.
  - Returns 409 if asset has `status = ASSIGNED` (must be returned first).
  - Sets `Asset.status = RETIRED`. Publishes `oms.inventory.asset.retired`.
- [ ] Unit tests: assign `ASSIGNED` asset → 409; retire `ASSIGNED` asset → 409; return asset → status becomes `AVAILABLE`; acknowledge by wrong employee → 403.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] `serialNumber` uniqueness enforced at DB unique constraint
- [ ] Asset status state machine transitions enforced (`AVAILABLE → ASSIGNED → RETURNED|RETIRED`)
- [ ] `oms.inventory.asset.assigned` event published; employee notified (in-app + email)
- [ ] Audit log entries for all assignment lifecycle events
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-45** — asset request workflow builds on the direct assignment flow defined here.
- **AOMS-46** — fault reporting sets `Asset.status = UNDER_MAINTENANCE` for HIGH/CRITICAL faults.
