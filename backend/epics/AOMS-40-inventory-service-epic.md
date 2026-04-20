# AOMS-40 — Inventory Service (Epic)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-40 |
| **Type** | Epic |
| **Priority** | 🟡 High |
| **Phase** | Phase 2 |
| **Labels** | inventory, supplies, assets, epic |
| **Service** | `inventory-service` (Spring Boot / Java · Port 8090) |

---

## Epic Summary

Track all physical items at each office location through their full lifecycle. **Supplies** are consumable items (stationery, coffee, cleaning products) managed through a two-stage approval workflow (Manager → Facilities Admin) with FIFO batch stock and automated low-stock reorder triggers. **Assets** are non-consumable items (laptops, monitors, phones) managed through an assignment workflow with fault reporting and maintenance tracking.

`inventory-service` owns `inventory_db` and is the single authority for all physical item state at a location.

---

## Business Goals

- Give employees a self-service way to request supplies and assets without emailing Facilities Admin directly.
- Enforce two-stage approval (manager review, then Facilities Admin fulfilment) so requests are authorised at team level before consuming stock.
- Track stock levels per supply item per batch with expiry dates so expired stock is never used and reorder happens automatically at threshold.
- Maintain a complete asset register with assignment history so HR and Facilities Admin always know which employee holds which asset.
- Surface fault reports so maintenance issues are tracked and escalated before assets fail in the field.

---

## Stories

### Supply Management

| ID | Title | Points |
|---|---|---|
| AOMS-41 | Supply Catalogue Management | 2 |
| AOMS-42 | Supply Request Two-Stage Approval Workflow | 5 |
| AOMS-43 | Supply Stock and Batch Management (FIFO + Reorder) | 3 |

### Asset Management

| ID | Title | Points |
|---|---|---|
| AOMS-44 | Asset Register and Direct Assignment | 3 |
| AOMS-45 | Asset Request and Fulfilment Workflow | 3 |
| AOMS-46 | Asset Fault Reporting and Maintenance Records | 2 |
| AOMS-47 | Asset Lifecycle Events and Audit Trail | 2 |

**Total: 7 stories · 20 story points**

---

## Data Model Overview

```
Supply side:
  SupplyCategory → SupplyItem (reorder_threshold, location_id)
    └── SupplyStockEntry (batch_id, quantity, expiry_date, received_at)  — FIFO fulfilment
  SupplyRequest (user_id, item_id, quantity, status: PENDING_MANAGER → PENDING_FACILITIES → FULFILLED | REJECTED)
  ReorderLog (item_id, triggered_at, quantity_requested, location_id)

Asset side:
  AssetCategory → Asset (serial_number, status: AVAILABLE|ASSIGNED|RETIRED, location_id)
    └── AssetAssignment (user_id, assigned_at, acknowledged_at, returned_at,
                         status: ASSIGNED|ACKNOWLEDGED|RETURNED|RETIRED)
  AssetRequest (user_id, category_id, status: PENDING_MANAGER → PENDING_FACILITIES → FULFILLED | REJECTED)
  FaultReport (asset_id, reported_by, severity: LOW|MEDIUM|HIGH|CRITICAL, description, status)
  MaintenanceRecord (asset_id, type: SCHEDULED|AD_HOC, performed_at, notes)
```

---

## Key Constraints

- **FIFO fulfilment:** Supply stock is decremented from the earliest non-expired `SupplyStockEntry` first. Expired batches are never used — they must be written off explicitly by Facilities Admin.
- **Two-stage approval — both stages required:** A supply or asset request cannot reach `FULFILLED` without both manager approval and Facilities Admin fulfilment confirmation. Bypassing either stage is not possible.
- **Low-stock trigger:** After each fulfilment, if aggregate remaining stock (across all non-expired batches) drops below `reorder_threshold`, the service publishes both `oms.inventory.stock.low` (notification) AND `oms.inventory.reorder.triggered` (procurement integration hook).
- **Asset status integrity:** An asset with status `ASSIGNED` cannot be assigned to another employee. It must be returned first (`RETURNED`) before reassignment. Assets with open `FaultReport` of severity `HIGH|CRITICAL` are flagged as `UNDER_MAINTENANCE` and cannot be assigned.
- **Location scoping:** All queries are filtered by `location_id`. An employee at Location A cannot see inventory from Location B.

---

## RabbitMQ Events Produced

```
oms.inventory.supply.request.submitted  → audit-service
oms.inventory.supply.request.approved   → notification-service → manager-to-facilities stage notification
oms.inventory.supply.request.fulfilled  → notification-service → employee notified (in-app + email)
oms.inventory.supply.request.rejected   → notification-service → employee notified (in-app + email)
oms.inventory.stock.low                 → notification-service → Facilities Admin notified (in-app + email)
oms.inventory.reorder.triggered         → future procurement integration hook
oms.inventory.asset.assigned            → notification-service → employee notified (in-app + email)
oms.inventory.asset.acknowledged        → audit-service
oms.inventory.asset.returned            → audit-service
oms.inventory.fault.reported            → notification-service → Facilities Admin notified (in-app)
oms.inventory.maintenance.logged        → audit-service
oms.inventory.asset.retired             → audit-service
```

---

## Acceptance at Epic Level

- [ ] Facilities Admin can manage supply catalogue per location
- [ ] Employee can request supplies; manager approves stage 1; Facilities Admin fulfils stage 2
- [ ] FIFO stock correctly decremented from oldest non-expired batch on fulfilment
- [ ] Low-stock alert triggered when aggregate stock drops below threshold
- [ ] Asset register maintained; employee can view own assigned assets
- [ ] Employee can request asset; two-stage approval identical to supply request
- [ ] Employee can submit fault report; Facilities Admin notified
- [ ] Maintenance records logged against asset; full audit trail queryable
- [ ] All state-changing operations write an audit entry via RabbitMQ

---

## Dependencies

- `ATT-P02` (Role + Location Interceptor) on all endpoints
- `ATT-P03` (Audit Log) wired to all state changes
- `ATT-P10` (In-App Notifications) for all inventory notification events
- `ATT-P07` (SES Email) for email channel notifications
- `ATT-P08` (Cloud Map) for service resolution
- Future: procurement system integration for `oms.inventory.reorder.triggered` (blocked on AD-026)
