# Developer Guide
## Office Management System (OMS)

Every OMS service follows the same structural, security, and operational standards. Consistency across services is what makes 11 independently deployed services manageable as a coherent platform.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 21 (LTS) |
| Framework | Spring Boot 3.x |
| Data access | Spring Data JPA + Hibernate |
| Database | PostgreSQL (per service) |
| Schema migrations | Flyway |
| Messaging | Apache Kafka (Spring Kafka) + AWS SQS (Spring Cloud AWS) |
| Resilience | Resilience4j |
| Security | Spring Security + internal JWT |
| API docs | springdoc-openapi (Swagger UI) |
| Observability | Micrometer + Prometheus, OpenTelemetry |
| Testing | JUnit 5, Mockito, TestContainers, Pact |
| Containerisation | Docker |
| Orchestration | Kubernetes (AWS EKS) |
| Build | Maven 3.9+ |

---

## Project Structure

Every service follows this exact structure. No exceptions.

```
{service-name}/
├── src/main/java/com/yourorg/oms/{domain}/
│   ├── config/          # Spring config — Security, Kafka, OpenAPI, Resilience4j
│   ├── controller/      # REST controllers — HTTP only, zero business logic
│   ├── service/         # All business logic — domain logic lives here exclusively
│   ├── repository/      # Spring Data JPA repositories — data access only
│   ├── entity/          # JPA entities — NEVER exposed in API responses
│   ├── dto/
│   │   ├── request/     # Inbound DTOs (validated with @Valid)
│   │   └── response/    # Outbound DTOs
│   ├── mapper/          # Entity ↔ DTO mapping
│   ├── event/
│   │   ├── producer/    # Kafka and SQS publishers
│   │   └── consumer/    # Kafka and SQS listeners
│   └── exception/       # Custom OmsException subclasses
├── src/main/resources/
│   └── application.yml  # No secrets — references env vars only
├── src/test/
│   ├── unit/            # Service logic, consumer handlers, policy enforcement
│   ├── integration/     # Controller + repository tests (TestContainers)
│   └── contract/        # Pact consumer / provider tests
├── Dockerfile
└── pom.xml
```

---

## Layer Rules (Strict)

```
HTTP Request
     ↓
Controller   ← HTTP mapping ONLY. Reads DTOs. Returns DTOs. Zero business logic.
     ↓
Service      ← ALL business logic. Calls repositories. Publishes events.
     ↓
Repository   ← Data access ONLY. Parameterised queries. Always filters by locationId.
     ↓
Database
```

**Absolute rules:**
- Controllers **never** call Repositories directly.
- Business logic **never** lives in Controllers.
- Entities are **never** returned in API responses — always map to a DTO first.
- Services publish Kafka / SQS events **after** committing the database transaction.
- All access denials must produce an audit event.

---

## Anti-Patterns — Forbidden

| Anti-Pattern | Why Forbidden |
|-------------|--------------|
| Service A reads Service B's database | Breaks service independence; tight coupling |
| Shared database between services | Absolute prohibition — own your data |
| Business logic in a Controller | Untestable; violates single responsibility |
| Entity in an API response | Exposes internal schema; breaks API contract |
| Hardcoded secrets or config values | 12-Factor violation; security risk |
| Stateful service (in-memory session state) | Prevents horizontal scaling |
| Unbounded database query (`findAll()`) | Performance risk; always paginate |
| String concatenation in SQL | SQL injection vulnerability |
| Calling `notification-service` directly | Publish to Kafka/SQS; never call via REST |
| Calling `audit-service` directly | Publish to Kafka/SQS; never call via REST |
| Synchronous call chain > 2 hops | Use Kafka for longer workflows |

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Classes | PascalCase | `SeatBookingService`, `AttendanceRecord` |
| Methods | camelCase | `createBooking`, `resolveWorkSession` |
| Variables | camelCase | `userId`, `locationId` |
| Booleans | `is`/`has` prefix | `isActive`, `hasDelegate` |
| Constants | UPPER_CASE_UNDERSCORES | `DEFAULT_BOOKING_WINDOW_DAYS` |
| REST endpoints | kebab-case | `/api/v1/seat-bookings` |
| DB columns | snake_case | `first_badge_in`, `location_id` |
| Kafka topics | `oms.{domain}.{entity}.{action}` | `oms.seating.booking.released` |
| Kafka consumer groups | `oms-{service}-{domain}` | `oms-attendance-remote` |
| Test classes | `{ClassName}Test` (unit), `{ClassName}IT` (integration) | `SeatBookingServiceTest` |

**Never** use abbreviations (`usr`, `svc`, `loc`). **Never** use vague names (`data`, `temp`, `obj`).

---

## Data Model Conventions

### Standard Fields (All Entities)

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;

@Column(nullable = false, updatable = false)
private Instant createdAt;

@Column(nullable = false)
private Instant updatedAt;

@Column(nullable = false)
private UUID createdBy;   // Resolved from SecurityContext
```

### Location Scoping — Mandatory

```java
@Column(nullable = false)
private UUID locationId;  // Present on every feature entity
```

Every repository query on a feature entity must filter by `locationId`. No exceptions.

```java
// ✅ Correct
List<SeatBooking> findByLocationIdAndDeletedAtIsNull(UUID locationId);

// ❌ Wrong — will leak cross-location data
List<SeatBooking> findAll();
```

### Soft Deletes

```java
private Instant deletedAt;  // NULL = active; non-null = deleted
```

All active-record queries use `WHERE deleted_at IS NULL`. This is enforced in the repository layer, not in service code.

---

## Security in Code

### Every Service Must Validate Internal JWT

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.decoder(internalJwtDecoder()))
    );
    return http.build();
}
```

### Role + Location Checks

```java
@PreAuthorize("@locationRoleChecker.hasRoleAtLocation(#locationId, 'MANAGER')")
public SeatBookingResponse bookForTeam(UUID locationId, BlockBookingRequest request) { ... }
```

### Parameterised SQL

```java
// ✅ Correct
@Query("SELECT a FROM AttendanceRecord a " +
       "WHERE a.userId = :userId " +
       "AND a.locationId = :locationId " +
       "AND a.deletedAt IS NULL")
List<AttendanceRecord> findActiveByUserAndLocation(
    @Param("userId") UUID userId,
    @Param("locationId") UUID locationId);
```

---

## Kafka Patterns

### Publishing (after transaction commit)

```java
@Transactional
public SeatBookingResponse book(SeatBookingRequest request) {
    SeatBooking booking = repository.save(mapToEntity(request));
    // Publish post-commit
    publisher.publish("oms.seating.booking.created", SeatBookingCreatedEvent.from(booking));
    auditPublisher.publish(AuditEvent.of("SEAT_BOOKED", "SeatBooking", booking.getId(), null, booking, currentUser));
    return mapToResponse(booking);
}
```

### Consuming (idempotent)

```java
@KafkaListener(topics = "oms.remote.request.approved", groupId = "oms-attendance-remote")
@Transactional
public void handleRemoteApproved(OmsEvent<RemoteApprovedPayload> event) {
    if (processedEventRepository.existsById(event.eventId())) {
        log.info("Duplicate event ignored: {}", event.eventId());
        return;
    }
    attendanceService.overlayRemoteStatus(event.payload().userId(), event.payload().dates(), event.locationId());
    processedEventRepository.save(new ProcessedEvent(event.eventId()));
}
```

---

## Error Handling

### Custom Exceptions

```java
public abstract class OmsException extends RuntimeException {
    private final String code;
    private final HttpStatus status;

    protected OmsException(String code, String message, HttpStatus status) {
        super(message);
        this.code = code;
        this.status = status;
    }
}

public class BookingWindowExceededException extends OmsException {
    public BookingWindowExceededException() {
        super("BOOKING_WINDOW_EXCEEDED",
              "Booking date exceeds the configured window for this location.",
              HttpStatus.BAD_REQUEST);
    }
}
```

### Global Handler (One Per Service)

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

### Response Envelope

```java
public record ApiResponse<T>(boolean success, T data, ErrorDetail error) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null);
    }
    public static ApiResponse<Void> error(String code, String message) {
        return new ApiResponse<>(false, null, new ErrorDetail(code, message));
    }
}
```

---

## API Documentation (Mandatory)

Every endpoint must be documented with Swagger annotations. No undocumented endpoint merges to `main`.

```java
@Operation(
    summary = "Book a hot-desk seat",
    description = "Books an available hot-desk for the specified date. Enforces the location booking window."
)
@ApiResponse(responseCode = "201", description = "Booking created")
@ApiResponse(responseCode = "400", description = "Validation error or booking window exceeded")
@ApiResponse(responseCode = "409", description = "Seat no longer available")
@PostMapping
public ResponseEntity<ApiResponse<SeatBookingResponse>> bookSeat(
        @Valid @RequestBody SeatBookingRequest request) { ... }
```

Swagger UI: `http://localhost:{port}/swagger-ui/index.html`

---

## Testing Strategy

| Level | Tool | What to Test |
|-------|------|-------------|
| Unit | JUnit 5 + Mockito | All service logic; Kafka consumer handlers; policy enforcement; job resolution logic |
| Integration | Spring Boot Test + TestContainers | All REST endpoints; repository queries with a real DB; Kafka producer/consumer pairs |
| Contract | Pact | Every consumer-provider REST relationship — prevents silent breaking API changes |
| Security | MockMvc + role simulation | Every endpoint against every role+location combination |
| Resilience | Resilience4j test mode | Circuit breaker, retry, timeout, fallback behaviour |
| Performance | k6 | Nightly sync job throughput; seat availability under concurrency |

### Pact Contract Testing Rule

When `seating-service` calls `user-service`:
1. `seating-service` defines the Pact contract (as **consumer**).
2. `user-service` verifies the contract in its own CI pipeline (as **provider**).
3. Neither service can deploy a breaking API change without the contract test failing in CI first.

### Scheduled Jobs — Testing Rules

All scheduled jobs must be unit-testable in isolation:
- Idempotent: re-running the job for the same date must not create duplicates.
- Bulkhead-isolated: job execution must not share the API request thread pool.
- Alerting: any job failure or partial failure must publish an alert event.

---

## Configuration Management (12-Factor)

All configuration is via **environment variables only**. No hardcoded values. No secrets in config files.

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

# Service-specific (seating-service example)
BOOKING_WINDOW_DAYS=14
CANCELLATION_CUTOFF_HOURS=2
NO_SHOW_RELEASE_CRON=0 10 * * *

# attendance-service
ATHENA_DATABASE=badge_events
ATHENA_OUTPUT_BUCKET=s3://oms-athena-results/
AWS_REGION=eu-west-1
BADGE_SYNC_CRON=0 2 * * *
SESSION_GAP_THRESHOLD_HOURS=6

# identity-service
SSO_JWKS_URI=https://sso.yourorg.com/.well-known/jwks.json
SSO_ISSUER_URI=https://sso.yourorg.com
SESSION_EXPIRY_SECONDS=3600
```

Three environments: `dev`, `staging`, `prod`. Only environment variables differ — no code changes between environments.

---

## Git Workflow

### Branch Strategy

```
main          ← Production-ready. Protected. PR + review + manual approval required.
staging       ← Staging environment. Auto-deployed on merge.
develop       ← Integration branch. PR + review required.
feature/<n>   ← Feature branches off develop
fix/<n>       ← Bug fix branches off develop
hotfix/<n>    ← Critical fixes off main (merged back to main + develop)
```

### Commit Message Format (Conventional Commits)

```
<type>(<scope>): <short summary — max 50 chars, imperative mood, no period>

<body — what and why, 72 chars per line>

<footer — Closes OMS-123>
```

**Types:** `feat` · `fix` · `chore` · `docs` · `refactor` · `style` · `perf` · `test` · `build` · `ci`

**Examples:**
```
feat(seating): implement no-show auto-release scheduled job

The daily no-show release job queries confirmed bookings with no
badge-in event by the configured cutoff time, marks them RELEASED,
creates a NoShowRecord, and publishes seat.booking.released to Kafka.

Closes OMS-198

---

feat(attendance): implement two-pass nightly resolution job

Pass 1: badge events from Athena → WorkSession → AttendanceRecord.
Pass 2: Kafka consumer for remote.request.approved and
ooo.request.approved overlays REMOTE/OOO statuses.

Closes OMS-142
```

### Definition of Done (Every Ticket)

- [ ] Code reviewed and merged to `main`
- [ ] Unit test coverage ≥ 80% on new code
- [ ] Integration tests passing
- [ ] Pact contract tests passing (if REST relationship involved)
- [ ] No hardcoded secrets or credentials
- [ ] Swagger annotations on all new endpoints
- [ ] Audit log entry wired for all state changes
- [ ] Role + location interceptor applied on all new endpoints
- [ ] CI pipeline green (lint → test → build)
- [ ] Acceptance criteria signed off by PO
