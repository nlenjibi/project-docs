# AOMS-42 — Supply Request Two-Stage Approval Workflow (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-42 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 5 |
| **Phase** | Phase 2 |
| **Labels** | inventory, supplies, workflow, api |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As an employee, I want to request supplies from the office catalogue so my manager can approve and Facilities Admin can fulfil the request from stock.

---

## Acceptance Criteria

- [ ] `POST /api/v1/supply-requests` — employee submits a supply request.
  - Request: `{ itemId, quantity, locationId, justification }`.
  - Creates `SupplyRequest` with `status = PENDING_MANAGER`.
  - Publishes `oms.inventory.supply.request.submitted` → audit-service.
  - Validates: item must exist and be active at `locationId`; quantity > 0.
- [ ] `PATCH /api/v1/supply-requests/{id}/approve-stage1` — manager approves.
  - Accessible to `MANAGER` (must be requesting employee's direct manager), `HR`, `SUPER_ADMIN`.
  - Sets `status = PENDING_FACILITIES`.
  - Publishes `oms.inventory.supply.request.approved` → notification-service notifies Facilities Admin (in-app + email).
- [ ] `PATCH /api/v1/supply-requests/{id}/reject-stage1` — manager rejects.
  - Request: `{ reason }` (required).
  - Sets `status = REJECTED`. Publishes `oms.inventory.supply.request.rejected` → notification-service notifies employee.
- [ ] `PATCH /api/v1/supply-requests/{id}/fulfil` — Facilities Admin fulfils the request.
  - Accessible to `FACILITIES_ADMIN`.
  - Must be `status = PENDING_FACILITIES` — return 409 otherwise.
  - Decrements stock from `SupplyStockEntry` using FIFO (oldest non-expired batch first). If insufficient stock → return 400 with `INSUFFICIENT_STOCK`.
  - After decrement: if aggregate remaining stock < `reorderThreshold` → publish `oms.inventory.stock.low` AND `oms.inventory.reorder.triggered`.
  - Sets `status = FULFILLED`. Publishes `oms.inventory.supply.request.fulfilled` → notification-service notifies employee.
- [ ] `PATCH /api/v1/supply-requests/{id}/reject-stage2` — Facilities Admin rejects at fulfilment stage.
  - Sets `status = REJECTED`. Reason required.
- [ ] `GET /api/v1/supply-requests/my` — employee's own request history, paginated, filterable by status and date.
- [ ] `GET /api/v1/supply-requests?locationId={id}&status={s}` — admin/manager view. Accessible to `MANAGER`, `HR`, `FACILITIES_ADMIN`.
- [ ] Unit tests: fulfil without stage1 approval → 409; fulfil with insufficient stock → 400; FIFO stock verified (oldest batch decremented first); reorder triggered when stock drops below threshold.
- [ ] Integration test: submit request → manager approves → Facilities Admin fulfils → verify stock decremented; verify `oms.inventory.supply.request.fulfilled` published.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] Two-stage guard enforced: PENDING_FACILITIES required for fulfilment
- [ ] FIFO stock decrement tested with multi-batch fixture
- [ ] Low-stock and reorder events published at correct threshold
- [ ] All notification events published and consumed by notification-service (integration test)
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-41** — supply items and categories must exist.
- **AOMS-43** — `SupplyStockEntry` batches must exist for FIFO decrement to work.
- **ATT-P10** (In-App Notifications) for employee and Facilities Admin notifications.
