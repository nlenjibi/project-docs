# Security Architecture
## Office Management System (OMS)

**Version:** 3.0

**Principle: Never Trust. Always Verify.** Every service validates the caller's identity on every request, regardless of whether the call originates from the API Gateway, another internal service, or a scheduled job. Security is not a gateway concern alone — it is enforced at every layer.

---

## Security Layers (Five Layers)

```
Layer 1: Network
  - TLS 1.3 on all external HTTPS traffic (ALB terminates, re-encrypts to ECS tasks)
  - All ECS tasks in private VPC subnets — no public IPs
  - RDS instances in isolated private subnets — not reachable from the internet or developer machines directly
  - Amazon MQ in private subnet — AMQPS (TLS) only
  - Spring Cloud Gateway is the sole public-facing entry point

Layer 2: Identity (auth-service)
  - SSO only — no local username/password management in OMS
  - OIDC authorisation code flow — token verification entirely server-side
  - JWKS endpoint verification (token signature, expiry, audience, issuer)
  - HTTP-only, SameSite=Strict, Secure session cookies — JavaScript cannot read them (XSS-safe)
  - Sessions stored server-side in auth_db (spring-session-jdbc) — can be revoked instantly on logout

Layer 3: API Gateway (Spring Cloud Gateway)
  - Session cookie validated on every inbound request via auth-service /internal/validate
  - Invalid/expired session → 401 — request rejected before reaching any downstream service
  - X-User-Id, X-User-Roles, X-Location-Id headers injected from session context
  - X-Correlation-ID generated for every request (traceability)
  - Rate limiting (100 req/s per authenticated user) — prevents abuse

Layer 4: Zero Trust Service-to-Service
  - Internal JWT (HMAC-SHA256, short-lived) attached to every Feign call
  - Every receiving service validates the internal JWT before processing
  - No service is trusted based on its network position alone
  - Internal JWT contains: userId, roles, locationId, expiry, correlationId

Layer 5: Data
  - Parameterised SQL everywhere — no string concatenation, ever
  - Soft deletes — historical records preserved for audit traceability
  - Separate DB credentials per service — minimum privilege
  - audit-service DB user: INSERT only — no UPDATE, no DELETE at DB level
  - Secrets in AWS Secrets Manager — never in code, config files, or Docker images
  - All data at rest encrypted (RDS KMS, S3 SSE-S3 for audit archival)
```

---

## Authentication Flow (14 Steps)

```
1.  User opens OMS React SPA (no session cookie present)
2.  React detects missing/expired session → calls POST /api/v1/auth/login
3.  auth-service issues a state CSRF token and redirects to SSO provider
4.  User authenticates at the SSO provider (corporate IdP — no OMS involvement)
5.  SSO provider redirects back with OIDC authorisation code to
    GET /api/v1/auth/callback?code=...&state=...
6.  auth-service verifies the state token (CSRF protection)
7.  auth-service exchanges the code for OIDC ID token + access token (server-to-server)
8.  auth-service verifies the ID token:
      - Signature: fetched JWKS from SSO_JWKS_URI
      - Expiry: token must not be expired
      - Audience: must match OMS client ID
      - Issuer: must match SSO_ISSUER_URI
9.  auth-service resolves the SSO subject (sub) → OMS User record
10. auth-service loads the user's UserRole records (all locations, all roles)
11. auth-service creates a server-side session (spring-session-jdbc in auth_db)
     and embeds { userId, roles, locationId } in the session
12. auth-service issues an HTTP-only, SameSite=Strict, Secure session cookie
    Set-Cookie: SESSION=...; HttpOnly; SameSite=Strict; Secure; Path=/
13. React SPA sends the cookie on every subsequent request automatically (browser handles it)
14. Spring Cloud Gateway calls auth-service /internal/validate on every request;
    auth-service returns user context → gateway injects headers → routes to downstream
```

**Key security properties:**
- The React SPA never handles, stores, or reads a raw token at any step.
- Sessions can be revoked instantly by deleting the session row in `auth_db`.
- Session expiry is controlled server-side via `SESSION_EXPIRY_SECONDS`.
- Expired or invalid sessions return `401` — no internal detail in the response body.

**Why server-side sessions over stateless JWTs for the browser:**

| Property | Server-side Session (HTTP-only cookie) | Stateless JWT (localStorage/memory) |
|----------|----------------------------------------|--------------------------------------|
| XSS accessible | ❌ No — HttpOnly cookie | ✅ Yes — JavaScript-readable |
| Instant revocation | ✅ Yes — delete session row | ❌ No — valid until expiry |
| CSRF risk | Mitigated by SameSite=Strict | N/A (token-based) |
| Scalability | Session store in DB (spring-session-jdbc) | Stateless — scales without shared state |
| OMS decision | **Chosen** — security over scalability at this scale | Rejected — XSS exposure unacceptable |

---

## Zero Trust — Service-to-Service

All services communicate via Feign. Every Feign call must carry a short-lived internal JWT signed with a shared HMAC-SHA256 key (`INTERNAL_JWT_SECRET` from Secrets Manager).

```
service-A calls service-B (Feign)
  ↓
service-A attaches:
  Authorization: Bearer <internal-JWT>
  X-Correlation-ID: <trace-id>
service-B verifies:
  1. Internal JWT signature (HMAC-SHA256 with shared secret)
  2. Internal JWT expiry (30 seconds — short-lived)
  3. Internal JWT claims: userId, roles[], locationId
  4. Role check: does the acting user hold the required role?
  5. Location check: does the user hold it at the target location?
```

**Internal JWT structure:**

```json
{
  "sub": "user-uuid",
  "roles": ["MANAGER"],
  "locationId": "accra-uuid",
  "correlationId": "trace-uuid",
  "iss": "oms-auth-service",
  "iat": 1737201600,
  "exp": 1737201630
}
```

**Why HMAC-SHA256 (symmetric) instead of RS256 (asymmetric):**
- Internal JWT is an internal implementation detail between services in the same VPC.
- RS256 requires key pair management and a JWKS endpoint for internal services — unnecessary operational overhead.
- The shared secret rotated via Secrets Manager provides sufficient security for internal communication.
- If the secret is ever compromised, all services are restarted with a new secret simultaneously (ECS `update-service --force-new-deployment`).

**Spring Security configuration (every service):**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(AbstractHttpConfigurer::disable)  // Stateless — no CSRF
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health/**", "/actuator/info").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt.decoder(internalJwtDecoder()))
        )
        .build();
}

@Bean
public JwtDecoder internalJwtDecoder() {
    // HMAC-SHA256 — shared secret from Secrets Manager
    return NimbusJwtDecoder
        .withSecretKey(new SecretKeySpec(
            internalJwtSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"))
        .build();
}
```

---

## Role-Based Access Control (RBAC)

### Roles

| Role | Location Scope | Description |
|------|---------------|-------------|
| `EMPLOYEE` | Location-scoped | Own data only — bookings, attendance, notifications |
| `MANAGER` | Location-scoped | Team visibility, approval authority, block bookings |
| `HR` | Location-scoped | Full org visibility at assigned locations; policy management |
| `FACILITIES_ADMIN` | Location-scoped | Floor plan, stock management, assets, visitor check-in/out |
| `SUPER_ADMIN` | Cross-location (NULL location_id) | Full access — all locations, all data |

### Location Scoping Rule

A user may hold different roles at different locations. The role check and the location check are **always performed together** — they cannot be separated.

```
UserRole table — Alice's roles:
| user_id | role      | location_id  |
|---------|-----------|--------------|
| alice   | MANAGER   | accra-uuid   |  ← Manager in Accra only
| alice   | EMPLOYEE  | lagos-uuid   |  ← Employee in Lagos
| bob     | SUPER_ADMIN | NULL        |  ← Cross-location; passes all location checks
```

**SUPER_ADMIN handling:** `location_id = NULL` on the `UserRole` record is the ONLY cross-location exception. The `LocationRoleInterceptor` handles this case explicitly — it is not inferred or defaulted.

### LocationRoleInterceptor

Applied to every HTTP request on every protected endpoint:

```java
@Component
@RequiredArgsConstructor
public class LocationRoleInterceptor implements HandlerInterceptor {

    private final UserRoleRepository userRoleRepository;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {

        if (!(handler instanceof HandlerMethod method)) return true;

        RequiresRole annotation = method.getMethodAnnotation(RequiresRole.class);
        if (annotation == null) return true;  // Unannotated endpoints = permit all authenticated

        UUID userId = extractUserIdFromJwt(request);
        UUID locationId = extractLocationIdFromHeader(request);  // X-Location-Id from gateway
        Role requiredRole = annotation.value();

        // SUPER_ADMIN: NULL location_id bypasses all location checks
        boolean hasAccess = userRoleRepository.existsByUserIdAndRoleAndLocationIdOrNull(
            userId, requiredRole, locationId);

        if (!hasAccess) {
            response.setStatus(HttpStatus.FORBIDDEN.value());
            return false;
        }

        return true;
    }
}
```

**Custom annotation:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresRole {
    Role value();
}

// Usage on controller methods:
@RequiresRole(Role.FACILITIES_ADMIN)
@PutMapping("/locations/{locationId}/floor-plan")
public ResponseEntity<ApiResponse<FloorPlanResponse>> updateFloorPlan(...) { ... }
```

### Authorisation Enforcement Matrix

| Check | Enforcement Point | Tool |
|-------|-----------------|------|
| Session validity | Spring Cloud Gateway | SessionAuthFilter calling `/internal/validate` |
| Rate limiting | Spring Cloud Gateway | RequestRateLimiter |
| Internal JWT validity | Every service | Spring Security `oauth2ResourceServer` |
| Role check (coarse) | Spring Cloud Gateway | Path-level route guards |
| Role + location check (authoritative) | Each service | `LocationRoleInterceptor` |
| Data scoping (location filter) | Repository layer | `WHERE location_id = :locationId` on every query |

---

## Data Security

### Parameterised SQL — No Exceptions

String concatenation in SQL is prohibited everywhere and enforced in code review. SQL injection is prevented at the framework level (Spring Data JPA / JPQL), but explicit parameterisation is still required in all native queries.

```java
// ✅ CORRECT — parameterised JPQL
@Query("SELECT s FROM SeatBooking s " +
       "WHERE s.userId = :userId " +
       "AND s.locationId = :locationId " +
       "AND s.deletedAt IS NULL")
List<SeatBooking> findActive(
    @Param("userId") UUID userId,
    @Param("locationId") UUID locationId);

// ❌ FORBIDDEN — string concatenation (SQL injection vector)
String sql = "SELECT * FROM seat_bookings WHERE user_id = '" + userId + "'";
```

### Input Validation

Every inbound request body is validated via `@Valid` (Jakarta Bean Validation) before the service layer is invoked. Validation failures return `400 VALIDATION_ERROR` with the failing field name — no stack trace.

```java
@PostMapping
public ResponseEntity<ApiResponse<SeatBookingResponse>> bookSeat(
        @Valid @RequestBody SeatBookingRequest request) { ... }

// DTO with validation annotations
public class SeatBookingRequest {
    @NotNull
    private UUID seatId;

    @NotNull
    @Future(message = "bookingDate must be a future date")
    private LocalDate bookingDate;

    @NotNull
    private UUID locationId;
}
```

### Secrets Management

| Rule | Implementation |
|------|---------------|
| No secrets in code | Enforced via GitHub secret scanning + OWASP dependency scan in CI |
| No secrets in config files | All `application.yml` files reference `${ENV_VAR}` — never hardcoded values |
| No secrets in Docker images | ECS injects at runtime via Task Definition `secrets` block |
| No secrets in logs | Logback pattern never includes headers, JWT values, or DB passwords |
| Secret rotation | AWS Secrets Manager rotation Lambda (every 90 days); ECS tasks restarted after rotation |

### Database Privilege Matrix

| Service | Privileges on own tables | Privileges on other tables |
|---------|------------------------|---------------------------|
| All feature services | SELECT, INSERT, UPDATE, DELETE on own tables | NONE — physically impossible |
| `audit-service` | INSERT only on `audit_logs` | NONE |
| Any service | NONE on any other service's database | NONE |

---

## Audit Trail

Every state-changing operation publishes an `oms.audit.event` to RabbitMQ **before returning a response**. `audit-service` persists these records immutably.

**No REST endpoint can write to `audit_logs` directly.** The only write path is via the RabbitMQ consumer. This prevents bypassing the audit trail via a direct API call.

```java
// AuditEventPublisher — called from every service layer method that mutates state
@Component
@RequiredArgsConstructor
public class AuditEventPublisher {
    private final StreamBridge streamBridge;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publish(AuditDomainEvent domainEvent) {
        OmsEvent<AuditPayload> event = new OmsEvent<>(
            UUID.randomUUID(),
            "oms.audit.event",
            "1",
            CorrelationIdHolder.get(),
            Instant.now(),
            domainEvent.locationId(),
            new AuditPayload(
                domainEvent.actorId(),
                domainEvent.actorRole(),
                domainEvent.action(),
                domainEvent.entityType(),
                domainEvent.entityId(),
                domainEvent.previousState(),
                domainEvent.newState()
            )
        );

        streamBridge.send("publishAuditEvent-out-0", event);
    }
}
```

Audit event structure:
```json
{
  "eventId": "uuid",
  "eventType": "oms.audit.event",
  "version": "1",
  "correlationId": "trace-uuid",
  "occurredAt": "2026-01-18T14:30:00Z",
  "locationId": "uuid",
  "payload": {
    "actorId": "user-uuid",
    "actorRole": "MANAGER",
    "action": "SEAT_BOOKED",
    "entityType": "SeatBooking",
    "entityId": "uuid",
    "previousState": null,
    "newState": { "seatId": "...", "bookingDate": "2026-01-20", "status": "CONFIRMED" }
  }
}
```

---

## Error Response Rules

External error responses **never** expose:
- Stack traces
- Internal class names or package paths
- Database error messages or constraint names
- Token or session contents
- Internal service names or ports

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OmsException.class)
    public ResponseEntity<ApiResponse<Void>> handleDomainException(OmsException ex) {
        // Domain exceptions have safe, user-readable messages
        return ResponseEntity.status(ex.getStatus())
            .body(ApiResponse.error(ex.getCode(), ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        // Full stack trace goes to structured logs only — NEVER to the response
        log.error("Unhandled exception: correlationId={}",
            CorrelationIdHolder.get(), ex);
        return ResponseEntity.status(500)
            .body(ApiResponse.error("INTERNAL_ERROR",
                "An unexpected error occurred. Reference: " + CorrelationIdHolder.get()));
    }
}
```

The `correlationId` is safe to include in the error response — it lets operations team trace the failure in Jaeger/CloudWatch without exposing internal implementation details.

---

## React Frontend Security

The React SPA follows these security rules:

| Rule | Implementation |
|------|---------------|
| No token storage in localStorage | Sessions use HTTP-only cookies — browser handles them transparently |
| No token storage in memory | No raw token is ever returned to the React app |
| No direct API calls bypassing the gateway | All API calls go to `/api/v1/*` (ALB → Gateway) |
| CSRF protection | `SameSite=Strict` on session cookie prevents cross-site request forgery |
| CSP header | Content-Security-Policy set at ALB level; no inline scripts, no `eval()` |
| Sensitive data in state | No raw tokens, passwords, or JWT payloads stored in React state or Redux |
| API error handling | 401 responses → redirect to login; 403 → show "Access Denied"; 5xx → show error notification |
| Dependency scanning | `npm audit` runs in CI; critical vulnerabilities block the build |

**React login flow:**

```typescript
// React SPA — no token handling, no localStorage
const login = async () => {
  // Redirect to auth-service; auth-service sets the cookie after SSO
  window.location.href = '/api/v1/auth/login';
};

// React SPA — check session via /api/v1/auth/me on app load
const checkSession = async (): Promise<User | null> => {
  try {
    const response = await fetch('/api/v1/auth/me', {
      credentials: 'include'  // Sends the HTTP-only cookie automatically
    });
    if (response.status === 401) return null;
    const data = await response.json();
    return data.data;
  } catch {
    return null;
  }
};
```

---

## OWASP Top 10 Mitigation

| Risk | OMS Mitigation |
|------|----------------|
| **A01: Broken Access Control** | Role+location checks on every endpoint via `LocationRoleInterceptor`; SUPER_ADMIN NULL-location handled explicitly; repository-level `locationId` filter on all queries |
| **A02: Cryptographic Failures** | TLS 1.3 on all external traffic; AMQPS for Amazon MQ; RDS SSL required; RDS encryption at rest (KMS); HTTP-only cookies; secrets in Secrets Manager |
| **A03: Injection** | Parameterised SQL in all repositories (enforced in code review + CI static analysis); `@Valid` DTO validation on all inputs; no dynamic query construction |
| **A04: Insecure Design** | ADRs document every security decision; Zero Trust mandated at design level; security review gate before production deployments |
| **A05: Security Misconfiguration** | No hardcoded secrets; secrets via Secrets Manager; OWASP dependency scan in CI; no debug endpoints exposed in production; Actuator paths whitelisted only |
| **A06: Vulnerable Components** | OWASP dependency-check in CI pipeline blocks merge on critical CVEs; `npm audit` for React dependencies |
| **A07: Identification and Auth Failures** | SSO only — no local password management; OIDC token verified server-side via JWKS; HTTP-only session cookies; instant session revocation; 401 on expired sessions |
| **A08: Software and Data Integrity** | Pact consumer-driven contract tests prevent silent breaking API changes; Docker image tags pinned to Git SHA; no `latest` tag in production |
| **A09: Logging and Monitoring Failures** | Immutable `audit-service` log (INSERT-only DB user); centralised structured JSON logging; PagerDuty critical alerts on auth failures and 5xx error rates |
| **A10: SSRF** | No user-controllable URLs in outbound requests; Athena endpoint, Arms HR endpoint, and SSO JWKS URI are environment-variable-configured only; no URL fetch from user input |

---

## Security Checklist (Per Service — Before Production)

- [ ] Internal JWT validation wired via `SecurityFilterChain` + `internalJwtDecoder()`
- [ ] `LocationRoleInterceptor` registered via `WebMvcConfigurer`
- [ ] `@RequiresRole` annotation on every protected endpoint
- [ ] All repository queries include `locationId` filter
- [ ] All SQL uses parameterised queries — no string concatenation
- [ ] `@Valid` on all `@RequestBody` parameters
- [ ] `GlobalExceptionHandler` configured — no stack trace in responses
- [ ] Audit event published on every state-changing operation
- [ ] No secrets in `application.yml` — all `${ENV_VAR}` references
- [ ] No passwords, tokens, or PII in logs
- [ ] OWASP dependency scan passing in CI
- [ ] Actuator endpoints restricted to health, info, prometheus only
