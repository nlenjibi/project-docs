# Microservices Architecture Document
## Office Management System (OMS)

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** March 2026  
**Methodology:** Microservices.io patterns · Twelve-Factor App · Clean Architecture · SOLID · Zero Trust Security

---

## Executive Summary

This document defines the microservices decomposition of the OMS. The system is decomposed into **nine independently deployable services**, each owning its own database, business logic, and API surface. All external traffic routes through a single API Gateway. Services communicate synchronously (REST with circuit breakers) for queries and synchronously/asynchronously (Kafka events) for state changes and cross-service workflows.

The architecture is cloud-native, horizontally scalable, observable by default, and security-first. Every design decision is evaluated against the core rule: **does this increase coupling? If yes — reject.**

---

## 1. Service Decomposition

Service boundaries are derived from Domain-Driven Design (DDD) bounded contexts. Each service represents exactly one business capability and owns its data exclusively.

### Service Inventory

| Service | Business Capability | Database | Communication |
|---------|-------------------|----------|---------------|
| `identity-service` | Authentication, SSO integration, session management | PostgreSQL | REST (sync) |
| `user-service` | User profiles, role assignments, HR sync | PostgreSQL | REST + Kafka |
| `attendance-service` | Badge ingestion, WorkSession resolution, AttendanceRecord management | PostgreSQL | Kafka (consume/produce) |
| `seating-service` | Floor plans, seat booking, no-show auto-release | PostgreSQL | REST + Kafka |
| `remote-service` | Remote day and OOO requests, approval workflow, policy management | PostgreSQL | REST + Kafka |
| `visitor-service` | Visitor lifecycle, check-in/out, agreement templates | PostgreSQL | REST + Kafka |
| `event-service` | Office events, RSVP, waitlist, recurring events | PostgreSQL | REST + Kafka |
| `supplies-service` | Supply catalogue, stock management, procurement triggers | PostgreSQL | REST + Kafka |
| `assets-service` | Asset register, assignments, maintenance, fault reports | PostgreSQL | REST + Kafka |
| `notification-service` | In-app and email notification dispatch | PostgreSQL | Kafka (consume only) |
| `audit-service` | Immutable audit log ingestion and query | PostgreSQL (append-only) | Kafka (consume only) |

> **11 services total.** Each is independently deployable, independently scalable, and owns no shared state.

---

## 2. System Architecture Diagram

```
External Clients (Browser / Mobile)
            │
            ▼
  ┌─────────────────────┐
  │     API Gateway      │  ← All external traffic enters here ONLY
  │  (Auth · Routing ·   │    Rate limiting, JWT validation, logging
  │   Rate Limit · Log)  │
  └──────────┬──────────┘
             │
     ┌───────┴────────────────────────────────────────┐
     │               Internal Service Mesh             │
     │         (mutual TLS between all services)       │
     ├─────────────┬──────────┬──────────┬────────────┤
     │             │          │          │            │
  identity-   user-      attendance- seating-    remote-
  service     service    service     service     service
     │             │          │          │            │
  [PG DB]      [PG DB]   [PG DB]    [PG DB]     [PG DB]
                    │          │
              ┌─────┴──────────┴─────────────────────────┐
              │              Apache Kafka                  │
              │  (Event bus for async cross-service flow)  │
              └──────┬─────────────────────────────────────┘
                     │
          ┌──────────┴──────────────────┐
          │                             │
   notification-service          audit-service
       [PG DB]                   [PG DB — append-only]
```

---

## 3. Service Definitions

### 3.1 Identity Service

**Responsibility:** SSO integration, OIDC token verification, session issuance, and session lifecycle management.

**Owns:**
- Session records
- SSO provider configuration

**APIs:**
```
POST   /api/v1/auth/login           → Initiates SSO redirect
GET    /api/v1/auth/callback        → SSO callback; verifies OIDC token; issues session
POST   /api/v1/auth/logout          → Invalidates session
GET    /api/v1/auth/validate        → Validates session (called internally by API Gateway)
GET    /api/v1/auth/me              → Returns authenticated user identity
```

**Rules:**
- Token verification is entirely server-side. Frontend never handles raw tokens.
- Issues HTTP-only session cookies; JavaScript cannot access them.
- Calls `user-service` via REST to resolve roles after identity verification.
- Produces no Kafka events.

**Database:** `identity_db` — sessions table only. Minimal schema.

---

### 3.2 User Service

**Responsibility:** User profiles, per-location role assignments, HR system (Arms) sync, and org structure.

**Owns:**
- `users` table
- `user_roles` table (user + role + location)
- `locations` table
- `location_configs` table

**APIs:**
```
GET    /api/v1/users/{id}                     → Get user profile
GET    /api/v1/users/{id}/roles               → Get user's roles across locations
GET    /api/v1/locations                      → List locations
GET    /api/v1/locations/{id}/config          → Get location config
PATCH  /api/v1/locations/{id}/config          → Update location config (Super Admin)
POST   /api/v1/users/{id}/roles               → Assign role (Super Admin)
DELETE /api/v1/users/{id}/roles/{roleId}      → Revoke role (Super Admin)
```

**Scheduled Jobs:**
- HR Sync Job — polls Arms API on schedule; upserts User records and UserRole assignments; emits `user.synced` event on Kafka.

**Kafka Events Produced:**
```
user.created      → { userId, locationId, employmentStartDate }
user.updated      → { userId, changedFields }
user.deactivated  → { userId, employmentEndDate }
user.synced       → { syncedCount, failedCount, timestamp }
```

**Database:** `user_db`

---

### 3.3 Attendance Service

**Responsibility:** Badge event ingestion from AWS Athena, WorkSession resolution, two-pass AttendanceRecord computation.

**Owns:**
- `badge_events` table (immutable after ingestion)
- `work_sessions` table
- `attendance_records` table

**APIs:**
```
GET    /api/v1/attendance/{userId}                       → Own attendance history
GET    /api/v1/attendance/team/{managerId}               → Team attendance (Manager/HR)
GET    /api/v1/attendance/location/{locationId}          → Location-wide (HR/Super Admin)
POST   /api/v1/attendance/{id}/override                  → Override record (Super Admin)
GET    /api/v1/attendance/reports/no-shows               → No-show report
GET    /api/v1/attendance/export                         → CSV/Excel export
```

**Scheduled Jobs:**
- Badge Sync Job — nightly Athena query; resolves WorkSessions; runs two-pass AttendanceRecord stamping.
- Pass 1: badge events → WorkSession → AttendanceRecord (PRESENT / LATE)
- Pass 2: consumes `remote.request.approved` and `ooo.request.approved` Kafka events to overlay REMOTE / OOO / PUBLIC_HOLIDAY statuses.

**Kafka Events Consumed:**
```
remote.request.approved   → Overlay REMOTE status for affected user-dates
ooo.request.approved      → Overlay OOO status for affected user-dates
user.deactivated          → Stop generating AttendanceRecords after employmentEndDate
```

**Kafka Events Produced:**
```
attendance.record.resolved  → { userId, recordDate, status, locationId }
attendance.no_show.detected → { userId, date, locationId }
```

**Database:** `attendance_db`

---

### 3.4 Seating Service

**Responsibility:** Floor plan management, hot-desk booking, permanent seat assignment, block reservations, no-show auto-release.

**Owns:**
- `floors`, `zones`, `seats` tables
- `seat_bookings` table
- `block_reservations` table
- `no_show_records` table

**APIs:**
```
GET    /api/v1/locations/{id}/floor-plan        → Floor plan with real-time availability
POST   /api/v1/seat-bookings                    → Book a hot-desk
DELETE /api/v1/seat-bookings/{id}               → Cancel booking
GET    /api/v1/seat-bookings/my                 → Own bookings
POST   /api/v1/seat-bookings/block              → Manager: block booking
POST   /api/v1/seats/{id}/permanent             → Assign permanent seat (HR/Facilities)
DELETE /api/v1/seats/{id}/permanent             → Release permanent seat
PUT    /api/v1/locations/{id}/floor-plan        → Configure floor plan (Facilities Admin)
```

**Seat availability** is computed in real-time from `seat_bookings` records — no pre-computed cache. At current scale (hundreds of seats per location) this query is sub-millisecond with proper indexing.

**Scheduled Jobs:**
- No-Show Release Job — runs daily at `no_show_release_time` per location; queries CONFIRMED bookings with no badge-in; sets status to RELEASED; creates NoShowRecord.

**Kafka Events Consumed:**
```
attendance.no_show.detected → Trigger no-show release check
user.deactivated            → Cancel future bookings for deactivated user
```

**Kafka Events Produced:**
```
seat.booking.created      → { bookingId, userId, seatId, date, locationId }
seat.booking.cancelled    → { bookingId, userId, locationId }
seat.booking.released     → { bookingId, userId, reason: NO_SHOW, locationId }
```

**Database:** `seating_db`

---

### 3.5 Remote Service

**Responsibility:** Remote day and OOO request workflow, approval delegation, configurable policy enforcement (HARD_BLOCK / SOFT_WARNING).

**Owns:**
- `remote_requests` table (child)
- `recurring_remote_schedules` table (parent)
- `ooo_requests` table
- `approval_delegates` table
- `remote_day_policies` table
- `public_holidays` table

**APIs:**
```
POST   /api/v1/remote-requests                       → Submit remote day request
POST   /api/v1/ooo-requests                          → Submit OOO request
PATCH  /api/v1/remote-requests/{id}/approve          → Approve (Manager)
PATCH  /api/v1/remote-requests/{id}/reject           → Reject with reason (Manager)
GET    /api/v1/teams/{managerId}/schedule            → Team schedule calendar
POST   /api/v1/approval-delegates                    → Nominate delegate
GET    /api/v1/remote-day-policies/{locationId}      → Get location policy
PATCH  /api/v1/remote-day-policies/{managerId}       → Override team policy
POST   /api/v1/remote-requests/{id}/override         → Admin override past dates
```

**Policy enforcement logic:**
- Checks team-level `RemoteDayPolicy` first; falls back to location-level default.
- HARD_BLOCK: rejects submission if weekly limit exceeded or overlap threshold breached.
- SOFT_WARNING: allows submission with a warning in the response.
- If manager goes OOO without nominating a delegate, HR role at that location becomes the fallback approver.

**Kafka Events Produced:**
```
remote.request.submitted    → { requestId, userId, dates, locationId }
remote.request.approved     → { requestId, userId, dates, locationId }
remote.request.rejected     → { requestId, userId, reason, locationId }
ooo.request.approved        → { requestId, userId, startDate, endDate, locationId }
```

**Database:** `remote_db`

---

### 3.6 Visitor Service

**Responsibility:** Visitor pre-registration, walk-in handling, check-in/check-out, agreement template versioning, visit audit.

**Owns:**
- `visitor_profiles` table
- `parent_visits` table
- `visit_records` table
- `agreement_templates` table (versioned)

**APIs:**
```
POST   /api/v1/visitors/pre-register            → Pre-register expected visitor
POST   /api/v1/visitors/walk-in                 → Register walk-in (Facilities Admin)
PATCH  /api/v1/visits/{id}/check-in             → Check in visitor
PATCH  /api/v1/visits/{id}/check-out            → Check out visitor
GET    /api/v1/visits/history                   → Own visitor history
GET    /api/v1/visits/location/{locationId}     → All visits (HR/Facilities Admin)
GET    /api/v1/visitors/audit-log               → Full audit log access
POST   /api/v1/agreement-templates              → Create template (Facilities Admin)
GET    /api/v1/agreement-templates/active       → Current active template
```

**Agreement versioning rule:** Agreement signing is stored at the `visit_record` level, capturing the exact template version signed on each visit. Returning visitors must re-sign when the active template has changed since their last visit.

**Kafka Events Produced:**
```
visitor.checked_in    → { visitId, visitorName, hostUserId, locationId, timestamp }
visitor.checked_out   → { visitId, locationId, timestamp }
```

**Database:** `visitor_db`

---

### 3.7 Event Service

**Responsibility:** Office event creation, RSVP management, waitlist, recurring event support.

**Owns:**
- `event_series` table (parent)
- `events` table (child)
- `event_invites` table

**APIs:**
```
GET    /api/v1/events                          → List upcoming events
GET    /api/v1/events/{id}                     → Event detail
POST   /api/v1/events                          → Create event (Manager/HR/Facilities)
PATCH  /api/v1/events/{id}                     → Update event
DELETE /api/v1/events/{id}                     → Cancel event
POST   /api/v1/events/{id}/rsvp                → RSVP (add to attendees or waitlist)
DELETE /api/v1/events/{id}/rsvp                → Cancel RSVP
GET    /api/v1/events/{id}/attendees           → Attendee list + RSVP report
```

**Recurring event edit scopes:** one instance only, this and future (splits EventSeries), entire series.

**Waitlist:** when capacity is reached, new RSVPs join the waitlist. On cancellation, the next waitlisted user is automatically promoted and notified.

**Kafka Events Produced:**
```
event.created             → { eventId, title, date, capacity, locationId, organizerId }
event.rsvp.confirmed      → { eventId, userId, locationId }
event.rsvp.waitlisted     → { eventId, userId, locationId }
event.rsvp.promoted       → { eventId, userId, locationId }
event.cancelled           → { eventId, locationId, attendeeIds[] }
event.reminder            → { eventId, locationId, attendeeIds[], hoursUntilEvent }
```

**Database:** `event_db`

---

### 3.8 Supplies Service

**Responsibility:** Supply catalogue management, batch-level stock tracking, two-stage approval workflow, procurement reorder trigger.

**Owns:**
- `supply_categories` table
- `supply_items` table
- `supply_stock_entries` table (per batch — expiry-tracked)
- `supply_requests` table
- `reorder_logs` table

**APIs:**
```
GET    /api/v1/supplies/catalogue                    → Browse catalogue
POST   /api/v1/supplies/catalogue                    → Add item (Facilities Admin)
PATCH  /api/v1/supplies/catalogue/{id}               → Update item
POST   /api/v1/supply-requests                       → Submit request
PATCH  /api/v1/supply-requests/{id}/approve-stage1   → Manager approve (Stage 1)
PATCH  /api/v1/supply-requests/{id}/reject-stage1    → Manager reject
PATCH  /api/v1/supply-requests/{id}/fulfil           → Facilities Admin fulfil (Stage 2)
PATCH  /api/v1/supply-requests/{id}/reject-stage2    → Facilities Admin reject
GET    /api/v1/supplies/inventory-report             → Inventory report (HR/Facilities)
POST   /api/v1/supplies/stock-entries                → Add stock batch
PATCH  /api/v1/supplies/stock-entries/{id}/write-off → Write off expired batch
```

**Stock management:** Tracked at batch level via `SupplyStockEntry`. Fulfilment decrements from the earliest non-expired batch (FIFO). When aggregate stock drops below the configured reorder threshold, a `supply.reorder.triggered` event is published and a `ReorderLog` is created. The procurement integration consumes this event (contract TBD — AD-026).

**Kafka Events Produced:**
```
supply.request.submitted    → { requestId, userId, items, locationId }
supply.request.approved     → { requestId, stage, locationId }
supply.request.rejected     → { requestId, stage, reason, userId }
supply.request.fulfilled    → { requestId, userId, locationId }
supply.stock.low            → { itemId, currentStock, threshold, locationId }
supply.reorder.triggered    → { itemId, quantityRequested, locationId }
```

**Database:** `supplies_db`

---

### 3.9 Assets Service

**Responsibility:** Non-consumable asset register, assignment lifecycle, fault reporting, maintenance tracking, retirement.

**Owns:**
- `asset_categories` table
- `assets` table
- `asset_assignments` table
- `asset_requests` table
- `maintenance_records` table
- `fault_reports` table

**APIs:**
```
GET    /api/v1/assets                               → Asset register (HR/Facilities)
POST   /api/v1/assets                               → Add asset to register
GET    /api/v1/assets/{id}                          → Asset detail
PATCH  /api/v1/assets/{id}/retire                   → Retire asset
GET    /api/v1/assets/my                            → Own assigned assets (Employee)
POST   /api/v1/asset-assignments                    → Assign asset to user
PATCH  /api/v1/asset-assignments/{id}/acknowledge   → Employee acknowledges receipt
PATCH  /api/v1/asset-assignments/{id}/return        → Return asset
POST   /api/v1/asset-requests                       → Submit asset request
PATCH  /api/v1/asset-requests/{id}/approve          → Manager approve
PATCH  /api/v1/asset-requests/{id}/fulfil           → Facilities Admin fulfil
POST   /api/v1/fault-reports                        → Submit fault report
POST   /api/v1/maintenance-records                  → Log maintenance event
GET    /api/v1/assets/{id}/audit-log                → Full asset audit trail
```

**Kafka Events Produced:**
```
asset.assigned            → { assetId, userId, locationId }
asset.acknowledged        → { assetId, userId, locationId }
asset.returned            → { assetId, userId, locationId }
asset.fault_reported      → { assetId, userId, faultType, locationId }
asset.maintenance_logged  → { assetId, maintenanceType, locationId }
asset.retired             → { assetId, reason, locationId }
```

**Database:** `assets_db`

---

### 3.10 Notification Service

**Responsibility:** In-app and email notification dispatch. Consumes events from all other services and routes notifications to the correct recipients via the correct channel.

**Owns:**
- `notifications` table (in-app notification records)
- `notification_templates` table

**Communication:** Kafka consumer only. No REST API called by other services. Triggered entirely by events.

**Kafka Events Consumed and Notification Triggered:**

| Event | Recipient | Channels |
|-------|-----------|----------|
| `remote.request.submitted` | Manager / delegate | In-app + Email |
| `remote.request.approved` | Employee | In-app + Email |
| `remote.request.rejected` | Employee | In-app + Email |
| `ooo.request.approved` | Delegate nominated | In-app + Email |
| `seat.booking.created` | Employee | In-app |
| `seat.booking.released` | Facilities Admin (daily summary) | In-app + Email |
| `visitor.checked_in` | Host employee | In-app + Email |
| `event.rsvp.confirmed` | Employee | In-app + Email |
| `event.rsvp.promoted` | Employee | In-app + Email |
| `event.cancelled` | All attendees | In-app + Email |
| `event.reminder` | All attendees | In-app + Email |
| `supply.stock.low` | Facilities Admin | In-app + Email |
| `supply.request.approved` | Employee | In-app + Email |
| `supply.request.rejected` | Employee | In-app + Email |
| `asset.assigned` | Employee | In-app + Email |
| `asset.fault_reported` | Facilities Admin | In-app |
| `asset.maintenance_logged` | Facilities Admin | In-app |

**APIs (for in-app reads):**
```
GET    /api/v1/notifications/my        → Own unread notifications
PATCH  /api/v1/notifications/{id}/read → Mark as read
PATCH  /api/v1/notifications/read-all  → Mark all as read
```

**Database:** `notification_db`

---

### 3.11 Audit Service

**Responsibility:** Immutable, append-only audit log of every significant action across all services. Read-only query API for authorised roles.

**Owns:**
- `audit_logs` table — append-only, no deletes, no updates

**Communication:** Kafka consumer only. All services publish `audit.*` events; this service persists them.

**Kafka Events Consumed:**
```
audit.event  → { actorId, actorRole, action, entityType, entityId, locationId,
                  previousState (JSONB), newState (JSONB), occurredAt }
```

Every service that mutates state publishes an `audit.event` to Kafka before (or as part of) the mutation. The Audit Service persists these records. Write access is restricted to the Kafka consumer — no service can call an audit write API directly.

**APIs:**
```
GET    /api/v1/audit-logs                         → Paginated audit log (HR/Facilities/Super Admin)
GET    /api/v1/audit-logs?entityType=SeatBooking  → Filter by entity type
GET    /api/v1/audit-logs?actorId={userId}        → Filter by actor
GET    /api/v1/audit-logs?locationId={locationId} → Filter by location
```

**Retention:** 24 months active. Background archival job moves older records to cold storage (S3).

**Database:** `audit_db` — PostgreSQL with append-only enforcement (no UPDATE or DELETE permissions granted to the application user).

---

## 4. Communication Patterns

### 4.1 Synchronous Communication (REST)

Used for: real-time queries, user-facing operations, operations requiring immediate confirmation.

**Requirements on every inter-service REST call:**

```yaml
timeout: 3000ms          # Never block indefinitely
retry:
  max-attempts: 3
  backoff: exponential   # 100ms, 200ms, 400ms
circuit-breaker:
  failure-threshold: 50% # Open after 50% failure rate
  wait-duration: 10s     # Stay open for 10s before half-open
fallback: defined        # Always define a degraded response
```

**Implementation (Resilience4j + Spring Boot):**

```java
@CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
@Retry(name = "user-service")
@TimeLimiter(name = "user-service")
public CompletableFuture<UserResponse> getUser(UUID userId) {
    return CompletableFuture.supplyAsync(() ->
        userServiceClient.getUser(userId));
}

private CompletableFuture<UserResponse> getUserFallback(UUID userId, Exception ex) {
    log.warn("user-service unavailable, returning cached/default for {}", userId);
    return CompletableFuture.completedFuture(UserResponse.defaultResponse(userId));
}
```

### 4.2 Asynchronous Communication (Kafka)

Used for: state change notifications, cross-service workflows, attendance overlay, notification dispatch.

**Rules:**
- All events are idempotent — consumers must handle duplicate delivery without side effects.
- All events carry a `correlationId` for distributed tracing.
- Event schema is versioned. Breaking changes require a new event type (e.g. `remote.request.approved.v2`).
- Consumers commit offsets only after successful processing.

**Kafka Topic Conventions:**
```
oms.{domain}.{entity}.{action}
Examples:
  oms.remote.request.approved
  oms.seating.booking.released
  oms.attendance.record.resolved
  oms.audit.event
```

**Event Envelope (all events):**
```json
{
  "eventId": "uuid-v4",
  "eventType": "remote.request.approved",
  "version": "1",
  "correlationId": "uuid-v4",
  "occurredAt": "2026-03-01T09:00:00Z",
  "locationId": "uuid",
  "payload": { ... }
}
```

**Idempotency enforcement:**
```java
@KafkaListener(topics = "oms.remote.request.approved")
@Transactional
public void handleRemoteApproved(RemoteApprovedEvent event) {
    if (processedEventRepository.existsById(event.getEventId())) {
        log.info("Duplicate event ignored: {}", event.getEventId());
        return;
    }
    attendanceService.overlayRemoteStatus(event.getUserId(), event.getDates(), event.getLocationId());
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

### 4.3 Communication Matrix

```
                    identity  user  attendance  seating  remote  notification  audit
identity-service       -      REST      -          -        -         -          ✦
user-service           -       -        ✦          -        -         ✦          ✦
attendance-service     -      REST      -          -       [K]        ✦          ✦
seating-service        -      REST      [K]         -        -         ✦          ✦
remote-service         -      REST      [K]         -        -         ✦          ✦
visitor-service        -      REST      -           -        -         ✦          ✦
event-service          -      REST      -           -        -         ✦          ✦
supplies-service       -      REST      -           -        -         ✦          ✦
assets-service         -      REST      -           -        -         ✦          ✦
notification-service   -       -        -           -        -         -           -
audit-service          -       -        -           -        -         -           -

REST = synchronous REST call
[K]  = Kafka event consumed
✦    = Kafka event produced (to notification/audit bus)
```

---

## 5. API Gateway

All external traffic enters through the API Gateway. No service is exposed directly to the internet.

**Responsibilities:**

| Concern | Implementation |
|---------|---------------|
| Authentication | Validates session cookie; calls `identity-service /api/v1/auth/validate` |
| Routing | Routes requests to correct downstream service by path prefix |
| Rate limiting | Per-user and per-IP rate limits (configurable) |
| Request logging | Logs all inbound requests with correlation ID |
| CORS | Configured per environment |
| TLS termination | HTTPS terminated at gateway; mTLS within service mesh |

**Routing table:**
```
/api/v1/auth/**          → identity-service
/api/v1/users/**         → user-service
/api/v1/locations/**     → user-service
/api/v1/attendance/**    → attendance-service
/api/v1/seat-*/**        → seating-service
/api/v1/seats/**         → seating-service
/api/v1/remote-*/**      → remote-service
/api/v1/ooo-*/**         → remote-service
/api/v1/visitors/**      → visitor-service
/api/v1/visits/**        → visitor-service
/api/v1/events/**        → event-service
/api/v1/supplies/**      → supplies-service
/api/v1/assets/**        → assets-service
/api/v1/notifications/** → notification-service
/api/v1/audit-logs/**    → audit-service
```

**Business logic rule:** The API Gateway contains **zero business logic**. It authenticates, routes, and logs. Nothing else.

---

## 6. Security Architecture (Zero Trust)

**Principle: Never trust. Always verify.** Every service validates the caller's identity on every request, regardless of whether the call originates from the gateway or another internal service.

### 6.1 External Security

- All external traffic over HTTPS (TLS 1.2+).
- Session tokens are HTTP-only cookies — inaccessible to JavaScript.
- API Gateway validates session on every inbound request before routing.
- Rate limiting enforced at the gateway.

### 6.2 Internal Security (Service-to-Service)

- All internal service communication uses **mutual TLS (mTLS)** — both parties present certificates.
- Services exchange a short-lived **internal JWT** signed by a shared private key (issued by `identity-service`). Every service verifies this JWT on every inbound call.
- No service is directly accessible from outside the cluster.

### 6.3 Authorisation

- **RBAC:** roles are EMPLOYEE, MANAGER, HR, FACILITIES_ADMIN, SUPER_ADMIN.
- **Location scoping:** every role check is coupled with a location check. A Manager in Accra cannot approve requests for Lagos.
- Authorisation is enforced at the **service layer** of each service — not only at the gateway.
- Super Admin has a NULL location_id on their UserRole record — the only cross-location exception.

### 6.4 Data Security

```java
// Rule: parameterised queries ONLY. No string concatenation. Ever.
@Query("SELECT a FROM AttendanceRecord a WHERE a.userId = :userId AND a.locationId = :locationId AND a.deletedAt IS NULL")
List<AttendanceRecord> findActiveByUserAndLocation(
    @Param("userId") UUID userId,
    @Param("locationId") UUID locationId
);
```

- Secrets managed via environment variables and AWS Secrets Manager.
- Database credentials are never shared between services.
- Each service has its own DB user with minimum required privileges.

### 6.5 Audit Trail

Every mutation publishes an `audit.event` to Kafka before completing. The Audit Service persists this immutably. There is no API to write or delete audit records — only the Kafka consumer can do so.

---

## 7. Resilience and Fault Tolerance

| Pattern | Applied To | Implementation |
|---------|-----------|----------------|
| Circuit Breaker | All synchronous service calls | Resilience4j `@CircuitBreaker` |
| Retry with backoff | All synchronous service calls | Resilience4j `@Retry`, exponential backoff |
| Timeout | All synchronous service calls | 3000ms hard limit |
| Fallback | All synchronous service calls | Always defined; return degraded/cached response |
| Idempotent consumers | All Kafka consumers | `processed_events` deduplication table |
| Graceful shutdown | All services | Spring Boot graceful shutdown; drain in-flight requests |
| Health checks | All services | `/actuator/health` — liveness + readiness probes |
| Bulkhead | Attendance sync job | Isolated thread pool; sync failure does not affect API requests |

**Failure modes and degraded behaviour:**

| Failure | Degraded Behaviour |
|---------|--------------------|
| `user-service` down | Return cached user profile; log warning; alert |
| `attendance-service` down | Attendance data unavailable; other features unaffected |
| Kafka unavailable | Services continue operating; audit/notification events queued and replayed on reconnect |
| Nightly sync job fails | Alert fired; previous day's data available; retry on next scheduled run |
| `identity-service` down | All logins blocked; existing sessions continue until expiry |

---

## 8. Observability

Every service implements the full observability stack. There are no exceptions.

### 8.1 Structured Logging (JSON)

```json
{
  "timestamp": "2026-03-01T09:15:42.123Z",
  "level": "INFO",
  "service": "seating-service",
  "correlationId": "d4e2f9a1-...",
  "userId": "uuid",
  "locationId": "uuid",
  "action": "SEAT_BOOKED",
  "seatId": "uuid",
  "message": "Hot-desk booking confirmed for 2026-03-02"
}
```

**Rules:**
- All logs output to **stdout** (12-Factor: logs as streams).
- No local log files.
- Centralised log aggregator (CloudWatch Logs or ELK) consumes stdout.
- Never log passwords, tokens, or sensitive PII.

### 8.2 Metrics (Prometheus)

Each service exposes `/actuator/prometheus`. Key metrics per service:

```
http_request_duration_seconds        # API latency by endpoint
http_requests_total                  # Request count by status code
job_duration_seconds                 # Scheduled job execution time
job_failures_total                   # Scheduled job failure count
kafka_consumer_lag                   # Consumer group lag per topic
circuit_breaker_state                # CLOSED / OPEN / HALF_OPEN
db_connection_pool_active            # Active DB connections
```

### 8.3 Distributed Tracing (OpenTelemetry)

Every request carries a `correlationId` (generated at the API Gateway, propagated via HTTP header `X-Correlation-ID` to all downstream calls and Kafka messages). OpenTelemetry SDK is included in every service.

```
Browser → API Gateway → seating-service → user-service
             [corr-id: abc123]  [corr-id: abc123]  [corr-id: abc123]
```

Traces are exported to Jaeger or AWS X-Ray. A single `correlationId` traces the complete path of a request across all services.

### 8.4 Health Checks

Every service exposes:
```
GET /actuator/health/liveness   → Is the process alive?
GET /actuator/health/readiness  → Is the service ready to handle traffic?
```

Kubernetes uses these for liveness and readiness probes. A service failing its readiness check is removed from the load balancer pool until it recovers.

### 8.5 Alerting

| Alert | Threshold | Destination |
|-------|-----------|-------------|
| Badge sync job failure | Any failure | PagerDuty / Slack |
| No-show release job failure | Any failure | Slack |
| Service error rate (5xx) | > 1% over 5 min | PagerDuty |
| Consumer group lag | > 10,000 messages | Slack |
| Circuit breaker OPEN | Any service | PagerDuty |
| DB connection pool saturation | > 80% | Slack |

---

## 9. Data Management

### 9.1 Database Per Service (Mandatory)

| Service | Database Name | Key Characteristics |
|---------|--------------|---------------------|
| identity-service | `identity_db` | Minimal schema; sessions only |
| user-service | `user_db` | Authoritative user and role data |
| attendance-service | `attendance_db` | Immutable BadgeEvents; high write volume nightly |
| seating-service | `seating_db` | Real-time availability queries; high read volume |
| remote-service | `remote_db` | Approval workflow state |
| visitor-service | `visitor_db` | Visit records; agreement versioning |
| event-service | `event_db` | Event series and invite management |
| supplies-service | `supplies_db` | Batch-level stock entries |
| assets-service | `assets_db` | Asset lifecycle; maintenance history |
| notification-service | `notification_db` | In-app notification records |
| audit-service | `audit_db` | Append-only; never modified |

**No service accesses another service's database. No shared schemas. No cross-service joins. This rule is absolute.**

### 9.2 Distributed Transaction Patterns

Where a business operation spans multiple services, the **Choreography-based Saga** pattern is used.

**Example: Supply Request Approval Flow**

```
1. supplies-service: status → PENDING_MANAGER
   Publishes: supply.request.submitted
        ↓
2. notification-service: notifies Manager
        ↓
3. Manager approves via supplies-service API
   supplies-service: status → PENDING_FACILITIES
   Publishes: supply.request.approved (stage 1)
        ↓
4. notification-service: notifies Facilities Admin
        ↓
5. Facilities Admin fulfils via supplies-service API
   supplies-service: status → FULFILLED; decrements stock
   Publishes: supply.request.fulfilled
        ↓
6. notification-service: notifies Employee
7. audit-service: persists full trail
```

Each step is local to its service. No distributed transaction coordinator. Compensation (rollback) is handled by publishing a reversal event if a step fails downstream.

### 9.3 Cross-Service Queries (API Composition)

Where a query requires data from multiple services (e.g. a manager's dashboard showing team attendance + remote schedule + bookings), the **API Composition** pattern is used: the API Gateway or a lightweight BFF (Backend for Frontend) layer calls multiple services in parallel and aggregates the response.

```
Dashboard request
     ↓
API Composition Layer
  ├── attendance-service: GET /team/{id}/attendance
  ├── remote-service:     GET /teams/{id}/schedule
  └── seating-service:    GET /seat-bookings/team/{id}
     ↓
Composed response returned to client
```

---

## 10. Configuration Management (12-Factor Compliance)

All configuration is managed via **environment variables only**. No hardcoded values. No config files committed to source control.

```bash
# identity-service
SSO_ISSUER_URI=https://sso.yourorg.com
SSO_JWKS_URI=https://sso.yourorg.com/.well-known/jwks.json
SESSION_EXPIRY_SECONDS=3600

# attendance-service
ATHENA_DATABASE=badge_events
ATHENA_OUTPUT_BUCKET=s3://oms-athena-results/
AWS_REGION=eu-west-1
BADGE_SYNC_CRON=0 2 * * *          # 2am daily
SESSION_GAP_THRESHOLD_HOURS=6

# seating-service
BOOKING_WINDOW_DAYS=14
CANCELLATION_CUTOFF_HOURS=2
NO_SHOW_RELEASE_CRON=0 10 * * *    # 10am daily

# All services — common
DB_URL=jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
DB_USERNAME=${DB_USER}
DB_PASSWORD=${DB_PASSWORD}
KAFKA_BOOTSTRAP_SERVERS=${KAFKA_HOST}:9092
INTERNAL_JWT_SECRET=${INTERNAL_JWT_SECRET}
```

**Three environments:** `dev`, `staging`, `prod`. Promoted identically through CI/CD — only environment variables differ. Dev/prod parity is maintained.

---

## 11. Deployment and Infrastructure

### 11.1 Containerisation

Each service is a single Docker container. One `Dockerfile` per service:

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 11.2 Kubernetes Architecture

```yaml
# Per service: Deployment + Service + HorizontalPodAutoscaler

apiVersion: apps/v1
kind: Deployment
metadata:
  name: seating-service
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: seating-service
          image: ecr.aws/oms/seating-service:v1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: seating-service-secrets
                  key: db-password
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: seating-service-hpa
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          averageUtilization: 70
```

### 11.3 CI/CD Pipeline

```
Developer → Pull Request
     ↓
[CI Pipeline — GitHub Actions]
  1. Lint (Checkstyle / ESLint)
  2. Unit tests
  3. Integration tests (TestContainers)
  4. Contract tests (Pact)
  5. Security scan (OWASP dependency check)
  6. Docker build
  7. Push to ECR (on merge to develop/staging/main)
     ↓
Auto-deploy → DEV (on develop merge)
Manual gate → STAGING
Manual gate → PRODUCTION (with rollback plan)
```

### 11.4 Service Dependencies Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| API Gateway | AWS API Gateway or Kong | External routing, auth, rate limiting |
| Message broker | Apache Kafka (MSK) | Async event bus |
| Service databases | AWS RDS PostgreSQL (per service) | Isolated data stores |
| Container registry | AWS ECR | Docker image storage |
| Orchestration | AWS EKS (Kubernetes) | Container management and scaling |
| Secrets | AWS Secrets Manager | Runtime secret injection |
| Logging | CloudWatch Logs / ELK | Centralised log aggregation |
| Metrics | Prometheus + Grafana | Dashboards and alerting |
| Tracing | OpenTelemetry + Jaeger | Distributed trace correlation |
| Config | Kubernetes ConfigMaps + Secrets | Environment-based config |

---

## 12. Testing Strategy

| Level | Tool | Scope |
|-------|------|-------|
| Unit | JUnit 5 + Mockito | All service logic; job resolution logic; policy enforcement |
| Integration | Spring Boot Test + TestContainers | Repository queries; Kafka producer/consumer pairs; full API flows |
| Contract | Pact | API contracts between consumer and provider services |
| Security | MockMvc + role simulation | Every endpoint against every role+location combination |
| Resilience | Chaos Monkey / Resilience4j test mode | Circuit breaker, retry, timeout behaviour |
| Performance | k6 | Nightly sync job throughput; seat availability under concurrency |

**Contract testing rule:** When `seating-service` calls `user-service`, a Pact contract is defined by the consumer (`seating-service`) and verified by the provider (`user-service`) in CI. This prevents breaking API changes from being deployed silently.

---

## 13. Anti-Patterns — Enforced Prohibitions

| Anti-Pattern | Status | Enforcement |
|-------------|--------|-------------|
| Shared database between services | ❌ FORBIDDEN | Separate DB users; network isolation |
| Service A reads Service B's database | ❌ FORBIDDEN | No shared credentials; no shared schemas |
| Business logic in API Gateway | ❌ FORBIDDEN | Gateway routes and authenticates only |
| Hardcoded secrets or config | ❌ FORBIDDEN | CI scan fails on detected secrets |
| Stateful service (in-memory session) | ❌ FORBIDDEN | Sessions stored in DB or cache; pods are stateless |
| Synchronous chain > 2 hops | ❌ AVOID | Use Kafka events for longer workflows |
| Unbounded database queries | ❌ FORBIDDEN | All list endpoints paginated |
| Entity exposed in API response | ❌ FORBIDDEN | DTOs only; entities never leave the service layer |
| String concatenation in SQL | ❌ FORBIDDEN | Parameterised queries enforced in code review |
| Distributed monolith (tight coupling via sync) | ❌ FORBIDDEN | Prefer async Kafka for state changes |

---

## 14. Decision Log — Microservices Specific

| Decision | Rationale |
|---------|-----------|
| 11 services, not fewer | Each service maps to exactly one bounded context; no responsibilities shared |
| Choreography-based Saga (not Orchestration) | Avoids a central orchestrator becoming a single point of failure; each service reacts to events autonomously |
| Kafka over RabbitMQ | Kafka's durable, replayable log is required for the attendance overlay use case; notifications and audit benefit from replay capability |
| Pact contract testing | With 11 services, integration test environments are expensive; contract tests verify compatibility without full integration environments |
| PostgreSQL per service (not polyglot) | All services share the same data model characteristics (relational, JSONB, UUID keys); polyglot adds operational overhead without benefit at this scale |
| No CQRS at launch | Read/write patterns are not complex enough to justify CQRS overhead; can be introduced per service if read/write traffic diverges |
| mTLS + internal JWT for service-to-service auth | Zero Trust — no service trusts another based on network position alone |

---

*End of Microservices Architecture Document*
