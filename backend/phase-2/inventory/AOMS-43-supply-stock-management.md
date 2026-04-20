# AOMS-43 — Supply Stock and Batch Management (FIFO + Reorder) (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-43 |
| **Epic** | Inventory Service — AOMS-40 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | inventory, supplies, stock, api |
| **Service** | `inventory-service` (Spring Boot / Java) |

---

## Story

As a Facilities Admin, I want to add stock batches with expiry dates and write off expired stock so the inventory is always accurate and expired items are never fulfilled.

---

## Acceptance Criteria

- [ ] `POST /api/v1/supplies/stock-entries` — adds a new stock batch.
  - Accessible to `FACILITIES_ADMIN`.
  - Request: `{ itemId, quantity, expiryDate (nullable for non-perishables), receivedAt, batchReference }`.
  - Creates a `SupplyStockEntry`. Stock is available immediately for fulfilment.
- [ ] `PATCH /api/v1/supplies/stock-entries/{id}/write-off` — writes off an expired or damaged batch.
  - Accessible to `FACILITIES_ADMIN`.
  - Request: `{ reason }`.
  - Sets `SupplyStockEntry.written_off_at = now()` and `quantity_remaining = 0`. Batch excluded from future FIFO fulfilment.
  - After write-off: recalculate aggregate stock; if below `reorderThreshold` → publish `oms.inventory.stock.low`.
- [ ] `GET /api/v1/supplies/inventory-report?locationId={id}` — stock levels per item.
  - Accessible to `HR`, `FACILITIES_ADMIN`.
  - Response per item: `itemName`, `totalQuantityAvailable` (sum of non-expired, non-written-off batches), `reorderThreshold`, `batches []` (batchId, quantity, expiryDate, receivedAt, status).
  - Items below `reorderThreshold` flagged: `belowThreshold: true`.
- [ ] FIFO ordering: `SupplyStockEntry` sorted by `received_at ASC` then `expiry_date ASC` for fulfilment. Entries with `written_off_at IS NOT NULL` or `expiry_date < today` are skipped.
- [ ] Unit tests: add batch → available immediately; write off batch → excluded from FIFO; all batches expired → fulfilment returns `INSUFFICIENT_STOCK`; write-off below threshold → `oms.inventory.stock.low` published.
- [ ] Integration test: add two batches (older, newer); request fulfilment → older batch decremented first; write off older → only newer batch available.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`; unit tests passing; coverage ≥ 80%
- [ ] FIFO ordering verified with multi-batch integration test
- [ ] Expired batches excluded from fulfilment (date check at query time, not application layer)
- [ ] `oms.inventory.stock.low` and `oms.inventory.reorder.triggered` published on threshold breach
- [ ] Swagger annotations complete; Jenkins pipeline green

---

## Dependencies

- **AOMS-41** — `SupplyItem` must exist before stock entries can reference it.
- **AOMS-42** — fulfilment logic reads `SupplyStockEntry` batches ordered by FIFO rules defined here.
