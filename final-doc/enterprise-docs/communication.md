# Service Communication
## Office Management System (OMS)

**Version:** 3.0 | Sync: Spring Cloud OpenFeign | Async: RabbitMQ via Spring Cloud Stream

---

## 1. Communication Principles

| Rule | Enforcement |
|---|---|
| Use REST for real-time user-facing queries | Gateway routes all external traffic |
| Use RabbitMQ for state changes and cross-service workflows | Audit and notification services consume events only |
| Never call notification or audit services directly via REST | No REST client to these services exists in any codebase |
| Every consumer is idempotent | `processed_events` deduplication table in every consumer |
| Consumers commit offset only after successful processing | `acknowledgeMode: MANUAL` equivalent via Spring AMQP ack |
| All events carry a correlationId | Propagated via message headers and HTTP headers |

---

## 2. Synchronous Communication (Spring Cloud OpenFeign)

Used for: user-facing queries requiring an immediate response.

### Feign Client Declaration

```java
// seating-service calling auth-service
@FeignClient(name = "auth-service", fallback = AuthServiceFallback.class)
public interface AuthServiceClient {

    @GetMapping("/api/v1/users/{id}")
    UserResponse getUser(@PathVariable UUID id);

    @GetMapping("/api/v1/locations/{id}/config")
    LocationConfigResponse getLocationConfig(@PathVariable UUID id);
}

@Component
public class AuthServiceFallback implements AuthServiceClient {

    @Override
    public UserResponse getUser(UUID id) {
        log.warn("auth-service unavailable, returning default for user {}", id);
        return UserResponse.defaultResponse(id);
    }

    @Override
    public LocationConfigResponse getLocationConfig(UUID id) {
        return LocationConfigResponse.defaultConfig(id);
    }
}
```

### Resilience4j Configuration (every service)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      auth-service:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 2s
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      auth-service:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - feign.RetryableException
          - java.net.ConnectException
  timelimiter:
    instances:
      auth-service:
        timeout-duration: 3s
        cancel-running-future: true

feign:
  circuitbreaker:
    enabled: true
  client:
    config:
      default:
        connectTimeout: 2000
        readTimeout: 3000
        loggerLevel: BASIC
```

### Inter-Service REST Call Matrix

```
                auth  attend  seating  remote  notif  audit
auth-service      -     -        -        -      -      -
attendance      REST    -        -       [MQ]    -      -
seating         REST   [MQ]      -        -      -      -
remote          REST    -        -        -      -      -
notification      -     -        -        -      -      -
audit             -     -        -        -      -      -

REST = synchronous Feign call
[MQ] = consumes RabbitMQ event from that service
```

---

## 3. Asynchronous Communication (RabbitMQ)

### Exchange and Queue Architecture

OMS uses one **Topic Exchange** per domain. Each consumer service has its own **durable queue** bound to the exchange with the relevant routing keys.

```
Exchange: oms.remote (type: topic)

remote-service publishes:
  routing key: "request.approved"
  payload: { eventId, requestId, userId, dates, locationId, occurredAt }

Bound queues:
  oms.remote.attendance-service    ← attendance-service (overlay REMOTE status)
  oms.remote.notification-service  ← notification-service (notify employee)
  oms.remote.audit-service         ← audit-service (persist immutably)

Dead letter queues (auto-created):
  oms.dlx.remote.attendance-service.failed
  oms.dlx.remote.notification-service.failed
  oms.dlx.remote.audit-service.failed
```

### Full Exchange Registry

| Exchange | Routing Keys Published | Producer | Consumer Services |
|---|---|---|---|
| `oms.user` | `user.created`, `user.updated`, `user.deactivated` | auth-service | notification, audit |
| `oms.attendance` | `record.resolved`, `no_show.detected` | attendance-service | seating (no_show only), audit |
| `oms.seating` | `booking.created`, `booking.cancelled`, `booking.released` | seating-service | notification, audit |
| `oms.remote` | `request.submitted`, `request.approved`, `request.rejected`, `ooo.request.approved` | remote-service | attendance (approved only), notification, audit |
| `oms.visitor` | `visit.checked_in`, `visit.checked_out` | workplace-service | notification, audit |
| `oms.event` | `rsvp.confirmed`, `rsvp.promoted`, `event.cancelled`, `event.reminder` | workplace-service | notification, audit |
| `oms.inventory` | `request.fulfilled`, `asset.assigned`, `stock.low`, `reorder.triggered` | inventory-service | notification, audit |
| `oms.audit` | `#` (all routing keys) | all services | audit-service |

---

### Spring Cloud Stream Configuration (per service)

**Producer (remote-service example):**

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: 5671
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}
    ssl.enabled: true

  cloud:
    stream:
      bindings:
        # Producers
        remote-request-approved-out-0:
          destination: oms.remote
          content-type: application/json
        remote-request-rejected-out-0:
          destination: oms.remote
          content-type: application/json

      rabbit:
        bindings:
          remote-request-approved-out-0:
            producer:
              exchange-type: topic
              routing-key-expression: "'request.approved'"
              durable-subscription: true
          remote-request-rejected-out-0:
            producer:
              exchange-type: topic
              routing-key-expression: "'request.rejected'"
              durable-subscription: true
```

**Consumer (attendance-service consuming from remote):**

```yaml
spring:
  cloud:
    stream:
      bindings:
        remoteRequestApproved-in-0:
          destination: oms.remote
          group: attendance-service           # queue = oms.remote.attendance-service
          content-type: application/json

      rabbit:
        bindings:
          remoteRequestApproved-in-0:
            consumer:
              exchange-type: topic
              binding-routing-key: request.approved
              durable-subscription: true
              auto-bind-dlq: true             # auto-creates DLQ
              dlq-ttl: 604800000              # 7 days in DLQ
              requeue-rejected: false         # failed → DLQ, not requeued
              max-attempts: 3
              back-off-initial-interval: 1000
              back-off-max-interval: 10000
              back-off-multiplier: 2.0
              acknowledge-mode: AUTO
```

---

### Producer Code Pattern

```java
@Service
@RequiredArgsConstructor
public class RemoteEventPublisher {

    private final StreamBridge streamBridge;
    private final ObjectMapper objectMapper;

    public void publishRequestApproved(RemoteRequest request) {
        var event = EventEnvelope.<RemoteApprovedPayload>builder()
            .eventId(UUID.randomUUID())
            .eventType("remote.request.approved")
            .version("1")
            .correlationId(MDC.get("correlationId"))
            .occurredAt(Instant.now())
            .locationId(request.getLocationId())
            .payload(RemoteApprovedPayload.from(request))
            .build();

        streamBridge.send("remote-request-approved-out-0", event);
        log.info("Published remote.request.approved for requestId={}", request.getId());
    }
}
```

### Consumer Code Pattern

```java
@Configuration
@RequiredArgsConstructor
public class AttendanceConsumers {

    private final AttendanceOverlayService overlayService;
    private final ProcessedEventRepository processedEventRepo;

    @Bean
    public Consumer<EventEnvelope<RemoteApprovedPayload>> remoteRequestApproved() {
        return event -> {
            // Idempotency check
            if (processedEventRepo.existsById(event.getEventId())) {
                log.info("Duplicate event skipped: eventId={}", event.getEventId());
                return;
            }

            MDC.put("correlationId", event.getCorrelationId().toString());

            try {
                overlayService.overlayRemoteStatus(
                    event.getPayload().getUserId(),
                    event.getPayload().getDates(),
                    event.getPayload().getLocationId()
                );
                processedEventRepo.save(new ProcessedEvent(event.getEventId(), Instant.now()));
                log.info("Processed remote.request.approved: eventId={}", event.getEventId());
            } catch (Exception e) {
                log.error("Failed to process remote.request.approved: eventId={}", event.getEventId(), e);
                throw e; // Spring Cloud Stream will retry, then send to DLQ
            } finally {
                MDC.clear();
            }
        };
    }
}
```

---

### Universal Event Envelope

All events use this structure:

```java
@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class EventEnvelope<T> {
    private UUID eventId;           // UUID v4 — idempotency key
    private String eventType;       // e.g., "remote.request.approved"
    private String version;         // "1" — increment on breaking schema change
    private UUID correlationId;     // propagated from HTTP X-Correlation-ID header
    private Instant occurredAt;     // event timestamp
    private UUID locationId;        // location context
    private T payload;              // event-specific data
}
```

JSON example:
```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "remote.request.approved",
  "version": "1",
  "correlationId": "660e8400-e29b-41d4-a716-446655440001",
  "occurredAt": "2026-04-17T09:15:00Z",
  "locationId": "770e8400-e29b-41d4-a716-446655440002",
  "payload": {
    "requestId": "880e8400-e29b-41d4-a716-446655440003",
    "userId": "990e8400-e29b-41d4-a716-446655440004",
    "dates": ["2026-04-18", "2026-04-19"],
    "approvedBy": "aa0e8400-e29b-41d4-a716-446655440005"
  }
}
```

---

## 4. Spring Cloud Gateway — Auth and Routing

### Session Validation Filter

```java
@Component
@Order(-1)
@RequiredArgsConstructor
public class SessionValidationFilter implements GlobalFilter {

    private final AuthServiceClient authClient;

    private static final Set<String> OPEN_PATHS = Set.of(
        "/api/v1/auth/login",
        "/api/v1/auth/callback",
        "/actuator/health"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        if (OPEN_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        HttpCookie sessionCookie = exchange.getRequest().getCookies().getFirst("SESSION");
        if (sessionCookie == null) {
            return unauthorized(exchange);
        }

        return authClient.validateSession(sessionCookie.getValue())
            .flatMap(userContext -> {
                ServerHttpRequest mutated = exchange.getRequest().mutate()
                    .header("X-User-Id", userContext.getUserId().toString())
                    .header("X-User-Roles", String.join(",", userContext.getRoles()))
                    .header("X-Location-Ids", String.join(",", userContext.getLocationIds()))
                    .build();
                return chain.filter(exchange.mutate().request(mutated).build());
            })
            .onErrorResume(e -> {
                log.warn("Session validation failed: {}", e.getMessage());
                return unauthorized(exchange);
            });
    }
}
```

### Gateway Route Configuration

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: auth-service
          uri: lb://auth-service
          predicates: [Path=/api/v1/auth/**, /api/v1/users/**, /api/v1/locations/**]

        - id: attendance-service
          uri: lb://attendance-service
          predicates: [Path=/api/v1/attendance/**]
          filters:
            - name: CircuitBreaker
              args: { name: attendance-cb, fallbackUri: forward:/fallback/attendance }

        - id: seating-service
          uri: lb://seating-service
          predicates: [Path=/api/v1/seat-bookings/**, /api/v1/seats/**, /api/v1/locations/**/floor-plan]

        - id: remote-service
          uri: lb://remote-service
          predicates: [Path=/api/v1/remote-requests/**, /api/v1/ooo-requests/**, /api/v1/teams/**/schedule, /api/v1/approval-delegates/**, /api/v1/remote-day-policies/**]

        - id: notification-service
          uri: lb://notification-service
          predicates: [Path=/api/v1/notifications/**]

        - id: audit-service
          uri: lb://audit-service
          predicates: [Path=/api/v1/audit-logs/**]

        - id: inventory-service
          uri: lb://inventory-service
          predicates: [Path=/api/v1/supplies/**, /api/v1/supply-requests/**, /api/v1/assets/**, /api/v1/asset-requests/**, /api/v1/fault-reports/**, /api/v1/maintenance-records/**]

        - id: workplace-service
          uri: lb://workplace-service
          predicates: [Path=/api/v1/visitors/**, /api/v1/visits/**, /api/v1/events/**]

      default-filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 100
            redis-rate-limiter.burstCapacity: 150
            redis-rate-limiter.requestedTokens: 1
        - AddRequestHeader=X-Correlation-ID, #{T(java.util.UUID).randomUUID().toString()}
        - DedupeResponseHeader=Access-Control-Allow-Origin
```

---

## 5. Correlation ID Propagation

Every request gets a `correlationId` at the gateway. It is propagated to all downstream services and RabbitMQ messages.

```
Browser request
    → Gateway: generates X-Correlation-ID: abc-123
    → Forwards to seating-service: X-Correlation-ID: abc-123
    → seating-service publishes RabbitMQ event: correlationId: abc-123
    → notification-service consumes: MDC.put("correlationId", "abc-123")
    → audit-service consumes: MDC.put("correlationId", "abc-123")

All log lines from all services: { "correlationId": "abc-123", ... }
Trace in Jaeger: search by traceId or correlationId → full path visible
```

Spring Boot MDC configuration:
```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
        throws ServletException, IOException {
        String correlationId = req.getHeader("X-Correlation-ID");
        if (correlationId != null) {
            MDC.put("correlationId", correlationId);
        }
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("correlationId");
        }
    }
}
```
