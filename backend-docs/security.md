# Security Architecture
## Office Management System (OMS)

**Principle: Never trust. Always verify.** Every service validates the caller's identity on every request, regardless of whether the call originates from the API Gateway or another internal service.

---

## Security Layers (Five Layers)

```
Layer 1: Network
  - TLS 1.3 on all external HTTPS traffic
  - All services in private VPC subnets
  - Database instances not reachable from the public internet
  - API Gateway is the sole public-facing entry point

Layer 2: Identity (identity-service)
  - SSO only; no local password management in OMS
  - OIDC token verified server-side via JWKS endpoint
  - HTTP-only, SameSite=Strict session cookies — JavaScript cannot access them

Layer 3: Service Mesh (Zero Trust)
  - Mutual TLS (mTLS) on every service-to-service call (Istio / AWS App Mesh)
  - Short-lived internal JWT verified by every receiving service on every call
  - No service trusted based on network position alone

Layer 4: Authorisation (per service)
  - Role check: does the user hold the required role?
  - Location check: does the user hold it at the target location?
  - Both checks enforced at the service layer — API Gateway does coarse filtering only

Layer 5: Data
  - Parameterised SQL only — no string concatenation anywhere, ever
  - Soft deletes — historical records never permanently destroyed
  - Separate DB credentials per service; minimum privilege
  - Audit DB user: INSERT only — no UPDATE, no DELETE at DB level
  - Secrets via AWS Secrets Manager — never in code or config files
```

---

## Authentication Flow

```
1.  User opens OMS Angular SPA
2.  SPA has no session → redirects to identity-service /api/v1/auth/login
3.  identity-service redirects to the SSO provider
4.  User authenticates at the SSO provider
5.  SSO provider redirects with OIDC authorisation code to
    identity-service /api/v1/auth/callback
6.  identity-service exchanges the code for OIDC tokens
7.  identity-service verifies token:
      - Signature via JWKS endpoint (SSO_JWKS_URI)
      - Expiry
      - Audience / issuer claims
8.  identity-service resolves the SSO subject (sub) → OMS User record
9.  identity-service issues an HTTP-only, SameSite=Strict session cookie
10. SPA sends the session cookie on every subsequent request
11. API Gateway calls identity-service /api/v1/auth/validate on every inbound request
12. identity-service resolves user identity, roles, and location context from the session
13. API Gateway injects X-User-Id and X-Location-Id headers; routes to the downstream service
14. Downstream service validates the internal JWT (Zero Trust) and enforces role+location check
```

**Key rules:**
- The frontend never handles, stores, or inspects raw tokens.
- Sessions are stored server-side using `spring-session-jdbc` in `identity_db`.
- Session expiry is configurable via `SESSION_EXPIRY_SECONDS`.
- Expired or invalid sessions return `401 UNAUTHORIZED` — no stack trace in the response.

---

## Zero Trust — Service-to-Service

```
service-A calls service-B
  ↓
service-A presents:
  1. Client certificate (mTLS) — proves it is a known service in the mesh
  2. Internal JWT — signed by the internal key managed by identity-service
service-B verifies:
  1. mTLS — is this certificate known to our service mesh?
  2. Internal JWT — is it signed by our internal key?
  3. Role check — does the acting user hold the required role?
  4. Location check — does the user hold that role at the target location?
```

**Spring Security configuration (every service):**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.decoder(internalJwtDecoder()))
    );
    return http.build();
}
```

---

## Role-Based Access Control (RBAC)

### Roles

| Role | Scope | Description |
|------|-------|-------------|
| `EMPLOYEE` | Location-scoped | Standard access — own data only |
| `MANAGER` | Location-scoped | Team visibility and approval authority |
| `HR` | Location-scoped | Org-wide visibility at assigned locations |
| `FACILITIES_ADMIN` | Location-scoped | Operational controls (floor plan, stock, assets) |
| `SUPER_ADMIN` | Cross-location (NULL) | Governance — all locations, all data |

### Location Scoping

A user may hold different roles at different locations. The role check and the location check are inseparable.

```java
// UserRole table: user may have multiple rows
| user_id | role    | location_id |
|---------|---------|-------------|
| alice   | MANAGER | accra-uuid  |   ← Manager in Accra only
| alice   | EMPLOYEE| lagos-uuid  |   ← Employee in Lagos
| bob     | SUPER_ADMIN | NULL    |   ← Cross-location access
```

**Super Admin handling:** `location_id = NULL` on the `UserRole` record means the user passes all location checks globally. This exception must be handled explicitly in the interceptor.

### LocationRoleInterceptor

A global `HandlerInterceptor` registered via `WebMvcConfigurer`. Applied on every request.

```java
@Component
public class LocationRoleInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        // 1. Extract userId + locationId from SecurityContext (populated by session filter)
        // 2. Resolve required role from @RequiresRole annotation on the handler
        // 3. Check UserRole records (location-scoped)
        // 4. Super Admin bypass: if UserRole.location_id IS NULL, pass all location checks
        // 5. Reject with 403 if check fails
    }
}
```

**Custom annotation:**

```java
@RequiresRole(Role.MANAGER)
public SeatBookingResponse bookForTeam(UUID locationId, BlockBookingRequest request) { ... }
```

**Super Admin check:**

```java
@PreAuthorize("@locationRoleChecker.hasRoleAtLocation(#locationId, 'MANAGER')")
```

**Public endpoints** (e.g., `/actuator/health`) are whitelisted and bypass the interceptor.

---

## Authorisation Enforcement Matrix

| Check | Enforcement Point |
|-------|-----------------|
| Session validity | API Gateway → `identity-service /api/v1/auth/validate` |
| Role check (coarse) | API Gateway (path prefix level) |
| Role + location check (authoritative) | `LocationRoleInterceptor` within each service |
| Internal JWT validity | Spring Security `@oauth2ResourceServer` in every service |
| Data scoping (location filter) | Repository layer — every query includes `locationId` |

---

## Data Security Rules

### Parameterised SQL — No Exceptions

```java
// ✅ Correct — Spring Data JPA parameterised query
@Query("SELECT s FROM SeatBooking s " +
       "WHERE s.userId = :userId " +
       "AND s.locationId = :locationId " +
       "AND s.deletedAt IS NULL")
List<SeatBooking> findActiveByUserAndLocation(
    @Param("userId") UUID userId,
    @Param("locationId") UUID locationId);

// ❌ Forbidden — never do this
String query = "SELECT * FROM seat_bookings WHERE user_id = '" + userId + "'";
```

### Secrets Management

- All secrets injected via environment variables at runtime.
- AWS Secrets Manager is the secrets source in staging and production.
- No secrets in code, config files, or Docker images.
- Secrets are never logged, even at DEBUG level.

### Database Privileges (Minimum Principle)

| Service | Privileges |
|---------|-----------|
| All feature services | SELECT, INSERT, UPDATE, DELETE on their own tables |
| audit-service | INSERT only — no UPDATE, no DELETE on `audit_logs` |
| No service | Any access to another service's database |

### Logging Rules

```java
// ❌ Never log
log.debug("Token: {}", rawToken);
log.info("Password: {}", password);
log.error("User PII: {}", personalData);

// ✅ Safe logging
log.warn("Authentication failed for userId={}", userId);
log.info("Session created for userId={}", userId);
```

---

## OWASP Top 10 Mitigation

| Risk | Mitigation |
|------|-----------|
| **A01: Broken Access Control** | Role+location checks in every service; `@PreAuthorize` on every sensitive endpoint; `SUPER_ADMIN` NULL-location handled explicitly |
| **A02: Cryptographic Failures** | TLS 1.3 on external traffic; mTLS on internal; RDS encryption at rest; HTTP-only cookies |
| **A03: Injection** | Parameterised SQL in all repositories enforced in code review; `@Valid` DTO validation on all inputs |
| **A04: Insecure Design** | ADRs document every security decision; Zero Trust mandated; security review gate before production |
| **A05: Security Misconfiguration** | No hardcoded secrets; secrets via Secrets Manager; CI scan fails on detected secrets |
| **A07: Identification and Auth Failures** | SSO only; no local passwords; server-side OIDC token verification; HTTP-only sessions |
| **A08: Software and Data Integrity** | Pact contract tests prevent silent API changes; OWASP dependency scan in CI pipeline |
| **A09: Logging and Monitoring Failures** | Immutable `audit-service` log; centralised structured logging; PagerDuty alerts on critical failures |
| **A10: SSRF** | No user-controllable URLs in outbound requests; Athena and Arms endpoints are environment-configured only |

---

## Input Validation

All inbound request bodies are validated via Spring's `@Valid` before reaching the service layer.

```java
@PostMapping
public ResponseEntity<ApiResponse<SeatBookingResponse>> bookSeat(
        @Valid @RequestBody SeatBookingRequest request) { ... }
```

Validation failures return `400 VALIDATION_ERROR` with the failing field name and message — no stack trace.

---

## Error Response Rules

External error responses never expose:
- Stack traces
- Internal class names
- Database error messages
- Token or session contents

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
        log.error("Unhandled exception", ex);   // Full trace goes to logs only
        return ResponseEntity.status(500)
            .body(ApiResponse.error("INTERNAL_ERROR", "An unexpected error occurred."));
    }
}
```

---

## Audit Trail

Every state-changing operation publishes an `oms.audit.event` to Kafka / SQS before returning. The `audit-service` persists these records immutably. No API endpoint can write or delete audit records — only the Kafka/SQS consumer has that capability.

```json
{
  "eventId": "uuid-v4",
  "actorId": "user-uuid",
  "actorRole": "MANAGER",
  "action": "SEAT_BOOKED",
  "entityType": "SeatBooking",
  "entityId": "entity-uuid",
  "locationId": "location-uuid",
  "previousState": null,
  "newState": { "seatId": "...", "date": "2026-03-02", "status": "CONFIRMED" },
  "occurredAt": "2026-03-01T09:00:00Z",
  "correlationId": "trace-uuid"
}
```
