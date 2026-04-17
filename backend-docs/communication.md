# Service Communication
## Office Management System (OMS)

**Version:** 3.0 — RabbitMQ (Amazon MQ) + Spring Cloud OpenFeign

Services communicate in two modes:
- **Synchronous REST via Feign** — user-facing queries and operations that require an immediate, confirmed response
- **Asynchronous RabbitMQ events** — state changes, multi-service workflows, audit logging, notifications

**The unbreakable rule:** `notification-service` and `audit-service` are **never called directly via REST**. All services publish events to RabbitMQ and move on. These two services react independently. This ensures that a notification outage never blocks a booking confirmation, and an audit-service restart never blocks an attendance override.

---

## Why RabbitMQ over Apache Kafka or SQS

### vs. Apache Kafka (MSK)

Kafka's primary advantage is durable, replayable event log storage. OMS has no requirement to replay events — attendance records are computed from badge data in Athena, not re-computed from Kafka topics. The OMS event volume (hundreds of events/day, not millions) does not justify Kafka's partition management overhead or its ~$200–350/month AWS MSK cost.

**Trade-off accepted:** RabbitMQ does not support event replay. If a consumer was down and missed events, those messages are in the DLQ rather than re-readable from a topic. Mitigation: robust DLQ processing and alerting ensure no messages are silently lost.

### vs. AWS SQS

SQS has no official Spring Cloud Stream binder. Implementing fan-out (one event → multiple consumers) would require SNS + multiple SQS queues, adding significant configuration complexity. RabbitMQ's Topic Exchange achieves the same fan-out pattern natively.

**Annual cost saving vs MSK:** ~$2,040–3,840/year.

---

## Communication Matrix

```
                    auth  attendance  seating  remote  notification  audit  inventory  workplace
auth-service          -       -          -        -          -          ✦        -          -
attendance-service   REST      -         [R]       -          ✦          ✦        -          -
seating-service      REST    [R]          -        -          ✦          ✦        -          -
remote-service       REST     -           -        -          ✦          ✦        -          -
inventory-service    REST     -           -        -          ✦          ✦        -          -
workplace-service    REST     -           -        -          ✦          ✦        -          -
notification-service  -       -           -        -          -          -         -          -
audit-service         -       -           -        -          -          -         -          -

REST  = synchronous Feign call (circuit breaker + retry + timeout + fallback)
[R]   = RabbitMQ event consumed from
✦     = RabbitMQ event produced (to notification and/or audit routing keys)
```

---

## Synchronous REST — Feign Clients with Resilience4j

Every inter-service REST call uses **Spring Cloud OpenFeign** with **Spring Cloud LoadBalancer** (resolves `lb://service-name` via Eureka) and **Resilience4j** for fault tolerance.

### Why Feign over RestTemplate or WebClient

Feign generates the HTTP client from a Java interface at compile time. The interface is the contract — no URL construction, no serialisation boilerplate. Combined with `@CircuitBreaker` and `@Retry`, a five-line interface replaces 40 lines of WebClient code. It also integrates directly with Micrometer for automatic latency metrics on every call.

### Required Resilience Patterns (All Feign Calls)

| Pattern | Configuration | Why It Matters |
|---------|--------------|----------------|
| **Circuit Breaker** | Opens after 50% failure rate over 10 calls; waits 10s before half-open retry | Stops cascading failures across services |
| **Retry** | Max 3 attempts; exponential backoff 100ms → 200ms → 400ms | Handles transient network hiccups and pod restarts |
| **Timeout** | 3,000ms hard ceiling | Prevents thread pool exhaustion from slow downstream |
| **Fallback** | Defined on every call — returns a safe degraded response | Ensures the calling service stays operational |

**No Feign call is made without all four patterns. No exceptions.**

### Resilience4j Configuration (application.yml — one instance per downstream)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      auth-service:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 2s
        wait-duration-in-open-state: 10s
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        permitted-number-of-calls-in-half-open-state: 3
        register-health-indicator: true

  retry:
    instances:
      auth-service:
        max-attempts: 3
        wait-duration: 100ms
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - feign.RetryableException
          - java.net.ConnectException
        ignore-exceptions:
          - com.oms.exception.ForbiddenException
          - com.oms.exception.NotFoundException

  timelimiter:
    instances:
      auth-service:
        timeout-duration: 3s
        cancel-running-future: true
```

**One named instance per downstream service.** Each service names instances for every service it calls: `auth-service`, `attendance-service`, etc.

### Feign Client Interface Pattern

```java
// seating-service calling auth-service
@FeignClient(name = "auth-service", path = "/api/v1")
public interface AuthServiceClient {

    @GetMapping("/users/{id}/direct-reports")
    List<UserSummaryResponse> getDirectReports(
        @PathVariable UUID id,
        @RequestParam UUID locationId);

    @GetMapping("/locations/{locationId}/role-holder")
    UserSummaryResponse getRoleHolder(
        @PathVariable UUID locationId,
        @RequestParam String role);
}
```

### Service Method with Resilience4j

```java
@Service
@RequiredArgsConstructor
public class TeamAttendanceService {
    private final AuthServiceClient authServiceClient;

    @CircuitBreaker(name = "auth-service", fallbackMethod = "getDirectReportsFallback")
    @Retry(name = "auth-service")
    @TimeLimiter(name = "auth-service")
    public CompletableFuture<List<UserSummaryResponse>> getDirectReports(
            UUID managerId, UUID locationId) {
        return CompletableFuture.supplyAsync(() ->
            authServiceClient.getDirectReports(managerId, locationId));
    }

    // Fallback: return empty list so the attendance query returns an empty result
    // rather than failing the entire request
    private CompletableFuture<List<UserSummaryResponse>> getDirectReportsFallback(
            UUID managerId, UUID locationId, Exception ex) {
        log.warn("auth-service unavailable for managerId={}, locationId={}. Returning empty team list.",
            managerId, locationId, ex);
        return CompletableFuture.completedFuture(Collections.emptyList());
    }
}
```

### Failure Modes and Degraded Behaviour

| Failure | Degraded Behaviour |
|---------|-------------------|
| `auth-service` down | New logins blocked; existing sessions continue until expiry; Feign callers receive fallback responses |
| `auth-service` down (role resolution) | Login fails with `503 SERVICE_UNAVAILABLE` — issuing a session without roles is a security violation |
| `attendance-service` down | Attendance data unavailable; all other features unaffected |
| `seating-service` down | Bookings unavailable; attendance, remote, auth unaffected |
| Synchronous chain > 2 hops | Use RabbitMQ events for longer workflows — avoid sync chains beyond A → B → C |

---

## Asynchronous RabbitMQ — Event Architecture

### Why Topic Exchange

OMS uses a single **Topic Exchange** (`oms.events`) with routing keys. This allows fine-grained subscription:

- `notification-service` binds to `oms.remote.request.*`, `oms.seating.booking.*`, etc.
- `audit-service` binds to `oms.audit.#` (all audit events) AND to every other routing key (for full audit coverage)
- A new consumer can subscribe to specific events without the producer knowing about it

**Trade-off vs. multiple direct exchanges:** A single Topic Exchange with routing key patterns is easier to reason about and maintain than 30 direct exchanges. The small routing overhead is negligible at OMS event volumes.

### Exchange and Queue Topology

```
Exchange: oms.events (topic, durable: true)

Queues (durable: true, auto-delete: false):
  oms.notification           → notification-service consumer
  oms.audit                  → audit-service consumer
  oms.seating.user-events    → seating-service (user deactivation + no-show detection)
  oms.attendance.remote      → attendance-service (remote/OOO approvals + user deactivation)

Dead-letter Exchange: oms.dlx (direct, durable: true)
Dead-letter Queues:
  oms.notification.dlq
  oms.audit.dlq
  oms.seating.user-events.dlq
  oms.attendance.remote.dlq
```

**Every queue has a DLQ.** Messages that fail processing 3 times are routed to the DLQ. An alert fires when DLQ depth > 0. Operations investigates root cause. DLQ messages are replayed after the fix is deployed.

### Routing Key Registry

| Routing Key | Producer | Consumers |
|-------------|---------|-----------|
| `oms.user.created` | auth-service | oms.notification, oms.audit |
| `oms.user.updated` | auth-service | oms.audit |
| `oms.user.deactivated` | auth-service | oms.attendance.remote, oms.seating.user-events, oms.audit |
| `oms.user.sync.completed` | auth-service | oms.audit |
| `oms.attendance.record.resolved` | attendance-service | oms.audit |
| `oms.attendance.no_show.detected` | attendance-service | oms.seating.user-events |
| `oms.seating.booking.created` | seating-service | oms.notification, oms.audit |
| `oms.seating.booking.cancelled` | seating-service | oms.notification, oms.audit |
| `oms.seating.booking.released` | seating-service | oms.notification, oms.audit |
| `oms.remote.request.submitted` | remote-service | oms.notification, oms.audit |
| `oms.remote.request.approved` | remote-service | oms.attendance.remote, oms.notification, oms.audit |
| `oms.remote.request.rejected` | remote-service | oms.notification, oms.audit |
| `oms.ooo.request.approved` | remote-service | oms.attendance.remote, oms.notification, oms.audit |
| `oms.inventory.supply.request.submitted` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.supply.request.approved` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.supply.request.rejected` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.supply.request.fulfilled` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.stock.low` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.reorder.triggered` | inventory-service | oms.audit |
| `oms.inventory.asset.assigned` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.asset.acknowledged` | inventory-service | oms.audit |
| `oms.inventory.asset.returned` | inventory-service | oms.audit |
| `oms.inventory.fault.reported` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.maintenance.logged` | inventory-service | oms.notification, oms.audit |
| `oms.inventory.asset.retired` | inventory-service | oms.audit |
| `oms.workplace.visit.checked_in` | workplace-service | oms.notification, oms.audit |
| `oms.workplace.visit.checked_out` | workplace-service | oms.audit |
| `oms.workplace.event.created` | workplace-service | oms.audit |
| `oms.workplace.event.rsvp.confirmed` | workplace-service | oms.notification, oms.audit |
| `oms.workplace.event.rsvp.waitlisted` | workplace-service | oms.notification, oms.audit |
| `oms.workplace.event.rsvp.promoted` | workplace-service | oms.notification, oms.audit |
| `oms.workplace.event.cancelled` | workplace-service | oms.notification, oms.audit |
| `oms.workplace.event.reminder` | workplace-service | oms.notification, oms.audit |
| `oms.audit.event` | All services | oms.audit |

---

### Event Envelope

Every RabbitMQ message uses a standard envelope. No message is published without it.

```java
public record OmsEvent<T>(
    UUID   eventId,       // UUID v4 — unique per event; used for idempotency deduplication
    String eventType,     // Matches routing key: e.g. "oms.seating.booking.created"
    String version,       // Schema version: "1". Increment on breaking payload changes.
    UUID   correlationId, // Propagated from the originating HTTP request's X-Correlation-ID header
    Instant occurredAt,   // Event timestamp — always UTC
    UUID   locationId,    // Location context for all location-scoped events
    T      payload        // Event-specific payload
) {}
```

JSON wire format:
```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440001",
  "eventType": "oms.seating.booking.created",
  "version": "1",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "occurredAt": "2026-01-18T14:30:00Z",
  "locationId": "accra-uuid",
  "payload": {
    "bookingId": "uuid",
    "userId": "uuid",
    "seatId": "uuid",
    "bookingDate": "2026-01-20"
  }
}
```

---

### Spring Cloud Stream Configuration (RabbitMQ Binder)

```yaml
spring:
  cloud:
    stream:
      binder:
        type: rabbit
      rabbit:
        bindings:
          # Producer
          publishBookingCreated-out-0:
            producer:
              exchange-name: oms.events
              routing-key-expression: '''oms.seating.booking.created'''
          # Consumer
          handleUserDeactivated-in-0:
            consumer:
              binding-routing-key: oms.user.deactivated
              dead-letter-exchange: oms.dlx
              dead-letter-routing-key: oms.seating.user-events.dlq
              max-attempts: 3
              back-off-initial-interval: 1000
              back-off-max-interval: 10000
              back-off-multiplier: 2.0
      bindings:
        publishBookingCreated-out-0:
          destination: oms.events
        handleUserDeactivated-in-0:
          destination: oms.events
          group: oms-seating-user-events    # Creates a durable named queue

  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: 5672
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}
    virtual-host: /oms
    ssl:
      enabled: true    # Always true in staging and production (Amazon MQ TLS)
```

---

### Publishing Events (Java — StreamBridge)

Events are published **after** the database transaction commits, using `@TransactionalEventListener(phase = AFTER_COMMIT)`. Never publish inside a transaction — if the publish fails, the transaction would roll back but the side effect (event) cannot be rolled back.

```java
@Service
@RequiredArgsConstructor
public class SeatBookingService {
    private final SeatBookingRepository repository;
    private final ApplicationEventPublisher applicationEventPublisher;

    @Transactional
    public SeatBookingResponse book(SeatBookingRequest request, UUID actorId) {
        // 1. Validate seat availability
        if (!seatRepository.isAvailable(request.getSeatId(), request.getBookingDate())) {
            throw new SeatUnavailableException();
        }

        // 2. Persist
        SeatBooking booking = repository.save(mapToEntity(request, actorId));

        // 3. Publish domain event — deferred until AFTER commit via @TransactionalEventListener
        applicationEventPublisher.publishEvent(
            new SeatBookingCreatedDomainEvent(booking));

        return mapToResponse(booking);
    }
}

// Domain event holder
public record SeatBookingCreatedDomainEvent(SeatBooking booking) {}

// Publisher — fires AFTER_COMMIT
@Component
@RequiredArgsConstructor
public class SeatBookingEventPublisher {
    private final StreamBridge streamBridge;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onBookingCreated(SeatBookingCreatedDomainEvent domainEvent) {
        SeatBooking b = domainEvent.booking();

        OmsEvent<SeatBookingCreatedPayload> event = new OmsEvent<>(
            UUID.randomUUID(),
            "oms.seating.booking.created",
            "1",
            CorrelationIdHolder.get(),  // ThreadLocal set by gateway filter
            Instant.now(),
            b.getLocationId(),
            new SeatBookingCreatedPayload(b.getId(), b.getUserId(), b.getSeatId(), b.getBookingDate())
        );

        streamBridge.send("publishBookingCreated-out-0", event);
    }
}
```

---

### Consuming Events (Java — Functional Consumer)

Spring Cloud Stream functional consumers — no `@KafkaListener` annotation; wired by bean name convention (`handleXxx-in-0`).

```java
// seating-service: handle user deactivation — cancel all future bookings
@Bean
public Consumer<OmsEvent<UserDeactivatedPayload>> handleUserDeactivated() {
    return event -> {
        // Idempotency guard — check processed_events table first
        if (processedEventRepository.existsById(event.eventId())) {
            log.info("Duplicate event ignored: eventId={}", event.eventId());
            return;
        }

        try {
            seatBookingService.cancelFutureBookingsForUser(
                event.payload().userId(),
                event.payload().employmentEndDate(),
                event.locationId()
            );

            // Mark processed WITHIN the same business transaction
            processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));

        } catch (Exception e) {
            log.error("Failed to process user deactivation event: eventId={}", event.eventId(), e);
            // Re-throw to trigger RabbitMQ retry → DLQ after max-attempts
            throw e;
        }
    };
}
```

### Idempotency — Mandatory on All Consumers

RabbitMQ guarantees **at-least-once delivery** — a message CAN be delivered more than once (consumer restart, acknowledgement timeout). Every consumer MUST be idempotent.

```java
// processed_events table — present in every service's database
CREATE TABLE processed_events (
    event_id     UUID PRIMARY KEY,     -- OmsEvent.eventId — deduplication key
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Consumer rules:**
1. Check `processedEventRepository.existsById(event.eventId())` before ANY business logic.
2. If already processed → log at INFO level, return immediately (do not re-throw).
3. If not processed → execute business logic + insert `ProcessedEvent` in the SAME transaction.
4. On exception → re-throw → RabbitMQ retries → after `max-attempts` → routes to DLQ.
5. NEVER manually ACK before business logic succeeds. Spring Cloud Stream handles ack/nack automatically.

---

## Choreography-Based Saga — Supply Request Fulfilment

Multi-step workflows use the **Choreography Saga** pattern. No central orchestrator. Each service reacts to events and publishes the next event in the chain. Each step is a local transaction.

```
Employee submits supply request via inventory-service
  ↓ inventory-service: status = PENDING_MANAGER
  ↓ publishes: oms.inventory.supply.request.submitted
        ↓
  notification-service: notifies Manager (in-app + email)
  audit-service: persists audit event
        ↓
Manager calls PATCH /api/v1/supply-requests/{id}/approve-stage1
  ↓ inventory-service: status = PENDING_FACILITIES
  ↓ publishes: oms.inventory.supply.request.approved { stage: 1 }
        ↓
  notification-service: notifies Facilities Admin
        ↓
Facilities Admin calls PATCH /api/v1/supply-requests/{id}/fulfil
  ↓ inventory-service: status = FULFILLED; decrements FIFO stock
  ↓ if stock < threshold: publishes oms.inventory.stock.low
  ↓ publishes: oms.inventory.supply.request.fulfilled
        ↓
  notification-service: notifies Employee
  audit-service: persists full trail

Compensation path (Manager rejects):
  ↓ inventory-service: status = REJECTED
  ↓ publishes: oms.inventory.supply.request.rejected { reason }
        ↓
  notification-service: notifies Employee with rejection reason
  audit-service: persists rejection event
```

**Why choreography over orchestration:** An orchestrator (Saga orchestrator service) would be a single point of failure and a coordination bottleneck. With choreography, `inventory-service` is the only stateful actor — it owns the request lifecycle. `notification-service` and `audit-service` react independently. The workflow can survive temporary downtime of any participant because RabbitMQ queues absorb the events until the consumer recovers.

---

## API Composition — Manager Dashboard

Where a frontend query requires data from multiple services, Spring Cloud Gateway (or a BFF route) composes responses with parallel Feign calls. Each call is independent — one failure does not block the others.

```java
// In Spring Cloud Gateway BFF route or a dedicated dashboard endpoint
public ManagerDashboardResponse getDashboard(UUID managerId, UUID locationId) {

    CompletableFuture<TeamAttendanceSummary> attendance =
        CompletableFuture.supplyAsync(() ->
            attendanceClient.getTeamAttendance(managerId, locationId));

    CompletableFuture<TeamScheduleResponse> schedule =
        CompletableFuture.supplyAsync(() ->
            remoteClient.getTeamSchedule(managerId));

    CompletableFuture<List<SeatBookingSummary>> bookings =
        CompletableFuture.supplyAsync(() ->
            seatingClient.getTeamBookings(managerId, locationId));

    // Combined latency ≈ slowest single call (all three run in parallel)
    CompletableFuture.allOf(attendance, schedule, bookings).join();

    return ManagerDashboardResponse.builder()
        .attendance(attendance.getNow(TeamAttendanceSummary.empty()))
        .schedule(schedule.getNow(TeamScheduleResponse.empty()))
        .bookings(bookings.getNow(Collections.emptyList()))
        .build();
}
```

Each `CompletableFuture` has its own circuit breaker. If `seating-service` is open, the dashboard returns attendance and schedule with `bookings: []` (graceful degradation) rather than a 503.

---

## Correlation ID Propagation

Every request carries a `correlationId` across the entire call chain — HTTP, Feign, and RabbitMQ events.

```
React SPA → Spring Cloud Gateway → seating-service → auth-service (Feign)
                  generates                  propagates        propagates
              X-Correlation-ID: abc       X-Correlation-ID: abc  X-Correlation-ID: abc
                                               ↓
                                   RabbitMQ message header
                                   "X-Correlation-ID": "abc"
                                               ↓
                                         audit-service [corr: abc]
                                         notification-service [corr: abc]
```

**Gateway filter (generates on inbound):**
```java
// GlobalFilter in Spring Cloud Gateway
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    String correlationId = exchange.getRequest().getHeaders()
        .getFirst("X-Correlation-ID");
    if (correlationId == null) {
        correlationId = UUID.randomUUID().toString();
    }
    exchange.getRequest().mutate()
        .header("X-Correlation-ID", correlationId)
        .build();
    return chain.filter(exchange);
}
```

**Feign interceptor (propagates on all outbound Feign calls):**
```java
@Bean
public RequestInterceptor correlationIdInterceptor() {
    return requestTemplate -> {
        String correlationId = CorrelationIdHolder.get();
        if (correlationId != null) {
            requestTemplate.header("X-Correlation-ID", correlationId);
        }
    };
}
```

**RabbitMQ publisher (includes in message headers):**
```java
// StreamBridge automatically includes message headers when using MessageBuilder
Message<OmsEvent<?>> message = MessageBuilder
    .withPayload(event)
    .setHeader("X-Correlation-ID", event.correlationId().toString())
    .setHeader("X-Event-Type", event.eventType())
    .build();
streamBridge.send("publishBookingCreated-out-0", message);
```

A single `correlationId` can trace the complete path: HTTP request → Feign call → RabbitMQ event → audit record — across all services and across Jaeger distributed traces.
