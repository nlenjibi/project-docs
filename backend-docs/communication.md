# Service Communication
## Office Management System (OMS)

Services communicate in two modes:
- **Synchronous REST** — user-facing queries, operations requiring immediate confirmation
- **Asynchronous Kafka / SQS** — state changes, cross-service workflows, audit logging, notifications

The key rule: `notification-service` and `audit-service` are **never called directly via REST**. All services publish events to Kafka or SQS and move on. These two services react independently.

---

## Communication Matrix

```
                    identity  user  attendance  seating  remote  notification  audit
identity-service       -       REST     -          -        -         -          ✦
user-service           -        -       ✦          -        -         ✦          ✦
attendance-service     -       REST     -          -       [K]        ✦          ✦
seating-service        -       REST    [K]          -        -         ✦          ✦
remote-service         -       REST    [K]          -        -         ✦          ✦
visitor-service        -       REST     -           -        -         ✦          ✦
event-service          -       REST     -           -        -         ✦          ✦
supplies-service       -       REST     -           -        -         ✦          ✦
assets-service         -       REST     -           -        -         ✦          ✦
notification-service   -        -       -           -        -         -           -
audit-service          -        -       -           -        -         -           -

REST  = synchronous REST call (with circuit breaker, retry, timeout, fallback)
[K]   = Kafka event consumed from
✦     = Kafka / SQS event produced (to notification bus and/or audit bus)
```

---

## Synchronous REST — Resilience Requirements

Every inter-service REST call **must** implement all four patterns. No inter-service call is made without them.

### Required Patterns

| Pattern | Config | Purpose |
|---------|--------|---------|
| Circuit Breaker | Opens after 50% failure rate over 10 calls | Stops cascading failures |
| Retry | Max 3 attempts, exponential backoff (100ms → 200ms → 400ms) | Handles transient errors |
| Timeout | 3 000ms hard ceiling | Prevents indefinite blocking |
| Fallback | Always defined — returns a safe degraded response | Ensures the calling service stays operational |

### Spring Configuration (application.yml — per downstream)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      user-service:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
        sliding-window-type: COUNT_BASED
  retry:
    instances:
      user-service:
        max-attempts: 3
        wait-duration: 100ms
        exponential-backoff-multiplier: 2
  timelimiter:
    instances:
      user-service:
        timeout-duration: 3s
```

### Java Implementation

```java
@CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
@Retry(name = "user-service")
@TimeLimiter(name = "user-service")
public CompletableFuture<UserResponse> getUser(UUID userId) {
    return CompletableFuture.supplyAsync(() ->
        userServiceClient.getUser(userId));
}

// Fallback: always return a safe degraded response — never propagate the exception
private CompletableFuture<UserResponse> getUserFallback(UUID userId, Exception ex) {
    log.warn("user-service unavailable for userId={}, using fallback", userId, ex);
    return CompletableFuture.completedFuture(UserResponse.unknown(userId));
}
```

### Failure Modes and Degraded Behaviour

| Failure | Behaviour |
|---------|-----------|
| `user-service` down | Return cached/default user profile; log warning; alert |
| `attendance-service` down | Attendance data unavailable; all other features unaffected |
| `identity-service` down | All new logins blocked; existing sessions continue until expiry |
| Synchronous chain > 2 hops | Use Kafka events for longer workflows — avoid sync chains |

---

## Asynchronous Kafka — Event Contracts

### Event Envelope

Every Kafka event uses this standard envelope. No event is published without it.

```java
public record OmsEvent<T>(
    UUID eventId,          // UUID v4 — unique per event; used for idempotency
    String eventType,      // e.g. "seating.booking.created"
    String version,        // "1" — increment for breaking schema changes
    UUID correlationId,    // Propagated from the originating HTTP request's X-Correlation-ID
    Instant occurredAt,
    UUID locationId,
    T payload
) {}
```

JSON wire format:
```json
{
  "eventId": "uuid-v4",
  "eventType": "seating.booking.created",
  "version": "1",
  "correlationId": "uuid-v4",
  "occurredAt": "2026-03-01T09:00:00Z",
  "locationId": "location-uuid",
  "payload": { ... }
}
```

### Topic Naming Convention

```
oms.{domain}.{entity}.{action}

Examples:
  oms.remote.request.approved
  oms.seating.booking.released
  oms.attendance.record.resolved
  oms.audit.event
  oms.user.user.deactivated
```

### Consumer Group Naming

```
oms-{consuming-service}-{domain}

Examples:
  oms-attendance-remote       → attendance-service consuming remote events
  oms-notification-seating    → notification-service consuming seating events
  oms-audit-all               → audit-service consuming all audit events
```

---

## Kafka Topic Registry

| Topic | Producer | Consumers |
|-------|---------|-----------|
| `oms.user.user.created` | user-service | notification-service, audit-service |
| `oms.user.user.updated` | user-service | audit-service |
| `oms.user.user.deactivated` | user-service | attendance-service, seating-service, audit-service |
| `oms.user.sync.completed` | user-service | audit-service |
| `oms.attendance.record.resolved` | attendance-service | audit-service |
| `oms.attendance.no_show.detected` | attendance-service | seating-service |
| `oms.seating.booking.created` | seating-service | notification-service, audit-service |
| `oms.seating.booking.cancelled` | seating-service | notification-service, audit-service |
| `oms.seating.booking.released` | seating-service | notification-service, audit-service |
| `oms.remote.request.submitted` | remote-service | notification-service, audit-service |
| `oms.remote.request.approved` | remote-service | attendance-service, notification-service, audit-service |
| `oms.remote.request.rejected` | remote-service | notification-service, audit-service |
| `oms.ooo.request.approved` | remote-service | attendance-service, notification-service, audit-service |
| `oms.visitor.visit.checked_in` | visitor-service | notification-service, audit-service |
| `oms.visitor.visit.checked_out` | visitor-service | audit-service |
| `oms.event.event.created` | event-service | audit-service |
| `oms.event.rsvp.confirmed` | event-service | notification-service, audit-service |
| `oms.event.rsvp.waitlisted` | event-service | notification-service, audit-service |
| `oms.event.rsvp.promoted` | event-service | notification-service, audit-service |
| `oms.event.event.cancelled` | event-service | notification-service, audit-service |
| `oms.event.event.reminder` | event-service | notification-service |
| `oms.supplies.request.submitted` | supplies-service | notification-service, audit-service |
| `oms.supplies.request.approved` | supplies-service | notification-service, audit-service |
| `oms.supplies.request.rejected` | supplies-service | notification-service, audit-service |
| `oms.supplies.request.fulfilled` | supplies-service | notification-service, audit-service |
| `oms.supplies.stock.low` | supplies-service | notification-service, audit-service |
| `oms.supplies.reorder.triggered` | supplies-service | procurement adapter, audit-service |
| `oms.assets.asset.assigned` | assets-service | notification-service, audit-service |
| `oms.assets.asset.acknowledged` | assets-service | audit-service |
| `oms.assets.asset.returned` | assets-service | audit-service |
| `oms.assets.fault_reported` | assets-service | notification-service, audit-service |
| `oms.assets.maintenance_logged` | assets-service | notification-service, audit-service |
| `oms.assets.asset.retired` | assets-service | audit-service |
| `oms.audit.event` | all services | audit-service |

---

## Idempotent Kafka Consumers (Mandatory)

Kafka guarantees **at-least-once delivery** — every consumer must handle duplicate events without side effects.

### Implementation Pattern

```java
@KafkaListener(topics = "oms.remote.request.approved", groupId = "oms-attendance-remote")
@Transactional
public void handleRemoteApproved(OmsEvent<RemoteApprovedPayload> event) {
    // Guard: skip if already processed
    if (processedEventRepository.existsById(event.eventId())) {
        log.info("Duplicate event ignored: {}", event.eventId());
        return;
    }

    // Business logic
    attendanceService.overlayRemoteStatus(
        event.payload().userId(),
        event.payload().dates(),
        event.locationId()
    );

    // Mark as processed within the same transaction
    processedEventRepository.save(new ProcessedEvent(event.eventId()));
}
```

Every service maintains a `processed_events` table with the `eventId` as the primary key.

### Kafka Consumer Rules

- Always commit offsets **after** successful processing — not before.
- Failed processing must **not** commit the offset — let Kafka redeliver.
- Consumer groups are isolated per consuming service and domain — no shared group IDs.
- Breaking schema changes require a new event type (e.g., `remote.request.approved.v2`). The old type continues to be published until all consumers migrate.

---

## Publishing Events (Service Pattern)

Events are published **after** the database transaction commits. Never publish before committing.

```java
@Service
@RequiredArgsConstructor
public class SeatBookingService {
    private final SeatBookingRepository repository;
    private final KafkaEventPublisher publisher;
    private final AuditEventPublisher auditPublisher;

    @Transactional
    public SeatBookingResponse book(SeatBookingRequest request) {
        // 1. Validate and persist
        SeatBooking booking = repository.save(mapToEntity(request));

        // 2. Publish domain event (post-commit via TransactionalEventListener)
        publisher.publish("oms.seating.booking.created",
            SeatBookingCreatedEvent.from(booking));

        // 3. Publish audit event
        auditPublisher.publish(AuditEvent.of(
            "SEAT_BOOKED", "SeatBooking", booking.getId(),
            null, booking, currentUser));

        return mapToResponse(booking);
    }
}
```

---

## SQS — Audit Event Dispatch

The `audit-service` and some alert paths use AWS SQS in addition to Kafka.

### SQS Queues

| Queue | Producer | Consumer | Purpose |
|-------|---------|---------|---------|
| `oms-audit-events` | All services via `AuditLogService` | audit-service `@SqsListener` | Async audit record persistence |
| `oms-notifications-queue` | attendance-service (failure) | notification-service | Badge sync failure alert to Super Admin |
| `oms-attendance-sync-status` | attendance-service | seating-service, notification-service | Badge sync completion signal |

### SQS Message Format (Audit)

```json
{
  "eventId": "uuid-v4",
  "actorId": "uuid",
  "actorRole": "MANAGER",
  "action": "SEAT_BOOKED",
  "entityType": "SeatBooking",
  "entityId": "uuid",
  "locationId": "uuid",
  "previousState": null,
  "newState": { ... },
  "occurredAt": "2026-03-01T09:00:00Z"
}
```

SQS consumers are idempotent — the `eventId` is stored in a `processed_audit_events` table to guard against duplicate delivery.

---

## Choreography-Based Saga — Supply Request Fulfilment

For multi-step workflows, the **Choreography Saga** pattern is used. No central orchestrator — each service reacts to events autonomously.

```
Employee submits supply request
  ↓ supplies-service: status = PENDING_MANAGER
  ↓ publishes: oms.supplies.request.submitted
        ↓
  notification-service: notifies Manager (in-app + email)
  audit-service: persists audit event
        ↓
Manager approves
  ↓ supplies-service: status = PENDING_FACILITIES
  ↓ publishes: oms.supplies.request.approved (stage 1)
        ↓
  notification-service: notifies Facilities Admin
        ↓
Facilities Admin fulfils
  ↓ supplies-service: status = FULFILLED; decrements stock (FIFO)
  ↓ publishes: oms.supplies.request.fulfilled
        ↓
  notification-service: notifies Employee
  audit-service: persists full trail

Compensation (Manager rejects):
  ↓ supplies-service: status = REJECTED
  ↓ publishes: oms.supplies.request.rejected
        ↓
  notification-service: notifies Employee with reason
```

Each step is local to its service. There is no distributed transaction coordinator.

---

## API Composition — Manager Dashboard

Where a query requires data from multiple services, the API Gateway (or a BFF layer) composes responses with parallel REST calls. Each call has its own circuit breaker and fallback — if one service is degraded, the others still return data.

```
GET /api/v1/dashboard/manager/{id}
  ↓ (parallel)
  ├── attendance-service: GET /api/v1/attendance/team/{id}
  ├── remote-service:     GET /api/v1/teams/{id}/schedule
  └── seating-service:    GET /api/v1/seat-bookings/team/{id}
  ↓
Composed response to Angular client
```
