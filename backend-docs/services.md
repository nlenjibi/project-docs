# Service Definitions
## Office Management System (OMS)

All 11 services follow the same structural, security, and operational standards. Each service:
- Owns exactly one business capability
- Has its own PostgreSQL database — never accessed by another service
- Validates every inbound request (internal JWT + role/location check)
- Publishes `audit.event` to Kafka/SQS on every state-changing operation

---

## 1. identity-service (Port 8081)

**Responsibility:** SSO integration, OIDC token verification, session issuance, and session lifecycle management.

**Owned Entities:**
- `Session` — stores HTTP-only session cookies; maps to OMS user identity

**APIs:**
```
POST  /api/v1/auth/login        → Initiates SSO redirect
GET   /api/v1/auth/callback     → Exchanges OIDC code; verifies JWKS; issues session cookie
POST  /api/v1/auth/logout       → Invalidates session; clears cookie
GET   /api/v1/auth/validate     → Internal — called by API Gateway on every inbound request
GET   /api/v1/auth/me           → Returns { userId, email, roles[] } from session context
```

**Rules:**
- Token verification is entirely server-side. The frontend never handles raw tokens.
- Sessions are HTTP-only, `SameSite=Strict` cookies — JavaScript cannot access them.
- `spring-session-jdbc` stores sessions in `identity_db`, keeping pods stateless.
- Calls `user-service` via REST to resolve roles after identity verification.
- Produces no Kafka events.

**Key Config:**
```bash
SSO_JWKS_URI=https://sso.yourorg.com/.well-known/jwks.json
SSO_ISSUER_URI=https://sso.yourorg.com
SESSION_EXPIRY_SECONDS=3600
```

**Database:** `identity_db` — minimal schema; `sessions` table only.

---

## 2. user-service (Port 8082)

**Responsibility:** User profiles, per-location role assignments, location configuration, and Arms HR system synchronisation.

**Owned Entities:**
- `User` — profile, SSO sub, employment dates
- `UserRole` — user + role enum + location_id (NULL = Super Admin; cross-location)
- `Location` — office locations
- `LocationConfig` — per-location settings (timezone, booking window, no-show cutoff, etc.)
- `PublicHoliday` — per-location public holiday calendar

**APIs:**
```
GET    /api/v1/users/{id}                     → User profile
GET    /api/v1/users/{id}/roles               → User roles across all locations
GET    /api/v1/locations                      → List all locations
GET    /api/v1/locations/{id}/config          → Location configuration
PATCH  /api/v1/locations/{id}/config          → Update location config (Super Admin)
POST   /api/v1/users/{id}/roles               → Assign role (Super Admin)
DELETE /api/v1/users/{id}/roles/{roleId}      → Revoke role (Super Admin)
```

**Scheduled Jobs:**
- `HrSyncJob` — polls the Arms HR API on a configurable schedule; upserts `User` records and `UserRole` assignments idempotently; emits events on Kafka.

**Kafka Events Produced:**
```
oms.user.user.created      → { userId, locationId, employmentStartDate }
oms.user.user.updated      → { userId, changedFields }
oms.user.user.deactivated  → { userId, employmentEndDate }
oms.user.sync.completed    → { syncedCount, failedCount, timestamp }
```

**Database:** `user_db`

---

## 3. attendance-service (Port 8083)

**Responsibility:** Nightly badge event ingestion from AWS Athena, WorkSession resolution, and two-pass AttendanceRecord computation.

**Owned Entities:**
- `BadgeEvent` — immutable after ingestion; one row per badge swipe
- `WorkSession` — resolved from badge events; captures first_badge_in, last_badge_out, duration
- `AttendanceRecord` — final daily status per user per date: `PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY`
- `BadgeSyncJobLog` — execution log per location per run

**Attendance Pipeline:**
```
AWS Athena (badge_events table)
  ↓ nightly BadgeSyncJob (2am, configurable)
BadgeEvent records (immutable)
  ↓ WorkSession resolution
WorkSession (first_badge_in, last_badge_out, is_late, crosses_midnight)
  ↓ Pass 1
AttendanceRecord: PRESENT or LATE
  ↓ Pass 2 (Kafka consumers: remote.request.approved, ooo.request.approved)
Final AttendanceRecord: PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY
```

**APIs:**
```
GET  /api/v1/attendance/{userId}                 → Own attendance history (Employee)
GET  /api/v1/attendance/team/{managerId}         → Team attendance (Manager / HR)
GET  /api/v1/attendance/location/{locationId}    → Location-wide (HR / Super Admin)
POST /api/v1/attendance/{id}/override            → Override record (Super Admin)
GET  /api/v1/attendance/reports/no-shows         → No-show report
GET  /api/v1/attendance/export                   → CSV / Excel export
GET  /api/v1/attendance/sync/logs                → Sync job logs (Super Admin, Facilities Admin)
GET  /api/v1/attendance/sync/logs/{jobId}        → Single job log
POST /api/v1/attendance/sync/trigger             → Manual sync trigger (Super Admin)
```

**Kafka Events Consumed:**
```
oms.remote.request.approved   → Overlay REMOTE status for affected user-dates
oms.ooo.request.approved      → Overlay OOO status for affected user-dates
oms.user.user.deactivated     → Stop generating AttendanceRecords after employmentEndDate
```

**Kafka Events Produced:**
```
oms.attendance.record.resolved   → { userId, recordDate, status, locationId }
oms.attendance.no_show.detected  → { userId, date, locationId }
```

**Key Config:**
```bash
ATHENA_DATABASE=badge_events
ATHENA_OUTPUT_BUCKET=s3://oms-athena-results/
AWS_REGION=eu-west-1
BADGE_SYNC_CRON=0 2 * * *
ATHENA_QUERY_TIMEOUT_SECONDS=120
SESSION_GAP_THRESHOLD_HOURS=6
```

**Database:** `attendance_db`

---

## 4. seating-service (Port 8084)

**Responsibility:** Floor plan management, real-time hot-desk booking, permanent seat assignment, manager block reservations, and no-show auto-release.

**Owned Entities:**
- `Floor`, `Zone`, `Seat` — spatial hierarchy
- `SeatBooking` — individual hot-desk reservations; statuses: `CONFIRMED / CANCELLED / RELEASED`
- `BlockReservation` — manager-reserved seat groups
- `NoShowRecord` — log of unreturned bookings released by the no-show job

**Seat Availability:** Computed in real-time from `seat_bookings` records filtered by `(booking_date, location_id)` with a compound index. No pre-computed cache — queries are sub-200ms at current scale.

**APIs:**
```
GET    /api/v1/locations/{id}/floor-plan      → Floor plan with real-time seat availability
POST   /api/v1/seat-bookings                  → Book a hot-desk (Employee)
DELETE /api/v1/seat-bookings/{id}             → Cancel booking (Employee)
GET    /api/v1/seat-bookings/my               → Own bookings
POST   /api/v1/seat-bookings/block            → Block-book seats for a team (Manager)
POST   /api/v1/seats/{id}/permanent           → Assign permanent seat (HR / Facilities Admin)
DELETE /api/v1/seats/{id}/permanent           → Release permanent seat
PUT    /api/v1/locations/{id}/floor-plan      → Configure floor plan (Facilities Admin)
```

**Scheduled Jobs:**
- `NoShowReleaseJob` — runs daily at `no_show_release_time` (per location config); queries CONFIRMED bookings with no badge-in by cutoff; marks them RELEASED; creates `NoShowRecord`.

**Kafka Events Consumed:**
```
oms.attendance.no_show.detected  → Trigger no-show release check
oms.user.user.deactivated        → Cancel all future bookings for the deactivated user
```

**Kafka Events Produced:**
```
oms.seating.booking.created    → { bookingId, userId, seatId, date, locationId }
oms.seating.booking.cancelled  → { bookingId, userId, locationId }
oms.seating.booking.released   → { bookingId, userId, reason: NO_SHOW, locationId }
```

**Key Config:**
```bash
BOOKING_WINDOW_DAYS=14
CANCELLATION_CUTOFF_HOURS=2
NO_SHOW_RELEASE_CRON=0 10 * * *
```

**Database:** `seating_db`

---

## 5. remote-service (Port 8085)

**Responsibility:** Remote day and OOO request workflow, configurable policy enforcement (`HARD_BLOCK` / `SOFT_WARNING`), manager approval delegation, and recurring schedule management.

**Owned Entities:**
- `RemoteRequest` — individual remote-day request (child of a RecurringRemoteSchedule or standalone)
- `RecurringRemoteSchedule` — parent record for recurring patterns
- `OOORequest` — out-of-office request
- `ApprovalDelegate` — nominee who approves when the manager is OOO
- `RemoteDayPolicy` — configurable limits per team / location

**Policy Logic:**
- Checks team-level `RemoteDayPolicy` first; falls back to location default.
- `HARD_BLOCK` → rejects the submission if the weekly limit is exceeded or team overlap threshold is breached.
- `SOFT_WARNING` → allows submission with a warning flag in the response body.
- If a manager goes OOO without nominating a delegate, the HR role at that location becomes the fallback approver.

**APIs:**
```
POST   /api/v1/remote-requests                      → Submit remote-day request (Employee)
POST   /api/v1/ooo-requests                         → Submit OOO request (Employee)
PATCH  /api/v1/remote-requests/{id}/approve         → Approve (Manager / Delegate)
PATCH  /api/v1/remote-requests/{id}/reject          → Reject with reason (Manager / Delegate)
GET    /api/v1/teams/{managerId}/schedule            → Team schedule calendar view (Manager)
POST   /api/v1/approval-delegates                   → Nominate approval delegate (Manager)
GET    /api/v1/remote-day-policies/{locationId}     → Get location-level policy
PATCH  /api/v1/remote-day-policies/{managerId}      → Override team policy (HR / Super Admin)
POST   /api/v1/remote-requests/{id}/override        → Override past dates (Super Admin)
GET    /api/v1/remote-requests/history              → Own request history (Employee)
```

**Kafka Events Produced:**
```
oms.remote.request.submitted    → { requestId, userId, dates, locationId }
oms.remote.request.approved     → { requestId, userId, dates, locationId }
oms.remote.request.rejected     → { requestId, userId, reason, locationId }
oms.ooo.request.approved        → { requestId, userId, startDate, endDate, locationId }
```

**Database:** `remote_db`

---

## 6. notification-service (Port 8086)

**Responsibility:** In-app and email notification dispatch. Triggered entirely by Kafka events — never called directly via REST by other services.

**Owned Entities:**
- `Notification` — in-app notification records (read/unread state)
- `NotificationTemplate` — per-action templates for email and in-app messages

**Communication:** Kafka consumer only. No REST API is called by other services. No Kafka events are produced.

**Kafka Events Consumed → Notifications Triggered:**

| Event | Recipient | Channels |
|-------|-----------|---------|
| `oms.remote.request.submitted` | Manager / Delegate | In-app + Email |
| `oms.remote.request.approved` | Employee | In-app + Email |
| `oms.remote.request.rejected` | Employee | In-app + Email |
| `oms.ooo.request.approved` | Nominated delegate | In-app + Email |
| `oms.seating.booking.created` | Employee | In-app |
| `oms.seating.booking.released` | Facilities Admin (daily summary) | In-app + Email |
| `oms.visitor.visit.checked_in` | Host employee | In-app + Email |
| `oms.event.rsvp.confirmed` | Employee | In-app + Email |
| `oms.event.rsvp.promoted` | Employee (from waitlist) | In-app + Email |
| `oms.event.event.cancelled` | All attendees | In-app + Email |
| `oms.event.event.reminder` | All attendees | In-app + Email |
| `oms.supplies.stock.low` | Facilities Admin | In-app + Email |
| `oms.supplies.request.approved` | Employee | In-app + Email |
| `oms.supplies.request.rejected` | Employee | In-app + Email |
| `oms.assets.asset.assigned` | Employee | In-app + Email |
| `oms.assets.fault_reported` | Facilities Admin | In-app |
| `oms.assets.maintenance_logged` | Facilities Admin | In-app |

**In-App Read APIs:**
```
GET    /api/v1/notifications/my         → Own unread notifications (Employee)
PATCH  /api/v1/notifications/{id}/read  → Mark notification as read
PATCH  /api/v1/notifications/read-all   → Mark all as read
```

**Database:** `notification_db`

---

## 7. audit-service (Port 8087)

**Responsibility:** Immutable, append-only audit log of every significant action across all services. Read-only query API for authorised roles.

**Owned Entities:**
- `AuditLog` — append-only; no `deleted_at`; no UPDATE or DELETE at DB level

**Communication:** Kafka consumer + SQS listener. All services publish `oms.audit.event`; this service persists records. No service can call an audit write API directly — write access is restricted to the consumers only.

**SQS / Kafka Consumed:**
```
oms.audit.event → {
  eventId, actorId, actorRole, action, entityType, entityId,
  locationId, previousState (JSONB), newState (JSONB), occurredAt
}
```

**Query APIs:**
```
GET  /api/v1/audit-logs                          → Paginated log (HR / Facilities Admin / Super Admin)
GET  /api/v1/audit-logs?entityType=SeatBooking   → Filter by entity type
GET  /api/v1/audit-logs?actorId={userId}         → Filter by actor
GET  /api/v1/audit-logs?locationId={locationId}  → Filter by location
GET  /api/v1/audit-logs?action=SEAT_BOOKED       → Filter by action
```

**Retention:** Records older than 24 months are archived to S3 (Glacier Instant Retrieval) by a background archival job.

**Database:** `audit_db` — PostgreSQL; the application DB user is granted INSERT privilege only.

---

## 8. visitor-service (Port 8088) — Phase 2

**Responsibility:** Visitor pre-registration, walk-in handling, check-in/check-out, agreement template versioning, and visit audit.

**Owned Entities:**
- `VisitorProfile` — reusable visitor identity
- `ParentVisit` — top-level visit record (one per visit engagement)
- `VisitRecord` — individual check-in/out entries; captures agreement template version signed
- `AgreementTemplate` — versioned NDA / access agreement; returning visitors must re-sign when the active version changes

**APIs:**
```
POST   /api/v1/visitors/pre-register           → Pre-register expected visitor (Employee)
POST   /api/v1/visitors/walk-in                → Register walk-in (Facilities Admin)
PATCH  /api/v1/visits/{id}/check-in            → Check visitor in
PATCH  /api/v1/visits/{id}/check-out           → Check visitor out
GET    /api/v1/visits/history                  → Own visit history
GET    /api/v1/visits/location/{locationId}    → All visits at location (HR / Facilities Admin)
GET    /api/v1/visitors/audit-log              → Full visit audit trail
POST   /api/v1/agreement-templates             → Create template (Facilities Admin)
GET    /api/v1/agreement-templates/active      → Current active template
```

**Kafka Events Produced:**
```
oms.visitor.visit.checked_in   → { visitId, visitorName, hostUserId, locationId, timestamp }
oms.visitor.visit.checked_out  → { visitId, locationId, timestamp }
```

**Database:** `visitor_db`

---

## 9. event-service (Port 8089) — Phase 2

**Responsibility:** Office event creation, RSVP management, automatic waitlist promotion, and recurring event series.

**Owned Entities:**
- `EventSeries` — parent for recurring events
- `Event` — individual event instance (child of EventSeries or standalone)
- `EventInvite` — per-user RSVP; statuses: `ATTENDING / WAITLISTED / CANCELLED`

**Recurring edit scopes:** single instance only · this and future (splits EventSeries) · entire series.

**Waitlist:** On RSVP cancellation, the next waitlisted user is automatically promoted and notified.

**APIs:**
```
GET    /api/v1/events                   → Upcoming events
GET    /api/v1/events/{id}              → Event detail
POST   /api/v1/events                   → Create event (Manager / HR / Facilities Admin)
PATCH  /api/v1/events/{id}              → Update event
DELETE /api/v1/events/{id}              → Cancel event
POST   /api/v1/events/{id}/rsvp        → RSVP (joins attendees or waitlist)
DELETE /api/v1/events/{id}/rsvp        → Cancel RSVP
GET    /api/v1/events/{id}/attendees   → Attendee list + RSVP report
```

**Kafka Events Produced:**
```
oms.event.event.created        → { eventId, title, date, capacity, locationId, organizerId }
oms.event.rsvp.confirmed       → { eventId, userId, locationId }
oms.event.rsvp.waitlisted      → { eventId, userId, locationId }
oms.event.rsvp.promoted        → { eventId, userId, locationId }
oms.event.event.cancelled      → { eventId, locationId, attendeeIds[] }
oms.event.event.reminder       → { eventId, locationId, attendeeIds[], hoursUntilEvent }
```

**Database:** `event_db`

---

## 10. supplies-service (Port 8090) — Phase 2

**Responsibility:** Supply catalogue, batch-level stock tracking, two-stage approval workflow Saga, and procurement reorder trigger.

**Owned Entities:**
- `SupplyCategory`, `SupplyItem` — catalogue structure
- `SupplyStockEntry` — per-batch stock tracking with expiry (FIFO fulfilment)
- `SupplyRequest` — request with two-stage approval: `PENDING_MANAGER → PENDING_FACILITIES → FULFILLED`
- `ReorderLog` — record of procurement reorder events

**Stock management:** Fulfilment decrements from the earliest non-expired batch (FIFO). When aggregate stock drops below the configured reorder threshold, a `supply.reorder.triggered` event is published.

**APIs:**
```
GET    /api/v1/supplies/catalogue                   → Browse catalogue
POST   /api/v1/supplies/catalogue                   → Add item (Facilities Admin)
PATCH  /api/v1/supplies/catalogue/{id}              → Update item
POST   /api/v1/supply-requests                      → Submit request (Employee)
PATCH  /api/v1/supply-requests/{id}/approve-stage1  → Manager approve (Stage 1)
PATCH  /api/v1/supply-requests/{id}/reject-stage1   → Manager reject
PATCH  /api/v1/supply-requests/{id}/fulfil          → Facilities Admin fulfil (Stage 2)
PATCH  /api/v1/supply-requests/{id}/reject-stage2   → Facilities Admin reject
GET    /api/v1/supplies/inventory-report            → Inventory report (HR / Facilities Admin)
POST   /api/v1/supplies/stock-entries               → Add stock batch
PATCH  /api/v1/supplies/stock-entries/{id}/write-off → Write off expired batch
```

**Kafka Events Produced:**
```
oms.supplies.request.submitted    → { requestId, userId, items, locationId }
oms.supplies.request.approved     → { requestId, stage, locationId }
oms.supplies.request.rejected     → { requestId, stage, reason, userId }
oms.supplies.request.fulfilled    → { requestId, userId, locationId }
oms.supplies.stock.low            → { itemId, currentStock, threshold, locationId }
oms.supplies.reorder.triggered    → { itemId, quantityRequested, locationId }
```

**Database:** `supplies_db`

---

## 11. assets-service (Port 8091) — Phase 2

**Responsibility:** Non-consumable asset register, full assignment lifecycle, fault reporting, maintenance tracking, and retirement.

**Owned Entities:**
- `AssetCategory`, `Asset` — register entries
- `AssetAssignment` — assignment to a user; statuses: `ASSIGNED / ACKNOWLEDGED / RETURNED / RETIRED`
- `AssetRequest` — request to be assigned an asset
- `MaintenanceRecord` — maintenance event log
- `FaultReport` — employee-submitted fault reports

**APIs:**
```
GET    /api/v1/assets                                → Asset register (HR / Facilities Admin)
POST   /api/v1/assets                               → Add asset
GET    /api/v1/assets/{id}                          → Asset detail
PATCH  /api/v1/assets/{id}/retire                   → Retire asset
GET    /api/v1/assets/my                            → Own assigned assets (Employee)
POST   /api/v1/asset-assignments                    → Assign asset (Facilities Admin)
PATCH  /api/v1/asset-assignments/{id}/acknowledge   → Employee acknowledges receipt
PATCH  /api/v1/asset-assignments/{id}/return        → Return asset
POST   /api/v1/asset-requests                       → Submit asset request (Employee)
PATCH  /api/v1/asset-requests/{id}/approve          → Manager approve
PATCH  /api/v1/asset-requests/{id}/fulfil           → Facilities Admin fulfil
POST   /api/v1/fault-reports                        → Submit fault report (Employee)
POST   /api/v1/maintenance-records                  → Log maintenance event (Facilities Admin)
GET    /api/v1/assets/{id}/audit-log                → Full asset audit trail
```

**Kafka Events Produced:**
```
oms.assets.asset.assigned           → { assetId, userId, locationId }
oms.assets.asset.acknowledged       → { assetId, userId, locationId }
oms.assets.asset.returned           → { assetId, userId, locationId }
oms.assets.fault_reported           → { assetId, userId, faultType, locationId }
oms.assets.maintenance_logged       → { assetId, maintenanceType, locationId }
oms.assets.asset.retired            → { assetId, reason, locationId }
```

**Database:** `assets_db`
