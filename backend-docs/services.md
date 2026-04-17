# Service Definitions
## Office Management System (OMS)

**Version:** 3.0 — 8 services (merged from 11)

All 8 services follow the same structural, security, and operational standards. Each service:
- Owns exactly one business capability and its bounded context
- Has its own PostgreSQL database — no other service can access it
- Registers with Netflix Eureka on startup — Spring Cloud Gateway routes via `lb://`
- Validates every inbound request with an internal JWT (HMAC-SHA256)
- Enforces role + location checks via `LocationRoleInterceptor` on every protected endpoint
- Publishes an `oms.audit.event` to RabbitMQ on every state-changing operation
- Implements Resilience4j circuit breaker + retry + timeout + fallback on every outgoing Feign call

---

## Service Merge Decisions

### Merge 1: identity-service + user-service → auth-service

**Why these were separate:** Original design kept authentication (session/token lifecycle) and user management (profiles, roles) in separate services to enforce single responsibility.

**Why they were merged:**
- `identity-service` called `user-service` on EVERY login to resolve roles — a hard synchronous dependency that could not be eliminated
- Both services owned the SSO subject (`sso_sub`) as their primary integration point
- They shared the same deployment cadence, the same team ownership, and the same on-call responsibility
- No independent scalability benefit: login volume and role-management volume scale together
- Merging eliminates one network hop on the hottest path (every login) and removes one RDS instance (~$35/month)

**Trade-off accepted:** `auth-service` is slightly larger than typical microservice guidance recommends. Mitigated by clear internal package separation (`auth.*` vs `user.*` vs `location.*`).

### Merge 2: supplies-service + assets-service → inventory-service

**Why these were separate:** Supply items (consumables) and assets (non-consumables) are conceptually different.

**Why they were merged:**
- Both manage physical items at a location under Facilities Admin authority
- Both use an approval workflow with the same roles (Employee → Manager → Facilities Admin)
- Both needed location-scoped stock/register queries — a merged `inventory_db` avoids cross-service joins for inventory reports
- Combined traffic is low (Phase 2, non-critical path)
- Saves ~$35/month RDS + ~$50/month ECS task = ~$1,020/year

**Trade-off accepted:** A very large supply request involving assets must be handled in one service. The bounded context is "physical items at a location" — this is cohesive enough.

### Merge 3: visitor-service + event-service → workplace-service

**Why these were separate:** Visitor management and office event management are distinct use cases.

**Why they were merged:**
- Both are Phase 2 features with low individual traffic
- Both are primarily driven by Facilities Admin and HR at a location
- Both use agreement/template versioning (visitors sign NDAs; events have terms)
- Merging halves operational overhead (one service, one DB, one deployment pipeline)
- Independent scaling is not needed at OMS's current traffic projections
- Saves ~$1,020/year

**Trade-off accepted:** If visitor or event traffic grows significantly, the service can be split again. The `visitor.*` and `event.*` packages are kept cleanly separated inside `workplace-service` to make a future split straightforward.

---

## 1. auth-service (Port 8081)

**Business purpose:** Provide a single, authoritative authentication and identity boundary for the OMS. No other service manages user identity or session lifecycle.

**Why this scope:** Every request to every service flows through auth validation at the gateway. The auth-service is the highest-trust, highest-availability service in the platform. It must own both session issuance (identity) and role resolution (user) because these two concerns are inseparable at login time.

**Owned Entities:**
- `Session` — HTTP-only session cookie; maps to OMS user identity (`spring-session-jdbc`)
- `User` — profile, SSO subject, employment dates, Arms HR personnel ID
- `UserRole` — user + role enum + `location_id` (NULL = SUPER_ADMIN, cross-location access)
- `Location` — office locations (name, country, timezone)
- `LocationConfig` — per-location operational settings
- `PublicHoliday` — per-location holiday calendar

**Public APIs:**
```
POST  /api/v1/auth/login                    → Initiates SSO redirect to provider
GET   /api/v1/auth/callback                 → Exchanges OIDC code; verifies JWKS; issues session cookie
POST  /api/v1/auth/logout                   → Invalidates session; clears cookie
GET   /api/v1/auth/me                       → Returns { userId, email, roles[] } from session

GET   /api/v1/users/{id}                    → User profile (own or HR/SUPER_ADMIN)
GET   /api/v1/users/{id}/roles              → User roles across locations (HR, SUPER_ADMIN)
GET   /api/v1/users?ssoSub={sub}            → Find user by SSO subject (internal gateway use)
GET   /api/v1/users/{id}/direct-reports     → Team members under a manager (used by other services via Feign)
GET   /api/v1/locations                     → All active locations
GET   /api/v1/locations/{id}/config         → Location operational configuration
PATCH /api/v1/locations/{id}/config         → Update location config (SUPER_ADMIN)
POST  /api/v1/users/{id}/roles              → Assign role at location (SUPER_ADMIN)
DELETE /api/v1/users/{id}/roles/{roleId}    → Revoke role (SUPER_ADMIN)
GET   /api/v1/locations/{id}/role-holder    → Find user holding a role at a location (internal Feign)
```

**Internal API (gateway only — not routed to frontend):**
```
GET   /internal/validate                    → Called by Spring Cloud Gateway on every inbound request;
                                              returns user context or 401; never called directly by frontend
```

**Scheduled Jobs:**
- `HrSyncJob` — polls the Arms HR API on a configurable cron; upserts `User` records and `UserRole` assignments idempotently; emits RabbitMQ events on changes.

**RabbitMQ Events Produced:**
```
oms.user.created          → { userId, locationId, employmentStartDate }
oms.user.updated          → { userId, changedFields[] }
oms.user.deactivated      → { userId, employmentEndDate }
oms.user.sync.completed   → { syncedCount, failedCount, timestamp }
```

**Availability note:** If auth-service is degraded, new logins are blocked. Existing sessions remain valid until expiry (configurable via `SESSION_EXPIRY_SECONDS`). All other services continue serving existing sessions without calling auth-service.

**Key Config:**
```bash
SSO_JWKS_URI=https://sso.yourorg.com/.well-known/jwks.json
SSO_ISSUER_URI=https://sso.yourorg.com
SESSION_EXPIRY_SECONDS=3600
HR_SYNC_API_URL=https://arms.internal/api/employees
HR_SYNC_CRON=0 0 1 * * *
```

**Database:** `auth_db`

---

## 2. attendance-service (Port 8082)

**Business purpose:** Provide a reliable, accurate daily attendance record for every employee at every location. Source of truth for HR reporting, payroll integration, and compliance.

**Why this scope:** Badge data ingestion, session resolution, and attendance computation are tightly coupled stages of a single pipeline. Separating them would require three services sharing the same data and the same Athena source — no benefit, high coupling.

**Owned Entities:**
- `BadgeEvent` — immutable after ingestion; one row per badge swipe from AWS Athena
- `WorkSession` — resolved from badge events; captures `first_badge_in`, `last_badge_out`, duration, lateness
- `AttendanceRecord` — final daily status per user per date: `PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY`
- `BadgeSyncJobLog` — execution audit log per location per run (inserted, skipped, failed counts)
- `PersonnelIdMapping` — maps Arms HR `personnel_id` to OMS `user_id` (local read-model, no cross-service call at sync time)

**Two-Pass Attendance Pipeline:**
```
AWS Athena (badge_events table)
  ↓ nightly BadgeSyncJob (2am per location, configurable cron)
BadgeEvent records — immutable after insert
  ↓ WorkSession resolution
     - Group badge events by (user_id, location_id, date)
     - Handle cross-midnight sessions (gap > SESSION_GAP_THRESHOLD_HOURS = new session)
     - Compute is_late: first_badge_in > attendance_cutoff_time (per LocationConfig)
  ↓ Pass 1 — AttendanceRecord
     - PRESENT or LATE based on WorkSession
     - ABSENT if no WorkSession and not a public holiday
     - PUBLIC_HOLIDAY if date is in public_holidays for that location
  ↓ Pass 2 — RabbitMQ consumers overlay status corrections
     - oms.remote.request.approved → overwrite with REMOTE
     - oms.ooo.request.approved    → overwrite with OOO
Final AttendanceRecord: PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY
```

**Design trade-off — two passes vs. one pass:** Pass 2 status corrections arrive asynchronously (manager approves remote day after midnight sync has run). Designing for this with a two-pass model means attendance records are eventually correct without requiring a synchronous coordination between `attendance-service` and `remote-service` at sync time.

**APIs:**
```
GET  /api/v1/attendance/{userId}                → Own attendance history (EMPLOYEE); team/all for higher roles
GET  /api/v1/attendance/team/{managerId}         → Team attendance (MANAGER, HR)
GET  /api/v1/attendance/location/{locationId}    → Location-wide (HR, SUPER_ADMIN)
POST /api/v1/attendance/{id}/override            → Manual override (SUPER_ADMIN) with audit trail
GET  /api/v1/attendance/reports/no-shows         → No-show report (HR, FACILITIES_ADMIN)
GET  /api/v1/attendance/export                   → CSV / Excel download (HR, SUPER_ADMIN)
GET  /api/v1/attendance/sync/logs                → Sync job execution logs (SUPER_ADMIN, FACILITIES_ADMIN)
POST /api/v1/attendance/sync/trigger             → Manual sync trigger (SUPER_ADMIN) — returns 202 Accepted
```

**RabbitMQ Events Consumed:**
```
oms.remote.request.approved  → Overlay REMOTE status for the user on affected dates
oms.ooo.request.approved     → Overlay OOO status for the user across the date range
oms.user.deactivated         → Stop generating AttendanceRecords after employmentEndDate
```

**RabbitMQ Events Produced:**
```
oms.attendance.record.resolved   → { userId, recordDate, status, locationId }
oms.attendance.no_show.detected  → { userId, date, locationId } — consumed by seating-service
```

**Key Config:**
```bash
ATHENA_DATABASE=badge_events
ATHENA_OUTPUT_BUCKET=s3://oms-athena-results/
BADGE_SYNC_CRON=0 0 2 * * *
ATHENA_QUERY_TIMEOUT_SECONDS=120
SESSION_GAP_THRESHOLD_HOURS=6
```

**Database:** `attendance_db`

---

## 3. seating-service (Port 8083)

**Business purpose:** Manage hot-desk availability in real time and ensure efficient space utilisation through automated no-show release.

**Why this scope:** Floor plan, booking, and no-show release are inseparable for real-time seat availability. The floor plan API must return live booking state — splitting booking from floor plan would require joins between two databases on the hottest read path.

**Owned Entities:**
- `Floor`, `Zone`, `Seat` — spatial hierarchy (location → floor → zone → seat)
- `SeatBooking` — individual hot-desk reservation; statuses: `CONFIRMED / CANCELLED / RELEASED`
- `BlockReservation` — manager-reserved seat groups for team use
- `NoShowRecord` — audit record of every booking released by the no-show job

**Availability computation:** Real-time from `seat_bookings` filtered by `(booking_date, location_id)` on a compound index. No pre-computed cache. Sub-200ms at current scale. Cache added only when load testing demonstrates a bottleneck.

**Trade-off — optimistic locking for concurrent booking:** Two employees booking the same seat at the same millisecond will result in one `CONFIRMED` and one `409 SEAT_UNAVAILABLE`. This is intentional — optimistic locking (`@Version` on `SeatBooking`) is lighter than pessimistic locks. The frontend prompts the user to select another seat.

**APIs:**
```
GET    /api/v1/locations/{id}/floor-plan       → Floor plan with live seat availability (any role)
POST   /api/v1/seat-bookings                   → Book a hot-desk (EMPLOYEE)
DELETE /api/v1/seat-bookings/{id}              → Cancel booking (EMPLOYEE own; MANAGER team; FACILITIES_ADMIN any)
GET    /api/v1/seat-bookings/my                → Own bookings (?from&to)
POST   /api/v1/seat-bookings/block             → Block-book seats for a team (MANAGER)
POST   /api/v1/seats/{id}/permanent            → Assign permanent seat (HR, FACILITIES_ADMIN)
DELETE /api/v1/seats/{id}/permanent            → Release permanent seat
PUT    /api/v1/locations/{id}/floor-plan       → Configure floor plan layout (FACILITIES_ADMIN)
```

**Scheduled Jobs:**
- `NoShowReleaseJob` — runs daily at `no_show_release_time` (per `LocationConfig`); queries CONFIRMED bookings with no badge-in by cutoff; marks them `RELEASED`; creates `NoShowRecord`; publishes `oms.seating.booking.released`.

**RabbitMQ Events Consumed:**
```
oms.attendance.no_show.detected  → Trigger seat release check for user on that date
oms.user.deactivated             → Cancel all future CONFIRMED bookings for that user
```

**RabbitMQ Events Produced:**
```
oms.seating.booking.created    → { bookingId, userId, seatId, date, locationId }
oms.seating.booking.cancelled  → { bookingId, userId, locationId }
oms.seating.booking.released   → { bookingId, userId, reason: NO_SHOW, locationId }
```

**Key Config:**
```bash
BOOKING_WINDOW_DAYS=14
CANCELLATION_CUTOFF_HOURS=2
NO_SHOW_RELEASE_CRON=0 0 10 * * *
```

**Database:** `seating_db`

---

## 4. remote-service (Port 8084)

**Business purpose:** Allow employees to formally declare remote-working days and OOO periods, with configurable policy enforcement and manager approval, so attendance records are accurate and team coverage is maintained.

**Why this scope:** Remote day policy enforcement, approval delegation, and recurring schedule management are all aspects of the same bounded context: "employee work location declarations."

**Owned Entities:**
- `RemoteRequest` — individual remote-day request (standalone or child of a recurring schedule)
- `RecurringRemoteSchedule` — parent for recurring remote patterns (e.g., every Monday remote)
- `OOORequest` — out-of-office request (date range)
- `ApprovalDelegate` — active nomination for who approves when manager is OOO
- `RemoteDayPolicy` — configurable limits: `max_days_per_week`, `team_overlap_threshold_pct`, `policy_type`

**Policy enforcement logic:**
1. Fetch team-level `RemoteDayPolicy` for the manager; fall back to location-level default.
2. Count existing approved remote requests for the same week.
3. If limit exceeded AND `policy_type = HARD_BLOCK` → reject with `POLICY_HARD_BLOCK` (400).
4. If limit exceeded AND `policy_type = SOFT_WARNING` → allow with `warning: true` in response body.
5. Check team overlap threshold — if too many team members are already remote on the same day → same HARD_BLOCK/SOFT_WARNING logic.

**Approval routing:**
- Primary approver: user's direct manager.
- If manager has an active `ApprovalDelegate` on that date → delegate approves.
- If manager is OOO with no delegate → falls back to HR role at that location via Feign call to auth-service (`GET /api/v1/locations/{locationId}/role-holder?role=HR`).

**APIs:**
```
POST   /api/v1/remote-requests                       → Submit remote-day request (EMPLOYEE)
POST   /api/v1/ooo-requests                          → Submit OOO request (EMPLOYEE)
PATCH  /api/v1/remote-requests/{id}/approve          → Approve (MANAGER, Delegate)
PATCH  /api/v1/remote-requests/{id}/reject           → Reject with reason (MANAGER, Delegate)
GET    /api/v1/teams/{managerId}/schedule            → Team schedule calendar (MANAGER, HR, SUPER_ADMIN)
POST   /api/v1/approval-delegates                    → Nominate delegate (MANAGER)
GET    /api/v1/remote-day-policies/{locationId}      → Location policy (any role)
PATCH  /api/v1/remote-day-policies/{managerId}       → Override team policy (HR, SUPER_ADMIN)
GET    /api/v1/remote-requests/history               → Own request history with status
POST   /api/v1/remote-requests/{id}/override         → Backdated override (SUPER_ADMIN)
```

**RabbitMQ Events Produced:**
```
oms.remote.request.submitted    → { requestId, userId, dates[], locationId }
oms.remote.request.approved     → { requestId, userId, dates[], locationId }
oms.remote.request.rejected     → { requestId, userId, reason, locationId }
oms.ooo.request.approved        → { requestId, userId, startDate, endDate, locationId }
```

**Database:** `remote_db`

---

## 5. notification-service (Port 8085)

**Business purpose:** Deliver timely in-app and email notifications to the right employees without creating a hard dependency from any upstream service.

**Why Kafka/RabbitMQ-only (never called via REST):** If notification-service were callable via REST, every service that sends a notification would have a hard availability dependency on it. A notification-service degradation would cascade to seating, remote, inventory, and workplace services. As a RabbitMQ consumer, notification-service can be down for minutes without any other service failing. Messages queue in RabbitMQ and are processed when the service recovers — no notifications are lost, and no upstream service is blocked.

**Design trade-off:** In-app notifications may be delayed if notification-service is down. This is accepted. The alternative (synchronous REST) would break booking confirmations, approval workflows, and visitor check-ins during notification outages. Delayed notifications are always preferable to failed business operations.

**Owned Entities:**
- `Notification` — in-app notification record (read/unread state, message, type, recipient)
- `NotificationTemplate` — per-event-type templates for email and in-app messages

**RabbitMQ Events Consumed → Notifications Triggered:**

| Routing Key | Recipient | Channels |
|-------------|-----------|---------|
| `oms.remote.request.submitted` | Manager / active Delegate | In-app + Email |
| `oms.remote.request.approved` | Requesting Employee | In-app + Email |
| `oms.remote.request.rejected` | Requesting Employee | In-app + Email |
| `oms.ooo.request.approved` | Nominated Delegate | In-app + Email |
| `oms.seating.booking.created` | Booking Employee | In-app |
| `oms.seating.booking.released` | Facilities Admin (daily summary) | In-app + Email |
| `oms.workplace.visit.checked_in` | Host Employee | In-app + Email |
| `oms.workplace.event.rsvp.confirmed` | RSVPing Employee | In-app + Email |
| `oms.workplace.event.rsvp.promoted` | Waitlisted Employee | In-app + Email |
| `oms.workplace.event.cancelled` | All RSVPed Attendees | In-app + Email |
| `oms.workplace.event.reminder` | All RSVPed Attendees | In-app + Email |
| `oms.inventory.stock.low` | Facilities Admin | In-app + Email |
| `oms.inventory.supply.request.approved` | Requesting Employee | In-app + Email |
| `oms.inventory.supply.request.rejected` | Requesting Employee | In-app + Email |
| `oms.inventory.asset.assigned` | Assigned Employee | In-app + Email |
| `oms.inventory.fault.reported` | Facilities Admin | In-app |
| `oms.user.created` | New Employee (welcome notification) | Email |

**Read APIs (no service calls these — frontend only):**
```
GET    /api/v1/notifications/my          → Own unread/all notifications (?unreadOnly=true)
PATCH  /api/v1/notifications/{id}/read   → Mark as read
PATCH  /api/v1/notifications/read-all    → Mark all as read
```

**Database:** `notification_db`

---

## 6. audit-service (Port 8086)

**Business purpose:** Provide an immutable, tamper-proof audit trail of every significant action across the entire OMS platform for compliance, HR investigations, and incident analysis.

**Why INSERT-only at the database level:** Audit integrity is a compliance requirement. If the application DB user had UPDATE or DELETE privileges, a compromised service or developer error could alter or erase audit records. Granting INSERT-only privilege at the database level makes tampering physically impossible from the application layer.

**Why RabbitMQ-only (no REST write API):** Any REST endpoint that writes audit records could be called incorrectly, bypassed, or exploited. The only correct write path is consuming the event that the business operation produced. This guarantees that audit records are always correlated with real state changes, not manually crafted API calls.

**Owned Entities:**
- `AuditLog` — append-only; no `deleted_at`; DB user has INSERT privilege only

**RabbitMQ Events Consumed:**
```
oms.audit.event (routing key on every service's audit binding):
{
  eventId, actorId, actorRole, action, entityType, entityId,
  locationId, previousState (JSONB), newState (JSONB), occurredAt, correlationId
}
```

**Read APIs:**
```
GET  /api/v1/audit-logs           → Paginated log with filters (HR, FACILITIES_ADMIN, SUPER_ADMIN)
     ?actorId=uuid                → Filter by actor
     ?entityType=SeatBooking      → Filter by entity type
     ?entityId=uuid               → Filter by specific entity
     ?locationId=uuid             → Filter by location (SUPER_ADMIN sees all)
     ?action=SEAT_BOOKED          → Filter by action
     ?from=2026-01-01&to=2026-03-31
     &page=0&size=50
```

**Retention and archival:**
- Records older than 24 months are archived to S3 (Glacier Instant Retrieval) by a background `AuditArchivalJob`.
- Archived records remain queryable via a separate search path (future: AWS Athena over S3).
- RDS growth is bounded; no unbounded table growth risk.

**Database:** `audit_db` — PostgreSQL; application DB user: INSERT privilege only on `audit_logs`.

---

## 7. inventory-service (Port 8087) — Phase 2

**Business purpose:** Track all physical items — consumable supplies and non-consumable assets — at each office location, managing their full lifecycle from procurement through usage to disposal.

**Why one service for supplies and assets:** Both domains manage physical items under Facilities Admin authority using an approval workflow. Splitting them would require a cross-service join for "all items requested by employee X this month" and for consolidated inventory reports. The merge provides a single `inventory_db` with full local query capability. The supply and asset bounded contexts are cleanly separated into `inventory.supply.*` and `inventory.asset.*` packages internally.

**Owned Entities (Supply side):**
- `SupplyCategory`, `SupplyItem` — catalogue (consumable items with reorder thresholds)
- `SupplyStockEntry` — per-batch stock tracking with expiry date; FIFO fulfilment order
- `SupplyRequest` — two-stage approval: `PENDING_MANAGER → PENDING_FACILITIES → FULFILLED`
- `ReorderLog` — record of triggered procurement reorder events

**Owned Entities (Asset side):**
- `AssetCategory`, `Asset` — non-consumable asset register
- `AssetAssignment` — assignment to user; statuses: `ASSIGNED / ACKNOWLEDGED / RETURNED / RETIRED`
- `AssetRequest` — employee request for an asset
- `MaintenanceRecord` — scheduled and ad-hoc maintenance events
- `FaultReport` — employee-submitted fault reports with severity

**Supply fulfilment logic:**
- Fulfilment decrements from the earliest non-expired `SupplyStockEntry` (FIFO).
- When aggregate stock after fulfilment drops below `reorder_threshold` → publish `oms.inventory.stock.low` AND `oms.inventory.reorder.triggered`.

**APIs (Supply):**
```
GET    /api/v1/supplies/catalogue                    → Browse catalogue (any role)
POST   /api/v1/supplies/catalogue                    → Add item (FACILITIES_ADMIN)
PATCH  /api/v1/supplies/catalogue/{id}               → Update item
POST   /api/v1/supply-requests                       → Submit request (EMPLOYEE) → status: PENDING_MANAGER
PATCH  /api/v1/supply-requests/{id}/approve-stage1   → Manager approves → PENDING_FACILITIES
PATCH  /api/v1/supply-requests/{id}/reject-stage1    → Manager rejects → REJECTED
PATCH  /api/v1/supply-requests/{id}/fulfil           → Facilities Admin fulfils → FULFILLED; decrements FIFO stock
PATCH  /api/v1/supply-requests/{id}/reject-stage2    → Facilities Admin rejects
GET    /api/v1/supplies/inventory-report             → Stock levels per item per location (HR, FACILITIES_ADMIN)
POST   /api/v1/supplies/stock-entries                → Add stock batch (FACILITIES_ADMIN)
PATCH  /api/v1/supplies/stock-entries/{id}/write-off → Write off expired batch
```

**APIs (Asset):**
```
GET    /api/v1/assets                                 → Asset register (HR, FACILITIES_ADMIN)
POST   /api/v1/assets                                → Register new asset (FACILITIES_ADMIN)
GET    /api/v1/assets/{id}                           → Asset detail
PATCH  /api/v1/assets/{id}/retire                    → Retire asset
GET    /api/v1/assets/my                             → Own assigned assets (EMPLOYEE)
POST   /api/v1/asset-assignments                     → Assign asset to user (FACILITIES_ADMIN)
PATCH  /api/v1/asset-assignments/{id}/acknowledge    → Employee acknowledges receipt
PATCH  /api/v1/asset-assignments/{id}/return         → Return asset
POST   /api/v1/asset-requests                        → Request an asset (EMPLOYEE)
PATCH  /api/v1/asset-requests/{id}/approve           → Manager approves request
PATCH  /api/v1/asset-requests/{id}/fulfil            → Facilities Admin fulfils
POST   /api/v1/fault-reports                         → Report asset fault (EMPLOYEE)
POST   /api/v1/maintenance-records                   → Log maintenance event (FACILITIES_ADMIN)
GET    /api/v1/assets/{id}/audit-log                 → Full chronological asset audit trail
```

**RabbitMQ Events Produced:**
```
oms.inventory.supply.request.submitted    → { requestId, userId, items[], locationId }
oms.inventory.supply.request.approved     → { requestId, stage, locationId }
oms.inventory.supply.request.rejected     → { requestId, stage, reason, userId }
oms.inventory.supply.request.fulfilled    → { requestId, userId, locationId }
oms.inventory.stock.low                   → { itemId, currentStock, threshold, locationId }
oms.inventory.reorder.triggered           → { itemId, quantityRequested, locationId }
oms.inventory.asset.assigned              → { assetId, userId, locationId }
oms.inventory.asset.acknowledged          → { assetId, userId, locationId }
oms.inventory.asset.returned              → { assetId, userId, locationId }
oms.inventory.fault.reported              → { assetId, userId, faultType, severity, locationId }
oms.inventory.maintenance.logged          → { assetId, maintenanceType, locationId }
oms.inventory.asset.retired               → { assetId, reason, locationId }
```

**Database:** `inventory_db`

---

## 8. workplace-service (Port 8088) — Phase 2

**Business purpose:** Manage non-employee presence at the office — visitors who must be tracked for security and compliance, and office events that employees can attend.

**Why one service for visitors and events:** Both are Phase 2, low-traffic, location-scoped features operated by the same roles (Facilities Admin, HR). Both use agreement/access templates (visitors sign NDAs; events can require terms). Both feed into the visitor/event audit trail. Merging halves the Phase 2 operational footprint. The `workplace.visitor.*` and `workplace.event.*` packages are kept cleanly separated for a future split.

**Owned Entities (Visitor side):**
- `VisitorProfile` — reusable visitor identity (email as natural key)
- `ParentVisit` — top-level visit engagement (pre-registered or walk-in)
- `VisitRecord` — individual check-in/check-out; records the agreement template version signed
- `AgreementTemplate` — versioned NDA / access agreement; returning visitors must re-sign when the active version changes

**Owned Entities (Event side):**
- `EventSeries` — parent for recurring event patterns
- `Event` — individual instance (child of EventSeries or standalone)
- `EventInvite` — per-user RSVP; statuses: `ATTENDING / WAITLISTED / CANCELLED`

**Agreement versioning logic:** On each visitor check-in, the service compares the visitor's last-signed template version with the current active version. If they differ, a new signature is required before check-in completes. This ensures visitors always consent to the latest terms regardless of how long between visits.

**Waitlist promotion logic:** On RSVP cancellation, the service automatically promotes the next `WAITLISTED` invite to `ATTENDING` and publishes `oms.workplace.event.rsvp.promoted`. Promotion is transactional — it cannot leave two attendees promoted for the same slot.

**Recurring event edit scopes:** `SINGLE` (one instance only) · `THIS_AND_FUTURE` (splits the EventSeries at this date) · `ALL` (modifies the entire series).

**APIs (Visitor):**
```
POST   /api/v1/visitors/pre-register              → Pre-register expected visitor (EMPLOYEE)
POST   /api/v1/visitors/walk-in                   → Register walk-in visitor (FACILITIES_ADMIN)
PATCH  /api/v1/visits/{id}/check-in               → Check visitor in; validate agreement version
PATCH  /api/v1/visits/{id}/check-out              → Check visitor out
GET    /api/v1/visits/history                     → Own visitor history (EMPLOYEE)
GET    /api/v1/visits/location/{locationId}       → All visits at location (HR, FACILITIES_ADMIN)
POST   /api/v1/agreement-templates                → Create new agreement template version (FACILITIES_ADMIN)
GET    /api/v1/agreement-templates/active         → Current active template
```

**APIs (Event):**
```
GET    /api/v1/events                             → Upcoming events (?locationId&from&to)
GET    /api/v1/events/{id}                        → Event detail
POST   /api/v1/events                             → Create event (MANAGER, HR, FACILITIES_ADMIN)
PATCH  /api/v1/events/{id}                        → Update event (?editScope=SINGLE|THIS_AND_FUTURE|ALL)
DELETE /api/v1/events/{id}                        → Cancel event (?editScope=...)
POST   /api/v1/events/{id}/rsvp                   → RSVP (EMPLOYEE) → ATTENDING or WAITLISTED
DELETE /api/v1/events/{id}/rsvp                   → Cancel RSVP; triggers waitlist promotion
GET    /api/v1/events/{id}/attendees              → Attendee + RSVP report (organiser, HR, FACILITIES_ADMIN)
```

**RabbitMQ Events Produced:**
```
oms.workplace.visit.checked_in           → { visitId, visitorName, hostUserId, locationId, timestamp }
oms.workplace.visit.checked_out          → { visitId, locationId, timestamp }
oms.workplace.event.created              → { eventId, title, date, capacity, locationId, organizerId }
oms.workplace.event.rsvp.confirmed       → { eventId, userId, locationId }
oms.workplace.event.rsvp.waitlisted      → { eventId, userId, locationId }
oms.workplace.event.rsvp.promoted        → { eventId, userId, locationId }
oms.workplace.event.cancelled            → { eventId, locationId, attendeeIds[] }
oms.workplace.event.reminder             → { eventId, locationId, attendeeIds[], hoursUntilEvent }
```

**Database:** `workplace_db`
