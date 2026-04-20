# AOMS-41 — Supply Catalogue Management (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-41 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 2 |
| **Phase** | Phase 2 |
| **Labels** | inventory, supplies, catalogue, api |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As a Facilities Admin, I want to manage the supply catalogue so employees can browse available items and I can track what is stocked at each location.

---

## Acceptance Criteria

- [ ] `GET /api/v1/supplies/catalogue?locationId={id}` — returns all active `SupplyItem`s at a location, grouped by `SupplyCategory`. Accessible to all roles.
- [ ] `POST /api/v1/supplies/catalogue` — creates a new supply item. Accessible to `FACILITIES_ADMIN` only.
  - Request: `{ categoryId, name, description, unit, reorderThreshold, locationId }`.
  - `reorderThreshold` is the minimum aggregate non-expired stock below which a reorder alert fires.
- [ ] `PATCH /api/v1/supplies/catalogue/{itemId}` — updates item details or `reorderThreshold`. Accessible to `FACILITIES_ADMIN`.
- [ ] `DELETE /api/v1/supplies/catalogue/{itemId}` — soft-deletes an item (`is_active = false`). Returns 409 if item has pending unfulfilled `SupplyRequest`s.
- [ ] `POST /api/v1/supplies/categories` — creates a `SupplyCategory` (e.g. "Stationery", "Beverages"). Accessible to `FACILITIES_ADMIN`.
- [ ] Unit tests: create item with non-existent category → 400; soft-delete item with pending request → 409; employee browsing catalogue → 200 with only active items.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] `is_active` soft-delete enforced — deleted items excluded from employee catalogue
- [ ] Audit log entry on `SUPPLY_ITEM_CREATED` and `SUPPLY_ITEM_UPDATED`
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-42** (supply requests) depends on items created here.
- **AOMS-43** (stock management) depends on `SupplyItem.reorderThreshold`.
