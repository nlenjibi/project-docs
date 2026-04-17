# Developer Guide
## Office Management System (OMS)

**Version:** 3.0

Every OMS service follows the same structural, security, and operational standards. Consistency across 8 independently deployed services is what makes the platform maintainable as a coherent system. New developers should read this guide before writing a single line of OMS code.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Language | Java 21 (LTS) | Virtual threads (Project Loom) for scalable blocking I/O; modern pattern matching; long-term support until 2029 |
| Framework | Spring Boot 3.x | Industry standard; rich ecosystem; best Spring Cloud integration |
| Spring Cloud | 2023.x | OpenFeign (declarative REST clients) + LoadBalancer (Eureka integration) |
| Service discovery | Netflix Eureka | Native Spring Cloud integration; `lb://` routing in Gateway; dynamic ECS task IP resolution |
| API Gateway | Spring Cloud Gateway | Native Eureka routing; Java session auth filter; same team tooling |
| Data access | Spring Data JPA + Hibernate | Parameterised queries; easy pagination; Spring Boot auto-config |
| Database | PostgreSQL 16 (per service) | JSONB for audit state; relational integrity; consistent tooling across team |
| Schema migrations | Flyway | Forward-only migrations; version-tracked schema evolution |
| Messaging | Spring Cloud Stream (RabbitMQ binder) | Declarative consumer/producer; automatic retry + DLQ wiring; `StreamBridge` for publishing |
| Resilience | Resilience4j | Circuit breaker, retry, timeout, fallback — all on every Feign call |
| Security | Spring Security (JWT resource server) | Internal JWT validation; `@PreAuthorize` / custom interceptor for RBAC |
| API docs | springdoc-openapi (Swagger UI) | Auto-generated from code; mandatory on all endpoints |
| Observability | Micrometer + Prometheus; OpenTelemetry; CloudWatch Logs | Full metrics, tracing, logging stack |
| Testing | JUnit 5, Mockito, TestContainers, Pact | Unit, integration, contract, resilience |
| Containerisation | Docker (multi-stage) | JRE-only images; non-root user |
| Orchestration | AWS ECS Fargate | Zero cluster management; ECS Service Auto Scaling |
| Build | Maven 3.9+ | Standard Java build tool; reproducible builds |

---

## Project Structure

Every service follows this exact layout. No exceptions. Consistency makes it possible to navigate any service without service-specific orientation.

```
{service-name}/
├── src/main/java/com/yourorg/oms/{domain}/
│   ├── config/
│   │   ├── SecurityConfig.java       # Spring Security; internal JWT decoder
│   │   ├── RabbitMQConfig.java       # Exchange, queue, DLQ declarations
│   │   ├── FeignConfig.java          # Feign interceptors; correlation ID propagation
│   │   ├── Resilience4jConfig.java   # Circuit breaker + retry instances
│   │   └── OpenApiConfig.java        # Swagger UI config
│   │
│   ├── controller/                   # REST controllers — HTTP mapping ONLY, zero business logic
│   ├── service/                      # All business logic — domain logic lives here exclusively
│   ├── repository/                   # Spring Data JPA repositories — data access only
│   ├── entity/                       # JPA entities — NEVER exposed in API responses
│   ├── dto/
│   │   ├── request/                  # Inbound DTOs (validated with @Valid + Bean Validation)
│   │   └── response/                 # Outbound DTOs
│   ├── mapper/                       # Entity ↔ DTO mapping (MapStruct or manual)
│   ├── event/
│   │   ├── publisher/                # StreamBridge publishers + @TransactionalEventListener
│   │   └── consumer/                 # Spring Cloud Stream @Bean Consumer<OmsEvent<T>>
│   ├── security/
│   │   ├── LocationRoleInterceptor.java
│   │   └── RequiresRole.java
│   └── exception/
│       ├── OmsException.java         # Abstract base
│       ├── GlobalExceptionHandler.java
│       └── {Domain}Exception.java    # Specific exceptions per business rule
│
├── src/main/resources/
│   ├── application.yml               # No secrets — all ${ENV_VAR} references
│   ├── application-dev.yml           # Dev overrides (RabbitMQ localhost, etc.)
│   └── db/migration/                 # Flyway migrations: V001__init.sql, V002__...
│
├── src/test/
│   ├── java/com/yourorg/oms/{domain}/
│   │   ├── unit/                     # Service logic, consumer handlers, policy enforcement
│   │   ├── integration/              # Controller + repository tests with TestContainers
│   │   └── contract/                 # Pact consumer / provider tests
│   └── resources/
│       └── application-test.yml      # TestContainers overrides
│
├── Dockerfile                        # Multi-stage; JRE-only runtime image
├── docker-compose.yml                # Local dev: Postgres + RabbitMQ + Eureka
├── .env.example                      # Template — copy to .env for local; NEVER commit .env
└── pom.xml
```

---

## Layer Rules (Strict)

```
HTTP Request
     ↓
Controller   ← HTTP mapping ONLY. Reads DTOs. Returns DTOs. Zero business logic.
     ↓        Never calls Repository directly.
Service      ← ALL business logic. Enforces rules. Calls Repositories. Publishes events.
     ↓        Events published via @TransactionalEventListener AFTER_COMMIT only.
Repository   ← Data access ONLY. Parameterised queries. Always filters by locationId.
     ↓        Never contains business logic.
Database
```

**Absolute rules:**
1. Controllers **never** call Repositories directly.
2. Business logic **never** lives in Controllers or Repositories.
3. Entities are **never** returned in API responses — always map to a DTO first.
4. Services publish events **after** the database transaction commits (`AFTER_COMMIT`).
5. Every state-changing operation publishes an audit event.
6. All access denials produce an audit event.

---

## Anti-Patterns — Forbidden

| Anti-Pattern | Why Forbidden | Correct Alternative |
|-------------|--------------|-------------------|
| Service A reads Service B's database | Breaks service independence; creates tight coupling; physically prevented | Pattern A (local read-model) or Pattern B (Feign) |
| Shared database between any two services | Absolute prohibition — violates bounded context | Each service owns its schema exclusively |
| Business logic in a Controller | Untestable in isolation; violates single responsibility | Move to Service layer |
| Entity returned in API response | Exposes internal schema; breaks API contract on every schema change | Map to a DTO |
| Hardcoded secrets or config values | Security risk; 12-Factor violation | `${ENV_VAR}` references only |
| Stateful service (in-memory state) | Prevents horizontal scaling | `spring-session-jdbc` for sessions; DB for all state |
| Unbounded query (`findAll()` without locationId) | Returns cross-location data; performance risk | Always paginate + `locationId` filter |
| String concatenation in SQL | SQL injection vulnerability | `@Query` with `@Param` always |
| Calling `notification-service` directly via REST | Creates availability dependency; a notification outage would block bookings | Publish to RabbitMQ; let notification-service react |
| Calling `audit-service` directly via REST | Bypasses the immutable write path | Publish `oms.audit.event` to RabbitMQ |
| Synchronous Feign chain > 2 hops | Cascading failure risk; latency multiplication | Use RabbitMQ events for longer workflows |
| Publishing event inside a transaction | Event visible before DB commit; consumers see phantom state | `@TransactionalEventListener(phase = AFTER_COMMIT)` |
| Feign call without circuit breaker | Cascading failure when downstream is slow | All 4 patterns: CB + retry + timeout + fallback |
| Consumer without idempotency check | At-least-once delivery causes duplicate side effects | Check `processedEventRepository` on every consumer |

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Classes | PascalCase | `SeatBookingService`, `AttendanceRecord` |
| Methods | camelCase | `createBooking`, `resolveWorkSession` |
| Variables | camelCase | `userId`, `locationId`, `bookingDate` |
| Booleans | `is`/`has` prefix | `isActive`, `hasDelegate`, `isCrossessMidnight` |
| Constants | `UPPER_SNAKE_CASE` | `DEFAULT_BOOKING_WINDOW_DAYS` |
| REST endpoints | kebab-case | `/api/v1/seat-bookings`, `/api/v1/remote-requests` |
| DB columns | `snake_case` | `first_badge_in`, `location_id`, `booking_date` |
| RabbitMQ routing keys | `oms.{domain}.{entity}.{action}` | `oms.seating.booking.released` |
| RabbitMQ queues | `oms.{consuming-service}.{domain}` | `oms.attendance.remote` |
| Test classes | `{ClassName}Test` (unit), `{ClassName}IT` (integration) | `SeatBookingServiceTest`, `SeatBookingControllerIT` |
| Feign client interfaces | `{Service}Client` | `AuthServiceClient`, `SeatingServiceClient` |
| Event payload records | `{Action}Payload` | `SeatBookingCreatedPayload`, `UserDeactivatedPayload` |

**Never use:**
- Abbreviations: `usr`, `svc`, `loc`, `repo`, `mgr`
- Vague names: `data`, `temp`, `obj`, `result`, `thing`
- Hungarian notation: `strName`, `intCount`

---

## Kafka/RabbitMQ Patterns

### Publishing (After Transaction Commit)

```java
@Service
@RequiredArgsConstructor
public class SeatBookingService {
    private final SeatBookingRepository repository;
    private final ApplicationEventPublisher domainEventPublisher;

    @Transactional
    public SeatBookingResponse book(SeatBookingRequest request, UUID actorId) {
        // Validate
        if (!seatRepository.isAvailable(request.getSeatId(), request.getBookingDate())) {
            throw new SeatUnavailableException();
        }

        // Persist
        SeatBooking booking = repository.save(mapToEntity(request, actorId));

        // Publish domain event — deferred to AFTER_COMMIT
        domainEventPublisher.publishEvent(new SeatBookingCreatedDomainEvent(booking));

        return mapToResponse(booking);
    }
}
```

### Consuming (Idempotent — Mandatory)

```java
@Bean
public Consumer<OmsEvent<RemoteApprovedPayload>> handleRemoteApproved() {
    return event -> {
        // 1. Idempotency guard — FIRST, before any business logic
        if (processedEventRepository.existsById(event.eventId())) {
            log.info("Duplicate event ignored: eventId={}", event.eventId());
            return;
        }

        // 2. Business logic
        for (LocalDate date : event.payload().dates()) {
            attendanceRepository.upsertStatus(
                event.payload().userId(), date, event.locationId(), "REMOTE");
        }

        // 3. Mark processed in the SAME transaction as business logic
        processedEventRepository.save(
            new ProcessedEvent(event.eventId(), Instant.now()));
        // If this transaction rolls back, both the upsert AND the processed_event
        // are rolled back → safe re-delivery on next attempt
    };
}
```

---

## Error Handling

### Custom Exception Hierarchy

```java
// Abstract base — every domain exception extends this
public abstract class OmsException extends RuntimeException {
    private final String code;
    private final HttpStatus status;

    protected OmsException(String code, String message, HttpStatus status) {
        super(message);
        this.code = code;
        this.status = status;
    }
}

// Examples
public class SeatUnavailableException extends OmsException {
    public SeatUnavailableException() {
        super("SEAT_UNAVAILABLE",
              "This seat is no longer available for the selected date.",
              HttpStatus.CONFLICT);
    }
}

public class BookingWindowExceededException extends OmsException {
    public BookingWindowExceededException(int windowDays) {
        super("BOOKING_WINDOW_EXCEEDED",
              "Booking date exceeds the configured " + windowDays + "-day window.",
              HttpStatus.BAD_REQUEST);
    }
}
```

### Global Handler (One Per Service)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OmsException.class)
    public ResponseEntity<ApiResponse<Void>> handleDomain(OmsException ex) {
        return ResponseEntity.status(ex.getStatus())
            .body(ApiResponse.error(ex.getCode(), ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(
            MethodArgumentNotValidException ex) {
        String field = ex.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getField).findFirst().orElse("unknown");
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage).findFirst().orElse("Validation failed");
        return ResponseEntity.badRequest()
            .body(ApiResponse.fieldError("VALIDATION_ERROR", message, field));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        log.error("Unhandled exception: correlationId={}",
            CorrelationIdHolder.get(), ex);
        return ResponseEntity.status(500)
            .body(ApiResponse.error("INTERNAL_ERROR",
                "An unexpected error occurred. Reference: " + CorrelationIdHolder.get()));
    }
}
```

---

## API Documentation (Mandatory)

Every endpoint must have Swagger annotations. **No undocumented endpoint merges to `main`.**

```java
@Operation(
    summary = "Book a hot-desk seat",
    description = "Books an available hot-desk for the specified date. " +
                  "Enforces the location booking window and optimistic lock " +
                  "for concurrent booking attempts."
)
@ApiResponse(responseCode = "201", description = "Booking confirmed")
@ApiResponse(responseCode = "400", description = "Validation error or booking window exceeded")
@ApiResponse(responseCode = "409", description = "Seat already booked for that date")
@SecurityRequirement(name = "sessionCookie")
@PostMapping
public ResponseEntity<ApiResponse<SeatBookingResponse>> bookSeat(
        @Valid @RequestBody SeatBookingRequest request) { ... }
```

Swagger UI available at: `http://localhost:8080/swagger-ui/index.html`

---

## Testing Strategy

### Test Pyramid

| Level | Tool | What to Test | Coverage Target |
|-------|------|-------------|----------------|
| Unit | JUnit 5 + Mockito | All service logic; consumer handlers; policy enforcement; job resolution logic | ≥ 80% on new code |
| Integration | Spring Boot Test + TestContainers | All REST endpoints; repository queries with real DB; RabbitMQ producer/consumer pairs | All happy paths + key error paths |
| Contract | Pact | Every Feign consumer-provider relationship | Every inter-service API call |
| Security | MockMvc + role simulation | Every endpoint against every applicable role + location combination | All protected endpoints |
| Resilience | Resilience4j test mode | Circuit breaker, retry, timeout, fallback behaviour | Every Feign client |

### TestContainers Integration Test (Example)

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@AutoConfigureMockMvc
@Testcontainers
class SeatBookingControllerIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("seating_db_test");

    @Container
    static RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:3.13-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.rabbitmq.host", rabbit::getHost);
        registry.add("spring.rabbitmq.port", rabbit::getAmqpPort);
    }

    @Test
    void bookSeat_confirmedBooking_returns201() throws Exception {
        // Arrange: seed floor plan + seat in test DB
        UUID seatId = seedSeat();

        // Act
        mockMvc.perform(post("/api/v1/seat-bookings")
            .header("X-User-Id", TEST_USER_ID)
            .header("X-User-Roles", "EMPLOYEE")
            .header("X-Location-Id", TEST_LOCATION_ID)
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                { "seatId": "%s", "bookingDate": "2026-02-01", "locationId": "%s" }
                """.formatted(seatId, TEST_LOCATION_ID)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data.status").value("CONFIRMED"));
    }
}
```

### Pact Contract Testing

When `seating-service` calls `auth-service`:
1. `seating-service` defines the Pact contract (as **consumer**) in its test.
2. `auth-service` verifies the contract in its own CI pipeline (as **provider**).
3. Neither service can deploy a breaking API change without the contract test failing in CI first.

```java
// seating-service (consumer side)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "auth-service")
class AuthServiceClientPactTest {

    @Pact(consumer = "seating-service")
    RequestResponsePact getDirectReports(PactDslWithProvider builder) {
        return builder
            .given("manager alice has 3 direct reports in Accra")
            .uponReceiving("get direct reports for manager")
                .path("/api/v1/users/" + MANAGER_ID + "/direct-reports")
                .method("GET")
                .query("locationId=" + LOCATION_ID)
            .willRespondWith()
                .status(200)
                .body(LambdaDsl.newJsonArrayMinLike(1, o ->
                    o.uuid("userId").stringType("displayName").stringType("email")
                ).build())
            .toPact();
    }
}
```

### Resilience Tests

```java
@Test
void getDirectReports_circuitBreakerOpen_returnsFallback() {
    // Simulate auth-service returning 500 enough times to open the circuit
    for (int i = 0; i < 10; i++) {
        authServiceWireMock.stubFor(get(anyUrl()).willReturn(serverError()));
        catchThrowable(() -> teamAttendanceService.getTeamAttendance(managerId, locationId, range, page));
    }

    // Now the circuit is OPEN — fallback should return empty list immediately
    List<UUID> result = teamAttendanceService.getDirectReports(managerId, locationId);
    assertThat(result).isEmpty();
    assertThat(circuitBreakerRegistry.circuitBreaker("auth-service").getState())
        .isEqualTo(CircuitBreaker.State.OPEN);
}
```

### Scheduled Job Tests

All scheduled jobs must be unit-testable in isolation:

```java
@Test
void badgeSyncJob_idempotent_doesNotCreateDuplicatesOnRerun() {
    // Run for 2026-01-15
    badgeSyncJob.runForDate(LocalDate.of(2026, 1, 15), locationId);
    long countAfterFirst = attendanceRepository.countByRecordDate(LocalDate.of(2026, 1, 15));

    // Re-run for the same date — should not create duplicates
    badgeSyncJob.runForDate(LocalDate.of(2026, 1, 15), locationId);
    long countAfterSecond = attendanceRepository.countByRecordDate(LocalDate.of(2026, 1, 15));

    assertThat(countAfterSecond).isEqualTo(countAfterFirst);
}
```

---

## Configuration Management (12-Factor)

All configuration via environment variables only. No hardcoded values. No secrets in any config file.

```yaml
# application.yml — uses ENV VAR references throughout
spring:
  application:
    name: seating-service
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT:5432}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}          # Injected from Secrets Manager at task startup
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}    # From Secrets Manager
    ssl.enabled: ${RABBITMQ_SSL:false}

eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka/}
  instance:
    preferIpAddress: true

internal:
  jwt:
    secret: ${INTERNAL_JWT_SECRET}    # From Secrets Manager

# Seating-specific
booking:
  windowDays: ${BOOKING_WINDOW_DAYS:14}
  cancellationCutoffHours: ${CANCELLATION_CUTOFF_HOURS:2}
  noShowReleaseCron: ${NO_SHOW_RELEASE_CRON:0 0 10 * * *}
```

```bash
# .env.example — developers copy to .env for local dev
DB_HOST=localhost
DB_PORT=5432
DB_NAME=seating_db
DB_USERNAME=seating_service
DB_PASSWORD=local_dev_only

RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest

EUREKA_URL=http://localhost:8761/eureka/
INTERNAL_JWT_SECRET=local-dev-secret-change-in-prod

BOOKING_WINDOW_DAYS=14
CANCELLATION_CUTOFF_HOURS=2
```

`.env` is in `.gitignore` — never committed. `.env.example` is committed without real values.

---

## Git Workflow

### Branch Strategy

```
main          ← Production. Protected. PR + review + manual approval gate.
staging       ← Staging environment. Auto-deployed on merge. Pact provider tests run here.
develop       ← Integration branch. PR + peer review required.
feature/<n>   ← Feature branches off develop
fix/<n>       ← Bug fix branches off develop
hotfix/<n>    ← Critical production fixes off main (merged to main + develop)
```

### Commit Message Format (Conventional Commits)

```
<type>(<scope>): <summary — max 50 chars, imperative mood, no trailing period>

<body — what changed and why, 72 chars per line, optional>

<footer — Closes OMS-123>
```

**Types:** `feat` · `fix` · `chore` · `docs` · `refactor` · `style` · `perf` · `test` · `build` · `ci`

```
feat(seating): implement no-show auto-release scheduled job

The NoShowReleaseJob runs daily at no_show_release_time per location.
It queries CONFIRMED bookings with no matching work session by the
cutoff, marks them RELEASED, creates a NoShowRecord, and publishes
oms.seating.booking.released to RabbitMQ.

Closes OMS-198

---

feat(attendance): implement two-pass nightly badge resolution

Pass 1: Athena badge events → WorkSession → AttendanceRecord PRESENT/LATE.
Pass 2: RabbitMQ consumers for remote.request.approved and
ooo.request.approved overlay REMOTE/OOO on existing records.

Closes OMS-142
```

### Definition of Done (Every Ticket)

- [ ] Code reviewed and approved (minimum 1 reviewer)
- [ ] Unit test coverage ≥ 80% on new code
- [ ] Integration tests passing with TestContainers
- [ ] Pact contract tests passing (if REST relationship involved)
- [ ] No hardcoded secrets or credentials in any file
- [ ] Swagger annotations on all new endpoints
- [ ] Audit event published on all new state-changing operations
- [ ] Role + location interceptor applied on all new protected endpoints
- [ ] Parameterised SQL only — no string concatenation
- [ ] `locationId` filter on all new repository queries
- [ ] All Feign calls have circuit breaker + fallback
- [ ] RabbitMQ consumers are idempotent (`processedEventRepository` check)
- [ ] CI pipeline green: lint → unit tests → integration tests → contract tests → build
- [ ] Acceptance criteria signed off by Product Owner
