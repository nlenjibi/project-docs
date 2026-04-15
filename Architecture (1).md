# SYSTEM ARCHITECTURE: Office Management System (OMS)
**Version:** 2.0 — Microservices Edition  
**Status:** Draft  
**Last Updated:** March 2026  

---

## EXECUTIVE SUMMARY

The Office Management System (OMS) is a cloud-native, microservices platform serving 2–5 office locations and hundreds of employees. The system is decomposed into **11 independently deployable services**, each owning exactly one business capability and its own PostgreSQL database. All external traffic enters through an API Gateway. Services communicate synchronously via REST with Resilience4j circuit breakers for user-facing queries, and asynchronously via Apache Kafka for state changes, cross-service workflows, audit logging, and notification dispatch.

The architecture is designed around three unbreakable rules: **no shared databases**, **no direct calls to notification or audit services** (Kafka only), and **Zero Trust security** (every service validates every caller). Every design decision is traceable to a documented Architecture Decision Record (AD-001 through AD-026 in `doch.md`).

---

## REQUIREMENTS SUMMARY

### Functional Requirements

1. **Identity** — SSO-only authentication (OAuth 2.0 / OIDC); server-side token verification; HTTP-only session management
2. **User Management** — User profiles, per-location role assignments, HR system (Arms) sync, location configuration
3. **Attendance Management** — Nightly Athena badge ingestion; two-pass WorkSession resolution; AttendanceRecord computation; reporting
4. **Seating Management** — Real-time floor plan; hot-desk booking; block booking; permanent seat assignment; no-show auto-release
5. **Remote Day Scheduling & OOO** — Request/approval workflow; configurable per-team policy; manager delegation; recurring schedules
6. **Visitor Management (Phase 2)** — Pre-registration; walk-in handling; check-in/out; versioned agreement templates; visit audit
7. **Office Events (Phase 2)** — Create events; RSVP; waitlist; recurring events; event cancellation
8. **Supplies Management (Phase 2)** — Supply catalogue; batch-level stock; two-stage approval Saga; procurement reorder trigger
9. **Assets Management (Phase 2)** — Asset register; assignment lifecycle; fault reporting; maintenance tracking; retirement
10. **Notifications** — Event-driven in-app and email dispatch; consumed from Kafka; never called directly
11. **Audit Logging** — Immutable, append-only event-driven audit trail; 24-month retention; archived to S3

### Non-Functional Requirements

| Category | Target |
|----------|--------|
| Service independence | Each service deployable with zero downtime to other services |
| Scale | 2–5 locations, hundreds of employees at launch; horizontally scalable |
| Authentication | SSO only; OAuth 2.0 / OIDC |
| Authorisation | Role + location checks at API Gateway AND within each service |
| Internal security | Zero Trust; mTLS + internal JWT on every service-to-service call |
| Data isolation | Database per service; no shared schemas; no cross-service DB access |
| SQL safety | Parameterised queries only |
| Resilience | Circuit breaker, retry (max 3), timeout (3s), fallback on every REST call |
| Idempotency | All Kafka consumers idempotent; duplicate delivery safe |
| Observability | Structured JSON logs, Prometheus metrics, OpenTelemetry tracing, health probes — all services |
| Audit retention | 24 months active; archived to S3 thereafter |

### Constraints

- Angular frontend, Java/Spring Boot backend mandated by existing team expertise (AD-001)
- SSO provider is a shared org dependency — OMS integrates via OAuth 2.0 / OIDC (AD-002)
- Arms HR system is the authoritative employee directory — `user-service` syncs from it (AD-024)
- Badge event ingestion is nightly via AWS Athena — real-time streaming deferred (AD-025)
- Procurement integration API contract pending discovery session (AD-026)

---

## DOMAIN MODEL

### Service-to-Entity Ownership

Each service owns its entities exclusively. No other service accesses these tables.

| Service | Owned Entities |
|---------|---------------|
| `identity-service` | Session |
| `user-service` | User, UserRole, Location, LocationConfig, PublicHoliday |
| `attendance-service` | BadgeEvent (immutable), WorkSession, AttendanceRecord |
| `seating-service` | Floor, Zone, Seat, SeatBooking, BlockReservation, NoShowRecord |
| `remote-service` | RemoteRequest, RecurringRemoteSchedule, OOORequest, ApprovalDelegate, RemoteDayPolicy |
| `notification-service` | Notification, NotificationTemplate |
| `audit-service` | AuditLog (append-only) |
| `visitor-service` | VisitorProfile, ParentVisit, VisitRecord, AgreementTemplate |
| `event-service` | EventSeries, Event, EventInvite |
| `supplies-service` | SupplyCategory, SupplyItem, SupplyStockEntry, SupplyRequest, ReorderLog |
| `assets-service` | AssetCategory, Asset, AssetAssignment, AssetRequest, MaintenanceRecord, FaultReport |

### Cross-Service Relationships (via Events, Not Joins)

There are no foreign keys across service databases. Cross-service data references use ID values only. Relationships are maintained through Kafka events and API Composition queries.

```
user-service         → publishes user.deactivated
attendance-service   → consumes user.deactivated (stop generating records)
seating-service      → consumes user.deactivated (cancel future bookings)

remote-service       → publishes remote.request.approved
attendance-service   → consumes remote.request.approved (overlay REMOTE status)
notification-service → consumes remote.request.approved (notify employee)

attendance-service   → publishes attendance.no_show.detected
seating-service      → consumes attendance.no_show.detected (release unattended bookings)

(all services)       → publish audit.event
audit-service        → consumes audit.event (persist immutably)

(all services)       → publish domain events
notification-service → consumes all notification-trigger events
```

---

## SERVICE ARCHITECTURE

### System Diagram

```
External Clients (Angular SPA)
            │  HTTPS
            ▼
  ┌──────────────────────────┐
  │        API Gateway        │
  │  Auth · Route · Rate · Log│
  │  (calls identity-service  │
  │   /validate per request)  │
  └────────────┬─────────────┘
               │  Internal mTLS + JWT
     ┌─────────┴──────────────────────────────────────────────────┐
     │                   Service Mesh (Istio / App Mesh)            │
     ├──────────────┬──────────────────┬─────────────────────────  │
     │              │                  │                            │
  identity-     user-service     attendance-          seating-      │
  service       [user_db]        service              service       │
  [identity_db]                  [attendance_db]      [seating_db]  │
     │              │                  │                            │
     └──────────────┴──────────────────┴────────────────────────── ┘
                                    │
                         ┌──────────┴──────────┐
                         │     Apache Kafka      │
                         │  (AWS MSK — durable   │
                         │   event bus)          │
                         └──────────┬────────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │                                 │
           notification-service               audit-service
           [notification_db]                  [audit_db — append-only]
```

### Service Inventory

| Service | Port | Phase | Key Dependencies |
|---------|------|-------|-----------------|
| `identity-service` | 8081 | 1 | SSO provider |
| `user-service` | 8082 | 1 | Arms HR API, `identity-service` |
| `attendance-service` | 8083 | 1 | AWS Athena, `user-service` (REST) |
| `seating-service` | 8084 | 1 | `user-service` (REST), Kafka |
| `remote-service` | 8085 | 1 | `user-service` (REST), Kafka |
| `notification-service` | 8086 | 1 | Kafka, email provider |
| `audit-service` | 8087 | 1 | Kafka |
| `visitor-service` | 8088 | 2 | `user-service` (REST), Kafka |
| `event-service` | 8089 | 2 | `user-service` (REST), Kafka |
| `supplies-service` | 8090 | 2 | `user-service` (REST), Kafka |
| `assets-service` | 8091 | 2 | `user-service` (REST), Kafka |

---

## DATA ARCHITECTURE

### Technology Choice

**PostgreSQL** — one database per service. All 11 databases are isolated instances with separate AWS RDS clusters and separate DB user credentials. No service's DB user has credentials for any other service's database.

Justification:
- JSONB support for audit state snapshots and raw badge event payloads
- Relational integrity within service boundaries (UUID keys, soft-delete patterns)
- Production-ready with Spring Data JPA / Hibernate
- Consistent tooling and operational model across all services

### Schema Conventions (Per Service)

```sql
-- Every entity
id             UUID PRIMARY KEY DEFAULT gen_random_uuid()
created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
created_by     UUID NOT NULL              -- User ID from session context

-- Every feature entity (not identity-service / user-service globals)
location_id    UUID NOT NULL              -- Data isolation at schema level

-- Most mutable entities
deleted_at     TIMESTAMPTZ               -- Soft delete; NULL = active
```

### Attendance Data Pipeline

```
AWS Athena (badge_events table)
        ↓ nightly Athena query (attendance-service job)
BadgeEvent records (immutable; persisted once; never modified)
        ↓ WorkSession resolution engine
WorkSession records
  - first_badge_in, last_badge_out, total_duration_minutes
  - is_late, minutes_late, crosses_midnight
  - Owned by start date (cross-midnight handled)
        ↓ Pass 1
AttendanceRecord (PRESENT / LATE for badge-active days)
        ↓ Pass 2 (Kafka consumer: remote.request.approved, ooo.request.approved)
Final AttendanceRecord per user per date
(PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY)
```

### Caching Strategy

No application-level caching at launch. Rationale:
- **Seat availability** is computed in real-time from SeatBooking records. At hundreds of seats per location, properly indexed queries are sub-200ms. Cache would require near-constant invalidation on bookings, cancellations, and no-show releases.
- **Role/location checks** are resolved from the session object (populated at login via `user-service`); no per-request DB join after session establishment.
- **Redis** can be introduced as a caching layer for seat availability if query latency degrades with scale — this is a service-internal change requiring no architectural restructuring.

### Key Database Indexes (Per Service)

```sql
-- All feature tables — location-scoped lookups
CREATE INDEX idx_{table}_location ON {table}(location_id) WHERE deleted_at IS NULL;

-- Attendance service — most queried pattern
CREATE INDEX idx_attendance_record_user_date
  ON attendance_records(user_id, record_date, location_id);
CREATE INDEX idx_badge_event_user_occurred
  ON badge_events(user_id, occurred_at, location_id);

-- Seating service — real-time availability
CREATE INDEX idx_seat_booking_date_location
  ON seat_bookings(booking_date, location_id) WHERE deleted_at IS NULL;

-- Audit service — query patterns
CREATE INDEX idx_audit_log_actor ON audit_logs(actor_id, occurred_at);
CREATE INDEX idx_audit_log_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_log_location ON audit_logs(location_id, occurred_at);
```

---

## API SPECIFICATIONS

### Global Conventions

- **Base path:** `/api/v1/` (versioned from day one)
- **Auth:** All endpoints require a valid session. Session validated by API Gateway via `identity-service`.
- **Response envelope:** `{ "success": true/false, "data": {}, "error": null }`
- **Pagination:** All list endpoints paginated; never unbounded
- **Content type:** `application/json`
- **Internal mTLS + JWT:** All service-to-service calls authenticated

### API Gateway Routing

```
/api/v1/auth/**          → identity-service:8081
/api/v1/users/**         → user-service:8082
/api/v1/locations/**     → user-service:8082
/api/v1/attendance/**    → attendance-service:8083
/api/v1/seat*/**         → seating-service:8084
/api/v1/remote*/**       → remote-service:8085
/api/v1/ooo*/**          → remote-service:8085
/api/v1/notifications/** → notification-service:8086
/api/v1/audit-logs/**    → audit-service:8087
/api/v1/visitors/**      → visitor-service:8088
/api/v1/events/**        → event-service:8089
/api/v1/supplies/**      → supplies-service:8090
/api/v1/assets/**        → assets-service:8091
```

### Kafka Topic Registry

| Topic | Producer | Consumers |
|-------|---------|-----------|
| `oms.user.user.created` | user-service | notification-service, audit-service |
| `oms.user.user.deactivated` | user-service | attendance-service, seating-service, audit-service |
| `oms.attendance.record.resolved` | attendance-service | audit-service |
| `oms.attendance.no_show.detected` | attendance-service | seating-service |
| `oms.seating.booking.created` | seating-service | notification-service, audit-service |
| `oms.seating.booking.released` | seating-service | notification-service, audit-service |
| `oms.remote.request.approved` | remote-service | attendance-service, notification-service, audit-service |
| `oms.remote.request.rejected` | remote-service | notification-service, audit-service |
| `oms.visitor.visit.checked_in` | visitor-service | notification-service, audit-service |
| `oms.event.rsvp.promoted` | event-service | notification-service, audit-service |
| `oms.event.event.cancelled` | event-service | notification-service, audit-service |
| `oms.supplies.request.fulfilled` | supplies-service | notification-service, audit-service |
| `oms.supplies.reorder.triggered` | supplies-service | procurement adapter, audit-service |
| `oms.assets.asset.assigned` | assets-service | notification-service, audit-service |
| `oms.audit.event` | all services | audit-service |

---

## INFRASTRUCTURE ARCHITECTURE

### Deployment Architecture

```
Internet / Corporate Network
        ↓ HTTPS / TLS 1.3
  [API Gateway — AWS API Gateway or Kong]
  Auth · Rate Limiting · Routing · Logging
        ↓ Internal mTLS (Istio / App Mesh)
  ┌─────────────────────────────────────────┐
  │         AWS EKS Kubernetes Cluster       │
  │  ┌──────────────────────────────────┐   │
  │  │  Service Namespace (per service) │   │
  │  │  Deployment (min 2 replicas)     │   │
  │  │  HPA (scale on CPU > 70%)        │   │
  │  │  Liveness + Readiness probes     │   │
  │  └──────────────────────────────────┘   │
  └──────────────────┬──────────────────────┘
                     │
        ┌────────────┼───────────────┐
        │            │               │
   AWS RDS        AWS MSK        AWS S3
  (PostgreSQL)   (Kafka)       (audit archival)
  11 isolated    multi-AZ       cold storage
  instances      cluster        >24 months
```

### Kubernetes Per-Service Resources

```yaml
# Per service — Deployment
replicas: 2               # Minimum for availability
resources:
  requests: { cpu: 250m, memory: 512Mi }
  limits:   { cpu: 1000m, memory: 1Gi }

# Per service — HorizontalPodAutoscaler
minReplicas: 2
maxReplicas: 10
targetCPUUtilizationPercentage: 70

# Probes
livenessProbe:  GET /actuator/health/liveness
readinessProbe: GET /actuator/health/readiness
```

### CI/CD Pipeline (Per Service)

Each service has its own independent pipeline. Deploying `seating-service` never blocks `attendance-service`.

```
PR created
  ↓
[GitHub Actions]
  1. Lint (Checkstyle)
  2. Unit tests (JUnit 5)
  3. Integration tests (TestContainers)
  4. Pact contract tests (consumer side)
  5. OWASP dependency security scan
  6. Docker build + push to ECR
  ↓
Merge to develop   → Auto-deploy to DEV
Merge to staging   → Auto-deploy to STAGING + run provider Pact verification
Merge to main      → Manual approval gate → PRODUCTION (with rollback plan)
```

### Disaster Recovery

- **RDS automated backups:** daily snapshots + point-in-time recovery (7-day window)
- **Kafka:** multi-AZ MSK cluster; topic replication factor ≥ 3
- **Audit archival:** records > 24 months moved to S3 (Glacier Instant Retrieval)
- **RTO target:** < 4 hours per service
- **RPO target:** < 24 hours (aligned with daily backup cadence)

---

## SECURITY ARCHITECTURE

### Authentication Flow

```
1. User navigates to OMS Angular SPA
2. SPA redirects to SSO provider
3. User authenticates at SSO provider
4. SSO redirects with OIDC authorization code to identity-service /api/v1/auth/callback
5. identity-service exchanges code for OIDC token with SSO provider
6. identity-service validates token (signature via JWKS, expiry, audience)
7. identity-service issues HTTP-only session cookie to the SPA
8. SPA sends session cookie on every request
9. API Gateway calls identity-service /api/v1/auth/validate on every request
10. identity-service resolves user identity and roles from session
11. API Gateway routes with user context injected in header
12. Downstream service validates internal JWT (Zero Trust)
```

### Zero Trust Service-to-Service

```
service-A calls service-B
  ↓
service-A presents client certificate (mTLS) and internal JWT
service-B verifies:
  1. mTLS — is this a known service in our mesh?
  2. Internal JWT — is this JWT signed by our internal key?
  3. Role check — does the acting user have the required role?
  4. Location check — does the acting user have that role at the target location?
```

### Authorisation Matrix

| Check | Enforcement Point |
|-------|------------------|
| Session validity | API Gateway → `identity-service /validate` |
| Role check | API Gateway (coarse) + Service layer (authoritative) |
| Location check | Service layer (authoritative) |
| Internal JWT | Every service via Spring Security `@oauth2ResourceServer` |

### Data Security

- TLS 1.3 on all external traffic
- mTLS on all internal service-to-service traffic
- All secrets via AWS Secrets Manager — never in code or config files
- Separate RDS DB user per service — minimum privilege (audit DB user: INSERT only)
- All data in transit: TLS; all data at rest: RDS encryption enabled
- JSONB audit snapshots create tamper-evident state history
- Parameterised queries only — no SQL injection surface

---

## ARCHITECTURE DECISION RECORDS

Key decisions specific to the microservices design (full ADR log in `doch.md`):

| ADR | Decision | Trade-off Accepted |
|-----|----------|--------------------|
| AD-001 | Angular + Java/Spring Boot + Docker | Angular boilerplate vs enterprise suitability |
| AD-002 | SSO-only authentication | OMS availability dependent on SSO provider |
| AD-003 | Server-side token verification | Slightly more complex auth flow |
| AD-004 | Location-scoped data architecture | Every query requires location filter |
| AD-005 | Role-per-location model | Role checks require UserRole join per operation |
| AD-006 | UUID primary keys | Larger index size vs integer keys |
| AD-007 | Soft deletes | All queries must filter `deleted_at IS NULL` |
| AD-008 | WorkSession separate from AttendanceRecord | More entities; cleaner separation of concerns |
| AD-009 | Two-pass attendance resolution | More complex sync job |
| AD-010 | Cross-midnight gap threshold | Requires per-location tuning |
| AD-015 | Real-time seat availability (no cache) | Slightly more expensive per query at scale |
| AD-016 | No-show auto-release via scheduled job | Employees arriving after cutoff may lose seats |
| AD-023 | Immutable audit log | Log grows indefinitely; archival job required |
| AD-025 | Nightly badge ingestion (not real-time) | Attendance data is T+1 day behind |
| AD-026 | Procurement integration TBD | Phase 2 reorder cannot complete until contract confirmed |
| — | Choreography-based Saga (not orchestration) | Each service reacts independently; no orchestrator SPOF |
| — | Kafka over direct REST for state changes | Durable replayable log; loose coupling; audit replay capability |
| — | Pact contract testing | 11 services require contract guarantees without full integration environments |
| — | No CQRS at launch | Read/write patterns not complex enough; can be introduced per service post-launch |

---

## NEXT STEPS & ROADMAP

### Immediate Pre-Development Actions

1. Confirm API Gateway selection (AWS API Gateway vs Kong) — blocks M1
2. Confirm service mesh selection (Istio vs AWS App Mesh) — blocks M1
3. Confirm SSO provider JWKS endpoint availability — blocks M2
4. Obtain Arms HR API contract and test credentials — blocks M2
5. Obtain AWS Athena badge event schema and sample data — blocks M4
6. Confirm email provider for `notification-service` — blocks M3
7. Schedule procurement discovery session — blocks M10 reorder integration (AD-026)

### Phase 1 Delivery Sequence

Build order is enforced by service dependencies:

```
M1: Infrastructure (EKS, Kafka, RDS, Gateway, observability)
  ↓
M2: identity-service + user-service (all other services depend on auth and user context)
  ↓
M3: audit-service + notification-service (must exist before business services start producing events)
  ↓
M4: attendance-service (badge pipeline; publishes no_show events consumed by seating-service)
  ↓
M5: seating-service (consumes attendance events)
  ↓
M6: remote-service (publishes approved events consumed by attendance-service)
  ↓
M7: Phase 1 hardening, security review, UAT, launch
```

### Phase 2 Delivery Sequence

```
M8: visitor-service
M9: event-service
M10: supplies-service (after procurement contract confirmed)
M11: assets-service
M12: Phase 2 hardening and launch
```

### Future Considerations (Post-Launch)

- CQRS per service where read/write traffic diverges significantly
- Redis caching layer for seat availability if query latency degrades at scale
- Real-time badge event streaming (replace nightly batch if same-day attendance becomes a requirement)
- Mobile-native application
- Room booking integration

---

*End of System Architecture Document*
