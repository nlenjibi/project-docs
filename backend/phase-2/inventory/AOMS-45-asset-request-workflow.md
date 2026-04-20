# AOMS-45 — Asset Request and Fulfilment Workflow (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-45 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | inventory, assets, workflow, api |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As an employee, I want to request an asset type so my manager can approve and Facilities Admin can assign a specific unit from the register.

---

## Acceptance Criteria

- [ ] `POST /api/v1/asset-requests` — employee submits an asset request.
  - Request: `{ categoryId, locationId, justification }`.
  - Creates `AssetRequest` with `status = PENDING_MANAGER`.
  - Publishes `oms.inventory.supply.request.submitted` (reused event type for both supplies and assets, distinguished by `entityType` field).
- [ ] `PATCH /api/v1/asset-requests/{id}/approve` — manager approves.
  - Sets `status = PENDING_FACILITIES`. Notifies Facilities Admin (in-app + email).
- [ ] `PATCH /api/v1/asset-requests/{id}/reject` — manager rejects with mandatory `reason`.
  - Sets `status = REJECTED`. Notifies requesting employee.
- [ ] `PATCH /api/v1/asset-requests/{id}/fulfil` — Facilities Admin fulfils by selecting a specific asset.
  - Request: `{ assetId }` — Facilities Admin picks the specific unit from the available register.
  - Validates: `AssetRequest.status = PENDING_FACILITIES`; selected `Asset.status = AVAILABLE`; `Asset.locationId = AssetRequest.locationId`.
  - Calls the same direct assignment logic from AOMS-44 (`POST /api/v1/asset-assignments` internally). Sets `AssetRequest.status = FULFILLED`.
  - Publishes `oms.inventory.asset.assigned` → employee notified.
- [ ] `GET /api/v1/asset-requests/my` — own request history, paginated.
- [ ] `GET /api/v1/asset-requests?locationId={id}&status={s}` — admin/manager view.
- [ ] Unit tests: fulfil without manager approval → 409; fulfil with unavailable asset → 400; reject at stage 2 by manager → 403 (manager can only reject at stage 1).

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] Two-stage guard enforced: PENDING_FACILITIES required for fulfilment
- [ ] Internal direct assignment (AOMS-44) called atomically from fulfilment — no partial state
- [ ] Notification events published at each stage change
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-44** — direct asset assignment logic called internally from fulfilment.
