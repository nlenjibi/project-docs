# Security Architecture
## Office Management System (OMS)

**Version:** 4.0 | Model: Zero Trust | Auth: OAuth 2.0 / OIDC via SSO | Edge: AWS API Gateway + Lambda Authorizer

---

## 1. Security Layers

```
Layer 1 — Network
  AWS VPC private subnets · Security groups · TLS 1.3 on all external traffic

Layer 2 — Edge (API Gateway)
  Session cookie validation on every request
  Rate limiting (100 req/s per user, 150 burst)
  CORS enforcement per environment

Layer 3 — Identity (auth-service)
  SSO-only authentication (OAuth 2.0 / OIDC)
  Server-side OIDC token verification via JWKS
  HTTP-only SameSite=Strict session cookies
  Session revocation on logout

Layer 4 — Service (Zero Trust)
  Internal JWT verified by every service on every inbound request
  Role check: does this user hold the required role?
  Location check: does this user hold it at the target location?
  No service trusted based on network position alone

Layer 5 — Data
  Parameterised SQL only — no string concatenation ever
  Separate DB user per service with minimum privilege
  audit_db user: INSERT only (UPDATE and DELETE not granted at DB level)
  Secrets via AWS Secrets Manager — never in code or config
  All data in transit: HTTPS / TLS 1.3
  All data at rest: RDS encryption enabled (AWS KMS)
```

---

## 2. Authentication Flow

```
1. User opens React SPA → clicks Login
2. SPA redirects to SSO provider (OAuth 2.0 Authorization Code flow)
3. User authenticates at SSO provider
4. SSO redirects to auth-service /api/v1/auth/callback with authorization code
5. auth-service exchanges code for OIDC token at SSO /token endpoint
6. auth-service verifies:
   a. Token signature via JWKS (SSO public key)
   b. Token expiry (exp claim)
   c. Audience (aud claim matches OMS client_id)
   d. Issuer (iss claim matches SSO_ISSUER_URI)
7. auth-service resolves user roles from local DB (or creates user if first login)
8. auth-service creates session record in auth_db
9. auth-service returns HTTP-only SameSite=Strict session cookie to SPA
10. SPA sends session cookie on every subsequent request (automatic by browser)

Per request:
11. SPA request → CloudFront → API Gateway
12. API Gateway invokes Lambda Authorizer (Python 3.12)
13. Lambda Authorizer extracts SESSION cookie from request headers
14. Lambda Authorizer calls auth-service /api/v1/auth/validate (HTTP GET, X-Session-Token header)
15. auth-service returns UserContext { userId, roles[], locationIds[] } or 401
16. Lambda Authorizer generates internal JWT (HMAC-SHA256, 300s TTL) with user context
17. Lambda Authorizer returns IAM ALLOW policy + context { userId, roles, locationIds, correlationId, internalJwt }
18. API Gateway caches authorizer result for 300s (same session token → no Lambda invocation)
19. API Gateway injects context as request headers; forwards internal JWT as Authorization: Bearer
20. Service validates internal JWT (Spring Security / FastAPI python-jose)
21. Service performs role + location check
```

---

## 3. Internal JWT Structure

```json
{
  "sub": "user-uuid",
  "roles": ["MANAGER"],
  "locationIds": ["location-uuid-1", "location-uuid-2"],
  "correlationId": "trace-uuid",
  "iss": "oms-gateway",
  "iat": 1713340800,
  "exp": 1713341100
}
```

- Signed with HMAC-SHA256 using `INTERNAL_JWT_SECRET` (stored in Secrets Manager)
- TTL: 300 seconds (long enough for a request-response cycle, short enough to limit replay)
- Every service validates signature, expiry, and issuer on every inbound request

---

## 4. Zero Trust Service-to-Service

```java
// Every service — Spring Security configuration
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())          // No CSRF for stateless internal API
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.decoder(internalJwtDecoder()))
            );
        return http.build();
    }

    @Bean
    public JwtDecoder internalJwtDecoder() {
        return NimbusJwtDecoder
            .withSecretKey(new SecretKeySpec(
                internalJwtSecret.getBytes(StandardCharsets.UTF_8),
                "HmacSHA256"
            ))
            .build();
    }
}
```

```python
# FastAPI services — internal JWT validation
from jose import jwt, JWTError
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

async def get_current_user(token: str = Depends(bearer)) -> UserContext:
    try:
        payload = jwt.decode(token.credentials, settings.INTERNAL_JWT_SECRET, algorithms=["HS256"])
        return UserContext(
            user_id=UUID(payload["sub"]),
            roles=[Role(r) for r in payload["roles"]],
            location_ids=[UUID(lid) for lid in payload.get("locationIds", [])],
            correlation_id=payload.get("correlationId"),
        )
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid internal JWT")
```

---

## 5. Role-Based Access Control (RBAC)

Roles are **location-scoped** — a user may hold different roles at different offices.

| Role | Scope | Capabilities |
|---|---|---|
| `EMPLOYEE` | Own data at their location | View own attendance, book seats, submit requests |
| `MANAGER` | Direct reports at their location | All Employee + team visibility + approve requests |
| `HR` | All employees at their location | All Manager + org-wide reports + employee lifecycle |
| `FACILITIES_ADMIN` | Physical assets at their location | Space config, supply/asset management, visitor reception |
| `SUPER_ADMIN` | All locations | Full system configuration, cross-location access |

**Super Admin exception:** `UserRole` record has `location_id = NULL` — interpreted as cross-location access everywhere. The only cross-location exception in the system.

### Enforcement Pattern

```java
// In every service — @PreAuthorize on controllers
@GetMapping("/api/v1/attendance/team/{managerId}")
@PreAuthorize("hasAnyRole('MANAGER', 'HR', 'SUPER_ADMIN')")
public ResponseEntity<Page<AttendanceRecordDto>> getTeamAttendance(
    @PathVariable UUID managerId,
    @RequestHeader("X-Location-Id") UUID locationId,
    @AuthenticationPrincipal Jwt jwt
) {
    // Service-layer location check — this is the authoritative check
    authorisationService.requireRoleAtLocation(jwt, Role.MANAGER, locationId);
    return ResponseEntity.ok(attendanceService.getTeamAttendance(managerId, locationId));
}
```

The Lambda Authorizer applies coarse filtering (is the user authenticated?). Each service performs the **authoritative** role + location check.

---

## 6. Data Security

### SQL Injection Prevention

Parameterised queries are mandatory. No string concatenation. Enforced in code review.

```java
// FORBIDDEN — never do this
String query = "SELECT * FROM users WHERE email = '" + email + "'";

// REQUIRED — always do this
@Query("SELECT u FROM User u WHERE u.email = :email AND u.deletedAt IS NULL")
Optional<User> findByEmail(@Param("email") String email);
```

### Secrets Management

```
Secrets stored in: AWS Secrets Manager
Injected as:       Environment variables in ECS Task Definitions
Never stored in:   Source code, Docker images, YAML files, git history

Secrets per service:
  DB_PASSWORD            → oms/{service}/db-password
  INTERNAL_JWT_SECRET    → oms/internal-jwt-secret  (Lambda Authorizer fetches at cold start, not ECS env var)
  SSO_CLIENT_SECRET      → oms/sso/client-secret  (auth-service only)
  EMAIL_API_KEY          → oms/email/api-key       (notification-service only)
```

### Soft Deletes

Entities are never permanently deleted — `deleted_at` timestamp is set instead. This ensures:
- Data is recoverable if deleted accidentally
- Audit trail is complete — the deletion action itself is logged
- No broken foreign key references in cross-service IDs

### Audit Database

The `audit_db` PostgreSQL user is granted only INSERT privilege:
```sql
-- Applied to audit_app database user:
GRANT INSERT ON audit_logs TO audit_app;
-- No UPDATE, no DELETE — enforced at database level, not just application level
```

---

## 7. Network Security

```
Public internet  →  Amazon CloudFront (port 443)
                         ↓ /api/**
                    AWS API Gateway HTTP API
                    + Lambda Authorizer
                    + VPC Link
                         ↓
                   ECS Services (private subnets)
                         ↓
                  RDS (private subnets)
                  No public access to any of these

VPC Flow Logs: enabled (stored in S3, retained 90 days)
AWS WAF: attached to API Gateway (OWASP Core Rule Set enabled)
```

---

## 8. OWASP Top 10 Mitigation

| Risk | OMS Mitigation |
|---|---|
| A01 Broken Access Control | Role + location check in every service; Lambda Authorizer coarse filter (authenticated?) |
| A02 Cryptographic Failures | TLS 1.3 in transit; RDS encryption at rest (KMS); HTTP-only cookies; no plaintext secrets |
| A03 Injection | Parameterised queries mandatory; `@Valid` DTO validation on all inputs; Checkstyle enforces in CI |
| A04 Insecure Design | ADRs document every security decision; Zero Trust mandated; threat modelling in architecture review |
| A05 Security Misconfiguration | Debug endpoints disabled in prod; verbose error messages suppressed (`server.error.include-message=never`); OWASP dependency scan in CI |
| A06 Vulnerable Components | OWASP Dependency Check in CI pipeline — fails on CVSS >= 7 |
| A07 Identification Failures | SSO-only; no local passwords; server-side token verification; immediate session revocation on logout |
| A08 Software Integrity | Docker images signed; ECR image scanning enabled; CI pipeline verified before deployment |
| A09 Security Logging | Immutable audit-service log; structured JSON logs with actor, action, entity on every mutation |
| A10 Server-Side Request Forgery | No user-controlled URLs in backend; all external integrations (Arms, Athena, email) use hardcoded endpoints from Secrets Manager |

---

## 9. Session Security

| Property | Value | Reason |
|---|---|---|
| Cookie flags | `HttpOnly`, `Secure`, `SameSite=Strict` | Prevent XSS token theft, CSRF |
| Session storage | Server-side in `auth_db` (spring-session-jdbc) | Immediate revocation possible |
| Session TTL | 3,600 seconds (configurable) | Limit stale session window |
| Session ID entropy | 128-bit random (Spring Session default) | Prevent enumeration |
| Session on login | New session ID issued on every login | Prevent session fixation |
| Logout | Session deleted server-side immediately | Token expiry cannot be relied on |

---

## 10. React Frontend Security

| Concern | Mitigation |
|---|---|
| Token storage | No tokens in localStorage or sessionStorage — HTTP-only cookie managed by browser |
| XSS | React JSX escapes output by default; `dangerouslySetInnerHTML` is forbidden without security review |
| CSRF | `SameSite=Strict` cookie prevents cross-site form submissions |
| Content Security Policy | `Content-Security-Policy` header configured at CloudFront / API Gateway |
| Sensitive data in logs | React error boundaries catch and log errors — no user data logged to console in production |
| Dependencies | `npm audit` run in CI pipeline; `dependabot` configured for automatic PRs |
