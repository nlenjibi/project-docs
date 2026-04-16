# ATT-P01 — SSO / OAuth 2.0 Integration (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P01 |
| **Epic** | Platform / Auth — PI-OBJ-04 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, auth, security |
| **Perspective** | Backend |

---

## Story

As the platform, I want to integrate with the organisation's SSO provider via OAuth 2.0 / OIDC so that all authentication is centralised, server-side only, and the frontend never handles raw tokens.

---

## Background (Backend Context)

This is the foundational auth ticket. Nothing else is testable end-to-end until this is in place. The Spring Boot backend acts as the OAuth 2.0 resource server. Token verification happens against the SSO provider's JWKS endpoint — the frontend (React) never inspects or stores the raw JWT.

**API Gateway notes:**
- **Dev:** Spring Cloud Gateway handles routing and will propagate the session context header to downstream services.
- **Prod:** AWS API Gateway or Kong will handle TLS termination and route to the Spring Boot app; auth validation logic remains server-side.

---

## Acceptance Criteria (Backend)

- [ ] `spring-boot-starter-oauth2-resource-server` added; JWKS URI configured via environment variable (`SSO_JWKS_URI`) — no hardcoded values.
- [ ] On a valid OIDC token, the backend extracts `sub` (SSO user ID), resolves it to an OMS `User` record, and populates a `SecurityContext`.
- [ ] An HTTP-only, `SameSite=Strict` session cookie is issued to the React client after successful token exchange; the raw JWT is never forwarded to the frontend.
- [ ] `POST /api/v1/auth/login` — redirects to SSO provider.
- [ ] `GET /api/v1/auth/callback` — exchanges OIDC code for token; verifies against JWKS; issues session.
- [ ] `POST /api/v1/auth/logout` — invalidates server-side session; clears cookie.
- [ ] `GET /api/v1/auth/me` — returns `{ userId, email, roles[] }` from session context; no raw token data.
- [ ] `GET /api/v1/auth/validate` — internal endpoint called by Spring Cloud Gateway (dev) or API Gateway (prod) on every inbound request; returns 200 with user context or 401.
- [ ] Session expiry is configurable via `SESSION_EXPIRY_SECONDS` environment variable.
- [ ] Expired or invalid tokens return HTTP 401 with `{ "success": false, "error": { "code": "UNAUTHORIZED", "message": "Session invalid or expired." } }` — no stack trace exposed.
- [ ] All endpoints return the standard `ApiResponse<T>` envelope.
- [ ] Unit tests cover: valid token → session created, expired token → 401, missing token → 401, logout → session cleared.
- [ ] Integration test covers the full callback → session → validate flow using a stubbed SSO provider.

---

## Implementation Notes

```
Controller  →  AuthService  →  OAuthTokenVerifier (JWKS)
                           →  UserRepository (resolve SSO sub → User)
                           →  SessionStore
```

- Use `spring-session-jdbc` for server-side session storage — keeps the app stateless between restarts.
- `SecurityContextHolder` is populated via a `OncePerRequestFilter` that reads the session cookie and rehydrates the `SecurityContext` on each request.
- Never log the raw token, even at DEBUG level.
- Configuration class must validate that `SSO_JWKS_URI` is non-null at startup; fail fast with a clear error if missing.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80% on new code
- [ ] No hardcoded secrets or credentials
- [ ] Swagger (`@Operation`, `@ApiResponse`) annotations on all endpoints
- [ ] Audit log entry written for login and logout events
- [ ] Jenkins pipeline green (lint → test → build)
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- SSO provider must deliver OAuth credentials and OIDC discovery document before this ticket can be completed. **Escalate to ART board on Sprint 1 Day 1 if not received.**
- `User` entity and `UserRole` entity must be seeded in the database.
- ATT-P02 (role + location interceptor) depends on this ticket being complete.
