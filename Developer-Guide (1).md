# Developer Guide
## Office Management System (OMS) — Microservices Edition

**Version:** 2.0  
**Status:** Draft  
**Last Updated:** March 2026  

---

## 1. Introduction

This guide is the authoritative reference for engineers building and maintaining OMS microservices. Every service in the OMS follows the same structural, security, and operational standards documented here. Consistency across services is not optional — it is what makes 11 independently deployed services manageable as a coherent platform.

**What this guide covers:**
- How to create, structure, and run a service
- Coding standards, naming conventions, and layer rules
- Security requirements (Zero Trust — non-negotiable)
- Kafka event patterns and idempotency
- Resilience patterns for inter-service REST calls
- Observability standards — logging, metrics, tracing, health
- Testing strategy including contract tests
- Git workflow and commit standards

**Tech stack per service:**

| Layer | Technology |
|-------|-----------|
| Language | Java 21 |
| Framework | Spring Boot 3.x |
| Data access | Spring Data JPA + Hibernate |
| Database | PostgreSQL (per service) |
| Messaging | Apache Kafka (Spring Kafka) |
| Resilience | Resilience4j |
| Security | Spring Security + internal JWT |
| API docs | springdoc-openapi (Swagger UI) |
| Observability | Micrometer + Prometheus, OpenTelemetry |
| Testing | JUnit 5, Mockito, TestContainers, Pact |
| Containerisation | Docker |
| Orchestration | Kubernetes (AWS EKS) |

---

## 2. Creating a New Service

### 2.1 Project Structure

Every service follows this exact structure. No exceptions.

```
{service-name}/
├── src/main/java/com/yourorg/oms/{domain}/
│   ├── config/              # Spring config: Security, Kafka, OpenAPI, Resilience4j
│   ├── controller/          # REST controllers — HTTP only, no business logic
│   ├── service/             # Business logic — all domain logic lives here
│   ├── repository/          # Spring Data JPA repositories
│   ├── entity/              # JPA entities — NEVER exposed in API responses
│   ├── dto/
│   │   ├── request/         # Inbound DTOs (validated with @Valid)
│   │   └── response/        # Outbound DTOs
│   ├── mapper/              # Entity ↔ DTO mapping
│   ├── event/
│   │   ├── producer/        # Kafka producers
│   │   └── consumer/        # Kafka consumers
│   └── exception/           # Custom exceptions
├── src/main/resources/
│   └── application.yml      # No secrets; references env vars only
├── src/test/
│   ├── unit/                # Service logic tests
│   ├── integration/         # Controller + repository tests (TestContainers)
│   └── contract/            # Pact consumer/provider tests
├── Dockerfile
└── pom.xml
```

### 2.2 Prerequisites

| Tool | Version |
|------|---------|
| Java | 21 (LTS) |
| Maven | 3.9+ |
| Docker | 25+ |
| Docker Compose | 2.x |
| Node.js | 20 LTS (for Angular frontend only) |

### 2.3 Running a Service Locally

```bash
# Clone the repo
git clone <repo-url>
cd oms/{service-name}

# Set environment variables (never commit .env)
cp .env.example .env

# Run with local Docker Compose (spins up Postgres + Kafka)
docker-compose up --build

# Or run directly
./mvnw spring-boot:run
```

### 2.4 Local Service Ports

| Service | Port |
|---------|------|
| identity-service | 8081 |
| user-service | 8082 |
| attendance-service | 8083 |
| seating-service | 8084 |
| remote-service | 8085 |
| notification-service | 8086 |
| audit-service | 8087 |
| visitor-service | 8088 |
| event-service | 8089 |
| supplies-service | 8090 |
| assets-service | 8091 |

Swagger UI per service: `http://localhost:{port}/swagger-ui/index.html`  
OpenAPI JSON: `http://localhost:{port}/v3/api-docs`

---

## 3. Architecture Rules

### 3.1 Layer Structure — Strict

```
HTTP Request
     ↓
Controller      ← HTTP mapping only. Reads DTOs. Returns DTOs. Zero business logic.
     ↓
Service         ← All business logic. Calls repositories. Publishes Kafka events.
     ↓
Repository      ← Data access only. Parameterised queries. Location filters always applied.
     ↓
Database
```

**Absolute rules:**
- Controllers **never** call Repositories directly
- Business logic **never** lives in Controllers
- Entities are **never** returned in API responses — always map to a DTO first
- DTOs are used for all API input and output
- Services publish Kafka events and audit events **after** committing the transaction

### 3.2 Anti-Patterns — Forbidden

| Anti-Pattern | Why Forbidden |
|-------------|--------------|
| Service A reads Service B's database | Breaks service independence; creates tight coupling |
| Shared database between services | Forbidden absolutely — own your data |
| Business logic in Controller | Untestable; violates SRP |
| Entity in API response | Exposes internal schema; breaks API contract |
| Hardcoded secrets or config values | 12-Factor violation; security risk |
| Stateful service (in-memory session state) | Prevents horizontal scaling |
| Unbounded database query (`findAll()`) | Performance risk; always paginate |
| String concatenation in SQL | SQL injection vulnerability |
| Calling `notification-service` or `audit-service` directly | Publish to Kafka; never call these services via REST |

---

## 4. Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Classes | PascalCase | `SeatBookingService`, `AttendanceRecord` |
| Methods | camelCase | `createBooking`, `resolveWorkSession` |
| Variables | camelCase | `userId`, `locationId` |
| Booleans | `is`/`has` prefix | `isActive`, `hasDelegate` |
| Constants | UPPER_CASE_WITH_UNDERSCORES | `DEFAULT_BOOKING_WINDOW_DAYS` |
| REST endpoints | kebab-case | `/api/v1/seat-bookings` |
| DB columns | snake_case | `first_badge_in`, `location_id` |
| Kafka topics | `oms.{domain}.{entity}.{action}` | `oms.seating.booking.released` |
| Kafka consumer group | `oms-{service}-{domain}` | `oms-attendance-remote` |

**Never** use abbreviations (`usr`, `svc`, `loc`). **Never** use vague names (`data`, `temp`, `obj`).

---

## 5. Security Rules — Non-Negotiable

### 5.1 Authentication and Session

- Authentication is SSO-only, managed entirely by `identity-service`.
- The API Gateway validates every inbound session by calling `identity-service /api/v1/auth/validate` before routing.
- The frontend never handles, stores, or inspects raw tokens.
- Sessions are HTTP-only cookies — inaccessible to JavaScript.

### 5.2 Zero Trust — Service-to-Service

Every service validates the caller's identity on every inbound request. Network position alone grants no trust.

```java
// Spring Security config on every service — validate internal JWT on every request
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.decoder(internalJwtDecoder()))
    );
    return http.build();
}
```

All service-to-service communication uses mutual TLS (mTLS) configured at the service mesh level (Istio / App Mesh).

### 5.3 Role + Location Authorisation

Role and location checks are enforced **within each service** — not only at the gateway.

```java
@PreAuthorize("@locationRoleChecker.hasRoleAtLocation(#locationId, 'MANAGER')")
public SeatBookingResponse bookForTeam(UUID locationId, BlockBookingRequest request) { ... }
```

Super Admin has NULL on `location_id` in their `UserRole` record — the check must handle this explicitly.

### 5.4 Parameterised SQL — No Exceptions

```java
// ✅ Correct
@Query("SELECT s FROM SeatBooking s WHERE s.userId = :userId AND s.locationId = :locationId AND s.deletedAt IS NULL")
List<SeatBooking> findActiveByUserAndLocation(
    @Param("userId") UUID userId,
    @Param("locationId") UUID locationId);

// ❌ Forbidden — never do this
String query = "SELECT * FROM seat_bookings WHERE user_id = '" + userId + "'";
```

### 5.5 Other Security Rules

- **Never** hardcode secrets, passwords, or API keys. Use environment variables exclusively.
- **Never** log passwords, tokens, or sensitive PII.
- **Never** expose stack traces in API responses. Use generic error messages externally.
- **Always** validate all inputs via DTOs with `@Valid`.
- **Always** apply the minimum DB privilege principle — each service's DB user has only the privileges it needs.
- Audit service DB user has INSERT only — no UPDATE, no DELETE.

---

## 6. Data Model Conventions

### 6.1 Standard Fields (All Entities)

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;

private Instant createdAt;
private Instant updatedAt;
private UUID createdBy;           // FK → User (resolved from session context)
```

Feature entities (not `identity-service` or `user-service` global entities) also include:

```java
private UUID locationId;          // FK → Location — MANDATORY; never omit
```

Entities that support deletion include:

```java
private Instant deletedAt;        // Soft delete — null = active record
```

### 6.2 Location Scoping

**Every repository query against a feature entity must filter by `locationId`.** This rule has no exceptions.

```java
// ✅ Correct
List<SeatBooking> findByLocationIdAndDeletedAtIsNull(UUID locationId);

// ❌ Wrong — will leak cross-location data
List<SeatBooking> findAll();
```

### 6.3 Soft Deletes

Hard deletes are not performed. Set `deletedAt` to mark a record as deleted. All active-record queries filter `WHERE deleted_at IS NULL` — this is enforced in the Repository layer, not assumed in Service code.

### 6.4 UUID Primary Keys

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

Never use sequential integers. UUIDs prevent enumeration attacks and support future distributed architectures.

---

## 7. Kafka Patterns

### 7.1 Publishing Events

Every service publishes events after committing the database transaction. Never publish before the transaction commits.

```java
@Service
@RequiredArgsConstructor
public class SeatBookingService {
    private final SeatBookingRepository repository;
    private final KafkaEventPublisher publisher;
    private final AuditEventPublisher auditPublisher;

    @Transactional
    public SeatBookingResponse book(SeatBookingRequest request) {
        // 1. Business logic and validation
        // 2. Persist
        SeatBooking booking = repository.save(mapToEntity(request));
        // 3. Publish domain event (after commit)
        publisher.publish("oms.seating.booking.created", SeatBookingCreatedEvent.from(booking));
        // 4. Publish audit event
        auditPublisher.publish(AuditEvent.of("SEAT_BOOKED", "SeatBooking", booking.getId(),
                null, booking, currentUser));
        return mapToResponse(booking);
    }
}
```

### 7.2 Event Envelope

All events use this standard envelope:

```java
public record OmsEvent<T>(
    UUID eventId,                  // UUID v4 — unique per event
    String eventType,              // e.g. "seating.booking.created"
    String version,                // "1" — increment on breaking schema change
    UUID correlationId,            // Propagated from the originating HTTP request
    Instant occurredAt,
    UUID locationId,
    T payload
) {}
```

### 7.3 Kafka Consumer — Idempotency (Mandatory)

Every Kafka consumer must be idempotent. Kafka guarantees at-least-once delivery — duplicate events are expected.

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

    // Mark as processed (within the same transaction)
    processedEventRepository.save(new ProcessedEvent(event.eventId()));
}
```

### 7.4 Kafka Topic Naming

```
oms.{domain}.{entity}.{action}

Examples:
  oms.remote.request.approved
  oms.seating.booking.released
  oms.attendance.record.resolved
  oms.audit.event
  oms.user.user.deactivated
```

Consumer group naming: `oms-{consuming-service}-{domain}`

```
oms-attendance-remote       (attendance-service consuming remote events)
oms-notification-seating    (notification-service consuming seating events)
oms-audit-all               (audit-service consuming all audit events)
```

---

## 8. Resilience Patterns

Every synchronous REST call to another service **must** implement all four patterns. No inter-service call is made without them.

```java
// application.yml — per downstream service
resilience4j:
  circuitbreaker:
    instances:
      user-service:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
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

```java
@CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
@Retry(name = "user-service")
@TimeLimiter(name = "user-service")
public CompletableFuture<UserResponse> getUser(UUID userId) {
    return CompletableFuture.supplyAsync(() ->
        userServiceClient.getUser(userId));
}

// Fallback: always return a safe degraded response
private CompletableFuture<UserResponse> getUserFallback(UUID userId, Exception ex) {
    log.warn("user-service unavailable for userId={}, using fallback", userId, ex);
    return CompletableFuture.completedFuture(UserResponse.unknown(userId));
}
```

**Rules:**
- Never block indefinitely — 3s timeout is a hard ceiling
- Retry maximum is 3 attempts with exponential backoff — do not retry indefinitely
- Always define a fallback — never let a downstream failure propagate an unhandled exception
- Log warnings when falling back; alert when circuit breaker OPEN state persists

---

## 9. Error Handling

### 9.1 Custom Exceptions

```java
// Base exception for all OMS domain errors
public abstract class OmsException extends RuntimeException {
    private final String code;
    private final HttpStatus status;

    protected OmsException(String code, String message, HttpStatus status) {
        super(message);
        this.code = code;
        this.status = status;
    }
}

// Example domain exception
public class BookingWindowExceededException extends OmsException {
    public BookingWindowExceededException() {
        super("BOOKING_WINDOW_EXCEEDED",
              "Booking date exceeds the configured window for this location.",
              HttpStatus.BAD_REQUEST);
    }
}
```

### 9.2 Global Exception Handler (One Per Service)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OmsException.class)
    public ResponseEntity<ApiResponse<Void>> handleDomainException(OmsException ex) {
        return ResponseEntity.status(ex.getStatus())
            .body(ApiResponse.error(ex.getCode(), ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.status(500)
            .body(ApiResponse.error("INTERNAL_ERROR", "An unexpected error occurred."));
    }
}
```

### 9.3 Standard API Response Envelope

```java
public record ApiResponse<T>(
    boolean success,
    T data,
    ErrorDetail error
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null);
    }

    public static ApiResponse<Void> error(String code, String message) {
        return new ApiResponse<>(false, null, new ErrorDetail(code, message));
    }
}
```

JSON output:
```json
// Success
{ "success": true, "data": { ... }, "error": null }

// Error
{ "success": false, "data": null, "error": { "code": "BOOKING_WINDOW_EXCEEDED", "message": "..." } }
```

---

## 10. Observability Standards

Every service implements all four pillars. No exceptions.

### 10.1 Structured Logging (JSON to Stdout)

```java
// Every log entry includes service name and correlation ID
log.info("Seat booking confirmed",
    kv("service", "seating-service"),
    kv("correlationId", correlationId),
    kv("userId", userId),
    kv("seatId", seatId),
    kv("locationId", locationId),
    kv("action", "SEAT_BOOKED"));
```

**Rules:**
- Output to stdout only — no local log files (12-Factor)
- Never log passwords, tokens, or PII
- Always include `correlationId` in every log entry
- Use JSON structured logging (Logback + logstash-logback-encoder)

### 10.2 Metrics (Prometheus)

Every service exposes `/actuator/prometheus`. Required metrics:

```
http_request_duration_seconds{service, endpoint, status}
http_requests_total{service, endpoint, status}
job_duration_seconds{service, job_name}         # Scheduled jobs
job_failures_total{service, job_name}
kafka_consumer_lag{service, topic, group}
circuit_breaker_state{service, name}            # 0=CLOSED, 1=OPEN, 2=HALF_OPEN
db_connection_pool_active{service}
```

### 10.3 Distributed Tracing (OpenTelemetry)

The API Gateway generates a `correlationId` (UUID v4) for every inbound request and injects it as `X-Correlation-ID` header. Every service:

- Reads `X-Correlation-ID` from inbound HTTP headers
- Propagates it on every outbound HTTP call
- Includes it in every Kafka event envelope
- Includes it in every log entry

```java
// Spring Boot auto-configures OpenTelemetry trace propagation via the OTel SDK
// Manual injection for Kafka events:
ProducerRecord<String, OmsEvent<?>> record = new ProducerRecord<>(topic, event);
record.headers().add("X-Correlation-ID", correlationId.toString().getBytes());
```

### 10.4 Health Checks

```yaml
# application.yml — every service
management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,prometheus,info
```

Kubernetes probes:
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
```

---

## 11. API Design Standards

### 11.1 Versioned REST Endpoints

All APIs are versioned from day one: `/api/v1/...`

```java
@RestController
@RequestMapping("/api/v1/seat-bookings")
@Tag(name = "Seat Bookings")
public class SeatBookingController { ... }
```

### 11.2 Swagger / OpenAPI (Mandatory)

Every endpoint must be documented. No undocumented endpoint is permitted to merge.

```java
@Operation(
    summary = "Book a hot-desk seat",
    description = "Books an available hot-desk for the specified date. Enforces the location booking window."
)
@ApiResponse(responseCode = "201", description = "Booking created")
@ApiResponse(responseCode = "400", description = "Invalid input or window exceeded")
@ApiResponse(responseCode = "409", description = "Seat no longer available")
@PostMapping
public ResponseEntity<ApiResponse<SeatBookingResponse>> bookSeat(
        @Valid @RequestBody SeatBookingRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(ApiResponse.ok(seatBookingService.book(request)));
}
```

### 11.3 Pagination (All List Endpoints)

```
GET /api/v1/seat-bookings?page=0&size=20&sort=createdAt,desc
```

```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  },
  "error": null
}
```

---

## 12. Configuration Management (12-Factor)

All configuration is managed via **environment variables only**. No hardcoded values. No config files with secrets committed to source control.

```bash
# Common to all services
DB_HOST=localhost
DB_PORT=5432
DB_NAME=seating_db
DB_USERNAME=seating_service
DB_PASSWORD=${DB_PASSWORD}         # Injected from Secrets Manager at runtime

KAFKA_BOOTSTRAP_SERVERS=kafka:9092
INTERNAL_JWT_SECRET=${INTERNAL_JWT_SECRET}
CORRELATION_ID_HEADER=X-Correlation-ID

# Service-specific example (seating-service)
BOOKING_WINDOW_DAYS=14
CANCELLATION_CUTOFF_HOURS=2
NO_SHOW_RELEASE_CRON=0 10 * * *
```

Three environments: `dev`, `staging`, `prod`. Only environment variables differ between them. Dev/prod parity is maintained.

---

## 13. Testing Strategy

| Level | Tool | Scope |
|-------|------|-------|
| Unit | JUnit 5 + Mockito | All service logic, Kafka consumer handlers, policy enforcement |
| Integration | Spring Boot Test + TestContainers | All REST endpoints, repository queries, Kafka producer/consumer pairs |
| Contract | Pact | Every consumer-provider REST relationship; prevents silent breaking changes |
| Security | MockMvc + role simulation | Every endpoint vs every role+location combination |
| Resilience | Resilience4j test mode | Circuit breaker, retry, timeout, fallback behaviour |
| Performance | k6 | Nightly sync job; seat availability under concurrency; Kafka lag under peak volume |

**Pact contract testing rule:** When `seating-service` calls `user-service`, `seating-service` defines a Pact contract (as consumer). `user-service` verifies this contract in its own CI pipeline (as provider). Neither service can make a breaking API change without the contract test failing first.

---

## 14. Scheduled Jobs

| Job | Service | Schedule | Rules |
|-----|---------|----------|-------|
| Badge Sync | `attendance-service` | Nightly (configurable, e.g. 2am) | Idempotent; retry on failure; alert on non-completion |
| No-Show Release | `seating-service` | Daily at `no_show_release_time` (per location) | Runs per location; bulkhead isolated from API requests |
| HR Sync | `user-service` | Scheduled (TBD) | Idempotent upsert; alert on Arms API failure |

All scheduled jobs must:
- Log start, end, and error states to stdout in structured JSON
- Publish an `audit.event` on completion (success or failure)
- Be idempotent — re-running on the same date must not create duplicate records
- Be bulkhead-isolated from the service's API request-handling thread pool

---

## 15. Git Workflow and Commit Standards

### 15.1 Branch Strategy

```
main          ← Production-ready. Protected. PR + review required.
staging       ← Staging environment. Auto-deployed on merge.
develop       ← Integration branch. PR + review required.
feature/<n>   ← Feature branches off develop
fix/<n>       ← Bug fix branches off develop
hotfix/<n>    ← Hotfix branches off main
```

### 15.2 Conventional Commits

```
<type>(<scope>): <short summary — max 50 chars, imperative, no period>

<body — what and why; 72 chars per line>

<footer — Closes OMS-123>
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `style`, `perf`, `test`, `build`, `ci`

**Examples:**
```
feat(seating): implement no-show auto-release scheduled job

The daily no-show release job queries confirmed bookings with no
badge-in event by the configured cutoff time, marks them RELEASED,
creates a NoShowRecord, and publishes seat.booking.released to Kafka.
Notification-service dispatches the daily Facilities Admin summary.

Closes OMS-198

feat(attendance): implement two-pass nightly resolution job

Pass 1: badge events from Athena → WorkSession → AttendanceRecord.
Pass 2: Kafka consumer for remote.request.approved and
ooo.request.approved overlays REMOTE/OOO statuses. Remaining
user-dates with no event are stamped ABSENT.

Closes OMS-142
```

---

## 16. Useful References

- Microservices Architecture: `micro-architecture.md`
- Architecture Decision Records: `doch.md` (AD-001 through AD-026)
- Data Model Reference: `dock.md`
- System Overview: `doc`
- Microservices.io patterns: https://microservices.io/patterns
- Twelve-Factor App: https://12factor.net
- OWASP Top 10: https://owasp.org/www-project-top-ten
- Resilience4j docs: https://resilience4j.readme.io
- Pact contract testing: https://docs.pact.io

---

*End of Developer Guide*
