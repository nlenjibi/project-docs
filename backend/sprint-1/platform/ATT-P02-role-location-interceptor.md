# ATT-P02 — Role + Location Interceptor (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P02 |
| **Epic** | Platform / Auth — PI-OBJ-04 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, auth, security, rbac |
| **Perspective** | Backend |

---

## Story

As the platform, I want a global interceptor that enforces the per-location role model on every API endpoint so that no request is processed without a verified role and location context.

---

## Background (Backend Context)

The OMS uses a **role-per-location** model — a user may be a Manager in Accra but only an Employee in Lagos. This interceptor enforces both the role check AND the location check on every protected endpoint. It runs as a `HandlerInterceptor` (or `@PreAuthorize` via AOP) applied globally — not per-controller. No endpoint should be manually checking roles; all role/location logic lives here.

**API Gateway notes:**
- **All environments:** AWS API Gateway Lambda Authorizer validates the session cookie by calling `auth-service /api/v1/auth/validate`, generates an internal JWT with user context (userId, roles[], locationIds[]), and injects `X-User-Id`, `X-User-Roles`, `X-Location-Ids` request context headers before routing to the service. The interceptor reads these headers — it does not re-verify the token or call auth-service again.

---

## Acceptance Criteria (Backend)

- [ ] A `LocationRoleInterceptor` is registered as a global `HandlerInterceptor` via `WebMvcConfigurer`.
- [ ] On every request, the interceptor extracts `userId` and `locationId` from the request context (populated by the gateway header or session filter).
- [ ] The interceptor fetches the user's `UserRole` records and verifies the user holds the required role at the specified location before the controller method executes.
- [ ] Super Admin check: if `UserRole.location_id` is `NULL`, the user passes all location checks globally.
- [ ] Requests missing a valid role+location context return HTTP 403 with `{ "success": false, "error": { "code": "FORBIDDEN", "message": "Insufficient role or location access." } }`.
- [ ] Endpoints can be annotated with `@RequiresRole(Role.MANAGER)` (custom annotation) to declare the minimum role required; the interceptor reads this.
- [ ] Public endpoints (e.g. `/actuator/health`) are whitelisted and bypass the interceptor.
- [ ] The interceptor adds no more than 5ms latency on cache-warm role lookups (UserRole records cached in the session context after first resolution).
- [ ] Unit tests: Manager in Accra cannot access Lagos data, Employee cannot access Manager-only endpoint, Super Admin passes all location checks.
- [ ] Integration test: a full request with a stubbed user context hits a protected endpoint and is correctly allowed or rejected.

---

## Implementation Notes

```java
@Component
public class LocationRoleInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        // 1. Extract userId + locationId from SecurityContext
        // 2. Resolve required role from @RequiresRole annotation on handler
        // 3. Check UserRole records (location-scoped)
        // 4. Super Admin bypass if location_id IS NULL
        // 5. Reject with 403 if check fails
    }
}
```

- UserRole resolution should hit a local cache (populated at session creation from `user-service` data) to avoid a DB round-trip on every request.
- The `locationId` context must come from the **request** (URL path variable or header), not from user preferences — the user's current action determines which location is in scope.
- All access denials must be written to the AuditLog (ATT-P03 dependency).

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80% on new code
- [ ] No hardcoded role or location checks in any controller
- [ ] Interceptor applied and verified on all existing ATT-P01 endpoints
- [ ] Jenkins pipeline green
- [ ] Audit log entry confirmed for all 403 rejections
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01** must be complete (session context must be populated before interceptor can read it).
- `UserRole` table seeded with test data for integration tests.
