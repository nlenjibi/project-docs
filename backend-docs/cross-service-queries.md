# Cross-Service Queries
## Office Management System (OMS)

**Version:** 3.0

In a microservices architecture, each service owns its database exclusively. There are **no cross-database SQL JOINs** — ever. Every service's database user has credentials only for its own database; connecting to another service's database is physically impossible.

This document lists every place where one OMS service needs data that belongs to another, and exactly how each case is handled.

---

## The Four Patterns

| Pattern | When to Use | Trade-off |
|---------|------------|-----------|
| **A — Local Read-Model** | Frequently queried fields needed on almost every request (e.g. user display names on every attendance row) | Slight eventual consistency (seconds); extra table + Kafka/RabbitMQ consumer to maintain |
| **B — Synchronous Feign Call** | Real-time lookups on low-frequency or write-path operations where stale data would be incorrect | Adds latency; circuit breaker required; introduces availability dependency |
| **C — RabbitMQ Event-Driven** | State changes that other services must react to | Async; eventual consistency; no query needed |
| **D — Parallel API Composition** | Aggregating data from multiple services for a single frontend response | Combines B instances in parallel; each has its own circuit breaker |

**Pattern A and C dominate.** Pattern B is used only where freshness is mandatory and event frequency is low. Pattern D is reserved for dashboard aggregation.

---

## Pattern A — Local Read-Model (RabbitMQ-Driven)

When a service needs another service's fields on almost every query (e.g., user display names appear on every attendance record in an export), making an API call each time is expensive and creates an availability dependency on `auth-service`. Instead, each service maintains a **local copy of only the fields it needs**, populated and kept current via RabbitMQ events.

**Why this is preferred:** Query performance is local (no network hop). The service remains fully operational even if `auth-service` is temporarily unavailable. The consistency lag is seconds — acceptable for display data like names and emails.

**Why eventual consistency is acceptable here:** Display names and emails change rarely. A stale name in an attendance export (showing "Alice M" instead of "Alice Mensah" for 30 seconds after a name update) is a tolerable trade-off for always-available queries. If strict consistency were required (e.g., for billing), Pattern B would be used instead.

---

### A1 — user_summaries table (replicated into multiple services)

The following services maintain a local `user_summaries` read-model, populated by consuming RabbitMQ events from `auth-service`:

| Service | Why it needs user_summaries |
|---------|----------------------------|
| `attendance-service` | Display names on reports and exports |
| `seating-service` | Occupant names on floor plan |
| `remote-service` | Team schedule calendar display names |
| `notification-service` | Recipient email + name for every dispatch |
| `audit-service` | Actor display name in audit log query results |
| `inventory-service` | Requester/assignee names on inventory reports |
| `workplace-service` | Host name on visitor history; attendee names on event attendee list |

**The table (identical in every service that uses it):**

```sql
-- Local read-model — NOT the source of truth
-- Source of truth: auth-service / auth_db / users table
CREATE TABLE user_summaries (
    user_id      UUID PRIMARY KEY,
    display_name VARCHAR(255) NOT NULL,
    email        VARCHAR(255) NOT NULL,
    is_active    BOOLEAN      NOT NULL DEFAULT TRUE,
    synced_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX idx_user_summaries_active ON user_summaries(user_id) WHERE is_active = TRUE;
```

**RabbitMQ consumer — identical in every service that holds `user_summaries`:**

```java
// Handles user.created, user.updated, user.deactivated
@Bean
public Consumer<OmsEvent<UserPayload>> handleUserEvent() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        boolean isActive = !"oms.user.deactivated".equals(event.eventType());

        userSummaryRepository.upsert(
            event.payload().userId(),
            event.payload().displayName(),
            event.payload().email(),
            isActive
        );

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}
```

**Repository upsert (PostgreSQL ON CONFLICT):**

```java
@Modifying
@Query(value = """
    INSERT INTO user_summaries (user_id, display_name, email, is_active, synced_at)
    VALUES (:userId, :displayName, :email, :isActive, now())
    ON CONFLICT (user_id) DO UPDATE
      SET display_name = EXCLUDED.display_name,
          email        = EXCLUDED.email,
          is_active    = EXCLUDED.is_active,
          synced_at    = now()
    """, nativeQuery = true)
void upsert(
    @Param("userId")      UUID userId,
    @Param("displayName") String displayName,
    @Param("email")       String email,
    @Param("isActive")    boolean isActive);
```

With this in place, every service can JOIN locally at query time — zero network calls.

---

### A2 — personnel_id_mapping table (attendance-service)

The nightly badge sync receives a `personnel_id` from AWS Athena (the Arms HR system identifier). `attendance-service` must resolve it to an OMS `user_id`. Making a Feign call to `auth-service` during the nightly bulk sync would create an N×API-call bottleneck (one call per badge row) and an availability dependency at 2am.

**Solution:** `attendance-service` maintains its own `personnel_id_mapping` table, populated via RabbitMQ events when users are created in `auth-service`.

```sql
CREATE TABLE personnel_id_mapping (
    personnel_id VARCHAR(100) PRIMARY KEY,  -- Arms HR ID from Athena
    user_id      UUID         NOT NULL,     -- OMS UUID
    synced_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

**RabbitMQ consumer (attendance-service only):**

```java
@Bean
public Consumer<OmsEvent<UserCreatedPayload>> handleUserCreated() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        if (event.payload().personnelId() != null) {
            personnelIdMappingRepository.upsert(
                event.payload().personnelId(),
                event.payload().userId()
            );
        }

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}
```

**Used during nightly BadgeSyncJob — fully local, zero Feign calls:**

```java
// BadgeSyncJob — resolve Athena personnel_id to OMS user_id locally
UUID userId = personnelIdMappingRepository
    .findByPersonnelId(athenaRow.getPersonnelId())
    .map(PersonnelIdMapping::getUserId)
    .orElse(null); // null = unresolved; badge event stored with user_id = NULL; logged as WARN
```

The entire nightly sync runs without calling `auth-service`. This is critical for reliability — the badge sync must complete even if `auth-service` is restarting.

---

## Pattern B — Synchronous Feign Call (Real-Time Lookup)

Used only where data must be real-time and the operation is infrequent enough that a network call is acceptable.

---

### B1 — auth-service resolves roles at login

**Service:** `auth-service` (internal — identity module calling user module)
**Trigger:** SSO callback — happens once per login session
**Why no fallback:** Issuing a session without verified roles would be a security violation. Login failure is the correct degraded behaviour when role data is unavailable.

**Note:** With the `identity-service + user-service → auth-service` merge, this is now an in-process call (same JVM), not a network hop. The merge eliminates a full round-trip on every login.

---

### B2 — attendance-service: team member IDs for a manager

**Service:** `attendance-service` calling `auth-service`
**Fields:** List of user IDs reporting to a given manager at a location
**Trigger:** `GET /api/v1/attendance/team/{managerId}` — one call per team attendance query

```java
// AttendanceService
public Page<AttendanceRecordResponse> getTeamAttendance(
        UUID managerId, UUID locationId, DateRange range, Pageable pageable) {

    // 1. Who reports to this manager? — one Feign call
    List<UUID> teamUserIds = authServiceClient.getDirectReports(managerId, locationId);
    // Fallback: if auth-service is down → return empty page (not a 500)

    if (teamUserIds.isEmpty()) {
        return Page.empty(pageable);
    }

    // 2. Query own DB using those IDs + local user_summaries JOIN — no second Feign call
    return attendanceRepository.findByUsersAndDateRange(teamUserIds, locationId, range, pageable);
}
```

```java
// AttendanceRepository — fully local JOIN on user_summaries
@Query("""
    SELECT a.record_date,
           a.status,
           a.work_duration_minutes,
           a.is_late,
           u.display_name,
           u.email
    FROM   attendance_records a
    JOIN   user_summaries u ON a.user_id = u.user_id
    WHERE  a.user_id IN :userIds
    AND    a.location_id = :locationId
    AND    a.record_date BETWEEN :from AND :to
    AND    a.deleted_at IS NULL
    ORDER  BY a.record_date DESC, u.display_name
    """)
Page<AttendanceWithUserView> findByUsersAndDateRange(...);
```

The REST call fetches only the team member UUIDs. All display enrichment uses the local `user_summaries` table.

---

### B3 — remote-service: resolving the fallback approver

**Service:** `remote-service` calling `auth-service`
**Fields:** User ID of the HR role holder at a given location
**Trigger:** Manager has no active `ApprovalDelegate` and a pending approval needs routing

```java
// ApprovalRoutingService inside remote-service
public UUID resolveApprover(UUID managerId, UUID locationId) {

    // 1. Check local approval_delegates table first — zero API calls
    Optional<ApprovalDelegate> delegate = approvalDelegateRepository
        .findActiveDelegate(managerId, locationId, LocalDate.now());

    if (delegate.isPresent()) {
        return delegate.get().getDelegateId();
    }

    // 2. No delegate — call auth-service to find HR at this location
    UserSummaryResponse hrUser = authServiceClient.getRoleHolder(locationId, "HR");
    // GET /api/v1/locations/{locationId}/role-holder?role=HR
    // Fallback: if auth-service down → throw ServiceUnavailableException
    // (cannot route approvals without a valid approver)
    return hrUser.getUserId();
}
```

---

### B4 — workplace-service: validate host employee at location

**Service:** `workplace-service` calling `auth-service`
**Fields:** Confirm `hostUserId` exists and is active at `locationId`
**Trigger:** `POST /api/v1/visitors/pre-register` — once per pre-registration

```java
// VisitorService
public VisitResponse preRegister(VisitorPreRegisterRequest request) {

    // Validate host — REST call (once at registration time, not on every query)
    UserSummaryResponse host = authServiceClient.getUser(request.getHostUserId());

    if (!host.isActive() || !host.getLocationId().equals(request.getLocationId())) {
        throw new InvalidHostException(
            "Host employee not found or not active at this location.");
    }

    // From here, use only local data
    VisitorProfile profile = visitorProfileRepository
        .findByEmail(request.getEmail())
        .orElseGet(() -> visitorProfileRepository.save(
            new VisitorProfile(request.getVisitorName(), request.getEmail())));

    ParentVisit visit = new ParentVisit(
        profile.getId(), request.getHostUserId(),
        request.getLocationId(), request.getExpectedDate());

    parentVisitRepository.save(visit);
    return mapToResponse(visit, profile, host.getDisplayName());
}
```

---

### B5 — audit-service: actor name enrichment at query time (via local read-model)

`audit-service` maintains its own `user_summaries` table (Pattern A). The actor name JOIN is fully local — no Feign call at query time.

```java
// AuditLogRepository — local JOIN on user_summaries
@Query("""
    SELECT al.id,
           al.action,
           al.entity_type,
           al.entity_id,
           al.location_id,
           al.previous_state,
           al.new_state,
           al.occurred_at,
           al.actor_role,
           al.correlation_id,
           u.display_name  AS actor_name,
           u.email         AS actor_email
    FROM   audit_logs al
    JOIN   user_summaries u ON al.actor_id = u.user_id
    WHERE  (:locationId IS NULL OR al.location_id = :locationId)
    AND    (:entityType IS NULL OR al.entity_type = :entityType)
    AND    (:actorId    IS NULL OR al.actor_id    = :actorId)
    AND    (:action     IS NULL OR al.action      = :action)
    AND    al.occurred_at BETWEEN :from AND :to
    ORDER  BY al.occurred_at DESC
    """)
Page<AuditLogView> findWithFilters(..., Pageable pageable);
```

SUPER_ADMIN passes `locationId = NULL` → the `(:locationId IS NULL OR ...)` clause returns all locations without a separate query path.

---

### B6 — seating-service: floor plan occupant names (via local read-model)

Occupant names on the floor plan come from local `user_summaries`. No cross-service call at query time — the floor plan can render even if `auth-service` is temporarily unavailable.

```java
// FloorPlanRepository — fully local
@Query("""
    SELECT s.id          AS seat_id,
           s.seat_label,
           s.seat_type,
           s.is_bookable,
           sb.id         AS booking_id,
           sb.status,
           u.display_name AS occupant_name,
           u.user_id      AS occupant_id
    FROM   seats s
    LEFT JOIN seat_bookings sb
           ON sb.seat_id      = s.id
           AND sb.booking_date = :date
           AND sb.status       = 'CONFIRMED'
           AND sb.deleted_at   IS NULL
    LEFT JOIN user_summaries u ON sb.user_id = u.user_id
    WHERE  s.zone_id IN (
               SELECT id FROM zones WHERE floor_id IN (
                   SELECT id FROM floors WHERE location_id = :locationId
               )
           )
    AND    s.deleted_at IS NULL
    ORDER  BY s.seat_label
    """)
List<SeatFloorPlanView> findFloorPlanWithOccupants(
    @Param("locationId") UUID locationId,
    @Param("date")       LocalDate date);
```

---

## Pattern C — RabbitMQ Event-Driven (No Query Required)

These are not queries. One service publishes an event; another service reacts and updates its own local state. No JOIN, no Feign call.

---

### C1 — Attendance overlay: REMOTE and OOO statuses

**Trigger:** `remote-service` approves a remote-day or OOO request
**Routing keys:** `oms.remote.request.approved`, `oms.ooo.request.approved`
**Consumer:** `attendance-service` (queue: `oms.attendance.remote`)

`attendance-service` does NOT query `remote-service`'s database. `remote-service` puts all needed information in the event payload. `attendance-service` updates its own `attendance_records` table using an upsert.

```java
// attendance-service: consume remote.request.approved
@Bean
public Consumer<OmsEvent<RemoteApprovedPayload>> handleRemoteApproved() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        for (LocalDate date : event.payload().dates()) {
            attendanceRepository.upsertStatus(
                event.payload().userId(),
                date,
                event.locationId(),
                "REMOTE"
            );
        }

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}

// attendance-service: consume ooo.request.approved
@Bean
public Consumer<OmsEvent<OooApprovedPayload>> handleOooApproved() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        LocalDate cursor = event.payload().startDate();
        while (!cursor.isAfter(event.payload().endDate())) {
            attendanceRepository.upsertStatus(
                event.payload().userId(), cursor, event.locationId(), "OOO");
            cursor = cursor.plusDays(1);
        }

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}
```

```java
// AttendanceRepository upsert — ON CONFLICT ensures idempotency
@Modifying
@Query(value = """
    INSERT INTO attendance_records (id, user_id, location_id, record_date, status, created_at)
    VALUES (gen_random_uuid(), :userId, :locationId, :date, :status, now())
    ON CONFLICT (user_id, record_date, location_id)
    DO UPDATE SET status = EXCLUDED.status, updated_at = now()
    """, nativeQuery = true)
void upsertStatus(
    @Param("userId")     UUID userId,
    @Param("date")       LocalDate date,
    @Param("locationId") UUID locationId,
    @Param("status")     String status);
```

---

### C2 — No-show seat release: did the user badge in?

**Trigger:** `attendance-service` detects no badge-in for a user who had a confirmed booking
**Routing key:** `oms.attendance.no_show.detected`
**Consumer:** `seating-service` (queue: `oms.seating.user-events`)

`seating-service` does NOT query `attendance_db`. `attendance-service` detects the no-show and publishes the event. `seating-service` then queries only its own tables.

```java
// attendance-service: NoShowDetectionService (part of nightly BadgeSyncJob)
public void detectNoShows(LocalDate date, UUID locationId) {
    List<UUID> noShowUserIds = workSessionRepository
        .findUsersWithBookingButNoSessionOnDate(date, locationId);
    // Queries only attendance_db — no cross-service call

    for (UUID userId : noShowUserIds) {
        eventPublisher.publishEvent(new NoShowDetectedDomainEvent(userId, date, locationId));
    }
}
```

```java
// seating-service: consume no_show.detected
@Bean
public Consumer<OmsEvent<NoShowDetectedPayload>> handleNoShow() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        List<SeatBooking> bookings = seatBookingRepository
            .findConfirmedByUserAndDate(
                event.payload().userId(),
                event.payload().date(),
                event.locationId()
            );

        for (SeatBooking booking : bookings) {
            booking.setStatus(BookingStatus.RELEASED);
            seatBookingRepository.save(booking);
            noShowRecordRepository.save(new NoShowRecord(booking));
            // Publishes oms.seating.booking.released → notification-service + audit-service
            eventPublisher.publishEvent(new BookingReleasedDomainEvent(booking));
        }

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}
```

---

### C3 — User deactivation: cancel bookings and stop attendance records

**Trigger:** `auth-service` deactivates a user (HR offboarding or Arms sync)
**Routing key:** `oms.user.deactivated`
**Consumers:** `attendance-service`, `seating-service` (both in queue `oms.seating.user-events` / `oms.attendance.remote`)

```java
// seating-service: cancel future bookings for deactivated user
@Bean
public Consumer<OmsEvent<UserDeactivatedPayload>> handleUserDeactivated() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        LocalDate endDate = event.payload().employmentEndDate();

        List<SeatBooking> futureBookings = seatBookingRepository
            .findConfirmedFromDate(event.payload().userId(), endDate);

        futureBookings.forEach(b -> {
            b.setStatus(BookingStatus.CANCELLED);
            b.setDeletedAt(Instant.now());
        });
        seatBookingRepository.saveAll(futureBookings);

        // Also update local user_summaries
        userSummaryRepository.markInactive(event.payload().userId());

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}
```

```java
// attendance-service: stop generating records after employment end date
@Bean
public Consumer<OmsEvent<UserDeactivatedPayload>> handleUserDeactivated() {
    return event -> {
        if (processedEventRepository.existsById(event.eventId())) return;

        deactivatedUserRepository.save(new DeactivatedUser(
            event.payload().userId(),
            event.payload().employmentEndDate()
        ));

        userSummaryRepository.markInactive(event.payload().userId());

        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    };
}
```

The nightly `BadgeSyncJob` checks `deactivated_users` before writing a new `AttendanceRecord`:

```java
if (!deactivatedUserRepository.isDeactivatedOnDate(userId, syncDate)) {
    attendanceRepository.upsertStatus(userId, syncDate, locationId, resolvedStatus);
}
```

---

### C4 — Supply request Saga: status propagates through events

No service queries `inventory-service`'s `supply_requests` table. The entire multi-step workflow is driven by events. `inventory-service` owns all state transitions; other services only react.

```
Employee submits → inventory-service: PENDING_MANAGER → publishes submitted event
notification-service: resolves manager from event payload → resolves email from local user_summaries → sends notification
Manager approves → inventory-service: PENDING_FACILITIES → publishes approved event
notification-service: resolves facilities admin from user_summaries → sends notification
Facilities Admin fulfils → inventory-service: FULFILLED; FIFO stock decrement → publishes fulfilled event
notification-service: resolves requester from user_summaries → sends notification
audit-service: persists all three audit records — full data in event payload, no cross-service query needed
```

---

## Pattern D — Parallel API Composition (Manager Dashboard)

The manager dashboard aggregates data from three independent services. No single service knows about the others. The aggregation happens at the API Gateway / BFF level.

```java
// Spring Cloud Gateway BFF route (or dedicated Gateway filter)
public ManagerDashboardResponse getDashboard(UUID managerId, UUID locationId) {

    // All three Feign calls fire concurrently
    CompletableFuture<TeamAttendanceSummary> attendance =
        CompletableFuture.supplyAsync(() ->
            attendanceClient.getTeamAttendance(managerId, locationId));

    CompletableFuture<TeamScheduleResponse> schedule =
        CompletableFuture.supplyAsync(() ->
            remoteClient.getTeamSchedule(managerId));

    CompletableFuture<List<SeatBookingSummary>> bookings =
        CompletableFuture.supplyAsync(() ->
            seatingClient.getTeamBookings(managerId, locationId));

    CompletableFuture.allOf(attendance, schedule, bookings).join();

    // Each getNow() uses a safe empty default if the circuit was open
    return ManagerDashboardResponse.builder()
        .attendance(attendance.getNow(TeamAttendanceSummary.empty()))
        .schedule(schedule.getNow(TeamScheduleResponse.empty()))
        .bookings(bookings.getNow(Collections.emptyList()))
        .build();
}
```

Combined latency ≈ slowest single call. If `seating-service` is down, the dashboard still shows attendance and schedule with `bookings: []`.

---

## Full Cross-Service Dependency Map

| Service needing data | Owns the data | Fields needed | Pattern | When |
|----------------------|--------------|---------------|---------|------|
| `attendance-service` | `auth-service` | personnelId → userId | A — local mapping table | Nightly badge sync |
| `attendance-service` | `auth-service` | direct reports list | B — Feign | Team attendance query |
| `attendance-service` | `auth-service` | display_name, email | A — user_summaries | Reports and exports |
| `attendance-service` | `remote-service` | approved dates, userId | C — RabbitMQ | Pass 2 attendance overlay |
| `seating-service` | `auth-service` | display_name | A — user_summaries | Floor plan occupants |
| `seating-service` | `attendance-service` | did user badge in? | C — RabbitMQ | No-show release |
| `seating-service` | `auth-service` | userId (deactivation) | C — RabbitMQ | Cancel future bookings |
| `remote-service` | `auth-service` | HR fallback approver userId | B — Feign | Approval routing |
| `remote-service` | `auth-service` | display_name, email | A — user_summaries | Team schedule display |
| `notification-service` | `auth-service` | email, display_name | A — user_summaries | Every notification dispatch |
| `audit-service` | `auth-service` | display_name, email | A — user_summaries | Audit log query enrichment |
| `inventory-service` | `auth-service` | display_name, email | A — user_summaries | Request history and reports |
| `workplace-service` | `auth-service` | host active + locationId | B — Feign | Pre-registration validation |
| `workplace-service` | `auth-service` | display_name | A — user_summaries | Visit history, attendee lists |
| `API Gateway (BFF)` | `attendance-service` + `remote-service` + `seating-service` | team dashboard data | D — parallel Feign | Manager dashboard |

---

## Rules That Enforce This

These rules prevent cross-service shortcuts:

1. **No service's DB user has credentials for any other service's database.** A cross-database JOIN is physically impossible.

2. **No direct REST calls to `notification-service` or `audit-service`.** These are RabbitMQ-only consumers. Calling them via REST is forbidden and enforced in code review.

3. **Every RabbitMQ consumer is idempotent.** All state updates use `ON CONFLICT DO UPDATE` or check `processedEventRepository` before acting.

4. **All Feign calls (Pattern B) have a circuit breaker + fallback.** If `auth-service` is down, the fallback is invoked — no 500 propagated to the user.

5. **Batch over N+1.** When enriching a list with user data, collect all IDs, make one Feign call, build a Map, enrich in-memory. Never call per row.

```java
// ❌ N+1 — one API call per record — FORBIDDEN
records.forEach(r -> r.setName(authServiceClient.getUser(r.getUserId()).getDisplayName()));

// ✅ Batch — one call for the entire result set
Set<UUID> ids = records.stream().map(Record::getUserId).collect(Collectors.toSet());
Map<UUID, String> nameMap = authServiceClient.getUsersByIds(ids)
    .stream()
    .collect(Collectors.toMap(UserSummaryResponse::getUserId, UserSummaryResponse::getDisplayName));
records.forEach(r -> r.setDisplayName(nameMap.getOrDefault(r.getUserId(), "Unknown")));
```

6. **Pattern A is preferred over Pattern B wherever eventual consistency (seconds) is acceptable.** Only use Pattern B when the data must be real-time at write time (login role resolution, host validation at visitor pre-registration).
