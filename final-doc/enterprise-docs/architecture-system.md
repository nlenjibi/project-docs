# SYSTEM ARCHITECTURE: Office Management System (OMS)
**Version:** 3.0 — Spring Cloud Edition  
**Status:** Draft  
**Last Updated:** April 2026  
**Frontend:** React | **Backend:** Java 21 + Spring Boot 3.x + Spring Cloud  
**Broker:** RabbitMQ (Amazon MQ) | **Orchestration:** AWS ECS Fargate

---

## EXECUTIVE SUMMARY

The Office Management System (OMS) is a cloud-native, event-driven microservices platform that replaces manual, spreadsheet-driven office operations with a unified, auditable, location-aware system. It serves employees, managers, HR, and facilities staff across 2–5 office locations.

The platform is composed of **8 independently deployable Spring Boot services**, each owning exactly one business capability and its own PostgreSQL database. All external traffic enters through a Spring Cloud Gateway. Services discover each other via Netflix Eureka. Asynchronous workflows use RabbitMQ via Amazon MQ with Topic Exchanges and durable queues. All infrastructure runs on AWS ECS Fargate.

**Three non-negotiable architectural rules drive every decision:**
1. **No shared databases** — each service owns its data exclusively; cross-service data flows through events and APIs
2. **No direct calls to notification or audit services** — these are event-driven only; no service waits for them
3. **Zero Trust** — every service validates every caller on every request, regardless of network position

---

## REQUIREMENTS SUMMARY

### Functional Requirements

| ID | Requirement |
|---|---|
| F-01 | SSO-only authentication (OAuth 2.0 / OIDC); server-side token verification |
| F-02 | User profiles, per-location role assignments, HR system sync |
| F-03 | Automated attendance from nightly badge event ingestion via AWS Athena |
| F-04 | Two-pass attendance resolution: PRESENT/LATE from badges + REMOTE/OOO overlays from events |
| F-05 | Real-time floor plan with hot-desk booking, permanent seat assignment, block reservations |
| F-06 | No-show auto-release: unattended bookings released at configurable daily cutoff |
| F-07 | Remote day and OOO request/approval workflow with configurable policy enforcement |
| F-08 | Approval delegation: manager nominates delegate before going OOO |
| F-09 | Event-driven in-app and email notification dispatch |
| F-10 | Immutable, append-only audit log for all state-changing operations |
| F-11 | Visitor pre-registration, walk-in handling, check-in/out, agreement versioning (Phase 2) |
| F-12 | Office event management, RSVP, waitlist, recurring events (Phase 2) |
| F-13 | Supply catalogue, batch-level stock tracking, two-stage approval, reorder trigger (Phase 2) |
| F-14 | Asset register, assignment lifecycle, fault reporting, maintenance tracking (Phase 2) |

### Non-Functional Requirements

| Category | Target |
|---|---|
| Service independence | Any service deployable with zero downtime to other services |
| Availability | No single service failure causes system-wide outage |
| Seat booking latency | < 200ms p95 under normal load |
| Scale | 2–5 locations, hundreds of employees at launch; horizontally scalable |
| Authentication | SSO only; OAuth 2.0 / OIDC |
| Authorisation | Role + location checks at gateway (coarse) and within each service (authoritative) |
| Internal security | Zero Trust; internal JWT on every service-to-service call |
| Data isolation | Database per service; no shared schemas; no cross-service DB access |
| SQL safety | Parameterised queries only; no string concatenation |
| Resilience | Circuit breaker, retry (max 3), timeout (3s), fallback on every REST call |
| Idempotency | All RabbitMQ consumers idempotent; duplicate delivery safe |
| Observability | Structured JSON logs, Prometheus metrics, OpenTelemetry traces, health probes on all services |
| Audit retention | 24 months active; archived to S3 thereafter |

### Constraints

| Constraint | Detail |
|---|---|
| Technology | React frontend, Java 21/Spring Boot 3.x per service — team expertise |
| Auth | Must integrate with existing SSO provider (OAuth 2.0 / OIDC) |
| Employee data | Arms HR system is the source of truth; `auth-service` syncs from it |
| Badge data | Ingested from AWS Athena nightly batch; real-time streaming deferred |
| Procurement | API contract pending discovery session (Phase 2 blocker) |
| Tracing | AWS X-Ray excluded; OpenTelemetry + Jaeger required |

---

## DOMAIN MODEL

### Service-to-Entity Ownership

No service accesses another service's database. Cross-service relationships use ID values only.

| Service | Owned Entities |
|---|---|
| `auth-service` | Session, User, UserRole, Location, LocationConfig, PublicHoliday |
| `attendance-service` | BadgeEvent (immutable), WorkSession, AttendanceRecord, BadgeSyncJobLog |
| `seating-service` | Floor, Zone, Seat, SeatBooking, BlockReservation, NoShowRecord |
| `remote-service` | RemoteRequest, RecurringRemoteSchedule, OOORequest, ApprovalDelegate, RemoteDayPolicy |
| `notification-service` | Notification, NotificationTemplate |
| `audit-service` | AuditLog (append-only, INSERT-only DB user) |
| `inventory-service` | SupplyCategory, SupplyItem, SupplyStockEntry, SupplyRequest, ReorderLog, AssetCategory, Asset, AssetAssignment, AssetRequest, MaintenanceRecord, FaultReport |
| `workplace-service` | VisitorProfile, ParentVisit, VisitRecord, AgreementTemplate, EventSeries, Event, EventInvite |

### Cross-Service Event Flow

```
auth-service         → publishes user.deactivated
attendance-service   → consumes user.deactivated (stop generating records)
seating-service      → consumes user.deactivated (cancel future bookings)

remote-service       → publishes remote.request.approved
attendance-service   → consumes remote.request.approved (overlay REMOTE status)
notification-service → consumes remote.request.approved (notify employee)

attendance-service   → publishes attendance.no_show.detected
seating-service      → consumes attendance.no_show.detected (release unattended booking)

(all services)       → publish oms.audit.event
audit-service        → consumes all audit events (persist immutably)

(all services)       → publish domain events to their exchange
notification-service → consumes relevant notification-triggering events
```

---

## SERVICE ARCHITECTURE

### System Diagram

```
React SPA (Browser)
        │  HTTPS / TLS 1.3
        ▼
[AWS ALB — public-facing]
        │
        ▼
[Spring Cloud Gateway — ECS Fargate, 2+ tasks, port 8080]
   Session validation → auth-service (Eureka lb://)
   Routing by path    → lb://service-name (Eureka)
   Rate limiting      → Redis (ElastiCache)
   Correlation ID     → X-Correlation-ID generated and propagated
        │  HTTPS internal (VPC private subnets)
        ▼
┌───────────────────────────────────────────────────────────┐
│                  ECS Cluster: oms-cluster                  │
│                                                            │
│  [eureka-server]  :8761  — service registry (2 tasks HA)  │
│                                                            │
│  [auth-service]   :8081  SSO · users · roles · HR sync    │
│  [attendance]     :8083  badge ingestion · attendance      │
│  [seating]        :8084  floor plans · bookings            │
│  [remote]         :8085  remote/OOO requests · approvals  │
│  [notification]   :8086  in-app + email dispatch          │
│  [audit]          :8087  immutable audit log               │
│  [inventory]      :8090  supplies + assets  (Phase 2)     │
│  [workplace]      :8088  visitors + events  (Phase 2)     │
└─────────────────────────┬─────────────────────────────────┘
                          │
        ┌─────────────────┼──────────────────┐
        │                 │                  │
   Amazon MQ          AWS RDS            AWS S3
   RabbitMQ           PostgreSQL         Audit archival
   active/standby     8 isolated         > 24 months
   Topic Exchanges    instances          Glacier IR
   Durable Queues
```

### Service Inventory

| Service | Port | Phase | Replaces | Min Tasks | Key Dependencies |
|---|---|---|---|---|---|
| `api-gateway` | 8080 | 1 | — | 2 | auth-service, Eureka, Redis |
| `eureka-server` | 8761 | 1 | — | 2 | — |
| `auth-service` | 8081 | 1 | identity + user | 2 | SSO provider, Arms HR API |
| `attendance-service` | 8083 | 1 | attendance | 2 | AWS Athena, auth-service (REST), RabbitMQ |
| `seating-service` | 8084 | 1 | seating | 2 | auth-service (REST), RabbitMQ |
| `remote-service` | 8085 | 1 | remote | 2 | auth-service (REST), RabbitMQ |
| `notification-service` | 8086 | 1 | notification | 2 | RabbitMQ, email provider |
| `audit-service` | 8087 | 1 | audit | 2 | RabbitMQ |
| `inventory-service` | 8090 | 2 | supplies + assets | 2 | auth-service (REST), RabbitMQ |
| `workplace-service` | 8088 | 2 | visitor + event | 2 | auth-service (REST), RabbitMQ |

---

## DATA ARCHITECTURE

### Technology Choice

**PostgreSQL 16 (AWS RDS, Multi-AZ, `db.t3.small`)** — one isolated instance per service.

Justification:
- JSONB support for audit state snapshots and raw badge event payloads
- Relational integrity within service boundaries
- Consistent tooling across all services: Spring Data JPA / Hibernate, Flyway migrations, TestContainers
- No polyglot overhead: all services share the same data model characteristics

### Database Isolation Guarantee

```
Service:         seating-service
DB instance:     seating_db (RDS)
DB user:         seating_app (SELECT, INSERT, UPDATE, DELETE on seating_* tables)
Firewall:        sg-rds-seating allows inbound 5432 from sg-seating-service ONLY

No other service's DB user has ANY privilege on seating_db.
No other service's security group can reach the seating_db instance.
This is enforced at both the application AND network level.
```

### Attendance Data Pipeline

```
AWS Athena (badge_events table)
        ↓ nightly BadgeSyncJob (2am, configurable per location)
BadgeEvent records (immutable; persisted once; never modified)
        ↓ WorkSession resolution engine
WorkSession (first_badge_in, last_badge_out, total_duration, is_late, crosses_midnight)
        ↓ Pass 1
AttendanceRecord: PRESENT or LATE (from badge data)
        ↓ Pass 2 (RabbitMQ consumers)
Final AttendanceRecord: PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY
```

---

## API SPECIFICATIONS

### Global API Conventions

- **Base path:** `/api/v1/`
- **Auth:** HTTP-only session cookie validated by gateway on every request
- **Response envelope:** `{ "success": true/false, "data": {}, "error": null }`
- **Pagination:** `?page=0&size=20` on all list endpoints; never unbounded
- **Content type:** `application/json`
- **Versioning:** URL-based; breaking changes → `/api/v2/`
- **Internal JWT:** Every service-to-service call carries `Authorization: Bearer <internal-jwt>`

### API Gateway Routing

```
/api/v1/auth/**              → auth-service:8081
/api/v1/users/**             → auth-service:8081
/api/v1/locations/**         → auth-service:8081
/api/v1/attendance/**        → attendance-service:8083
/api/v1/seat-bookings/**     → seating-service:8084
/api/v1/seats/**             → seating-service:8084
/api/v1/remote-requests/**   → remote-service:8085
/api/v1/ooo-requests/**      → remote-service:8085
/api/v1/notifications/**     → notification-service:8086
/api/v1/audit-logs/**        → audit-service:8087
/api/v1/supplies/**          → inventory-service:8090  (Phase 2)
/api/v1/assets/**            → inventory-service:8090  (Phase 2)
/api/v1/visitors/**          → workplace-service:8088  (Phase 2)
/api/v1/events/**            → workplace-service:8088  (Phase 2)
```

### RabbitMQ Exchange Registry

| Exchange | Type | Producer | Consumer Queues |
|---|---|---|---|
| `oms.user` | topic | auth-service | notification, audit |
| `oms.attendance` | topic | attendance-service | seating, audit |
| `oms.seating` | topic | seating-service | notification, audit |
| `oms.remote` | topic | remote-service | attendance, notification, audit |
| `oms.visitor` | topic | workplace-service | notification, audit |
| `oms.event` | topic | workplace-service | notification, audit |
| `oms.inventory` | topic | inventory-service | notification, audit |
| `oms.audit` | topic | all services | audit-service |

---

## INFRASTRUCTURE ARCHITECTURE

### Deployment Architecture

```
Internet
    ↓ HTTPS TLS 1.3
AWS ALB (public subnet)
    ↓
Spring Cloud Gateway (ECS Fargate, 2+ tasks, private subnet)
    ↓ HTTPS internal
ECS Cluster (8 services + Eureka, private subnets, multi-AZ)
    ↓
Amazon MQ RabbitMQ (active/standby, private subnets)
AWS RDS PostgreSQL (Multi-AZ, private subnets, 8 instances)
AWS S3 (audit archival, Glacier Instant Retrieval)
```

### Key AWS Services

| Service | Configuration | Monthly Cost (est.) |
|---|---|---|
| ECS Fargate | 10 services × 2 tasks avg, 0.25 vCPU, 512 MB | ~$80–150 |
| Amazon MQ | `mq.m5.large` active/standby | ~$230 |
| AWS RDS | 8 × `db.t3.small` Multi-AZ | ~$300–480 |
| AWS ALB | 2 LCUs baseline | ~$20–35 |
| ElastiCache | `cache.t3.micro` | ~$15 |
| **Total** | | **~$710–970/month** |

### CI/CD Per Service

```
PR created → GitHub Actions:
  1. Checkstyle lint
  2. JUnit 5 unit tests
  3. Spring Boot + TestContainers integration tests
  4. Pact consumer contract tests
  5. OWASP Dependency Check (fail CVSS >= 7)
  6. Docker multi-stage build
  7. Push to AWS ECR (on merge)

develop → auto-deploy DEV
staging → auto-deploy STAGING + Pact provider verify
main    → manual approval gate → PRODUCTION (with rollback plan)
```

---

## SECURITY ARCHITECTURE

### Authentication Flow

```
1. User → React SPA → clicks Login
2. SPA redirects to SSO (OAuth 2.0 Authorization Code flow)
3. User authenticates at SSO
4. SSO redirects to auth-service /api/v1/auth/callback
5. auth-service validates OIDC token (JWKS, expiry, audience, issuer)
6. auth-service creates session in auth_db
7. auth-service issues HTTP-only SameSite=Strict session cookie
8. All subsequent requests: browser sends cookie automatically
9. Gateway extracts cookie → calls auth-service /validate
10. auth-service returns UserContext { userId, roles, locationIds }
11. Gateway generates internal JWT, injects X-User-* headers, forwards to service
12. Service validates internal JWT, performs role + location check
```

### Security Controls Summary

| Layer | Control |
|---|---|
| Network | VPC private subnets; ALB only public entry point; security groups per service |
| Edge | Rate limiting (100 req/s per user); CORS enforcement; AWS WAF (OWASP CRS) |
| Identity | SSO only; HTTP-only cookies; immediate session revocation |
| Service-to-Service | Internal JWT (HMAC-SHA256, 300s TTL); validated on every inbound call |
| Authorisation | Role + location check in every service (`@PreAuthorize` + service layer) |
| Data | Parameterised SQL; soft deletes; separate DB user per service; INSERT-only for audit |
| Secrets | AWS Secrets Manager; injected as env vars; never in code or git |
| Transport | TLS 1.3 external; HTTPS internal |
| Audit | Immutable audit log via RabbitMQ consumer; no REST write endpoint |

---

## ARCHITECTURE DECISION RECORDS

Full ADR log is in [adr.md](adr.md). Key decisions:

| ADR | Decision | Primary Trade-off |
|---|---|---|
| ADR-001 | React frontend | Less opinionated — team sets conventions |
| ADR-002 | Microservices | Distributed complexity accepted |
| ADR-003 | 8 services (merged from 11) | Larger merged codebases |
| ADR-004 | ECS Fargate over EKS | No Istio — internal JWT-based Zero Trust instead |
| ADR-005 | Netflix Eureka | Eureka Server is additional infrastructure |
| ADR-006 | Spring Cloud Gateway | Java filter code + Redis for rate limiting |
| ADR-007 | RabbitMQ over Kafka/SQS | No event replay; saves ~$2,796/year vs Kafka |
| ADR-008 | REST not GraphQL | Dashboard composition via BFF endpoint |
| ADR-009 | PostgreSQL per service | 8 RDS instances; no cross-service joins |
| ADR-010 | Zero Trust — internal JWT | Manual cert rotation without Istio |
| ADR-012 | OpenTelemetry + Jaeger | Jaeger must be operated as ECS service |
| ADR-013 | HTTP-only session cookie | CSRF protection required |
| ADR-014 | Notification/audit event-driven only | Audit is slightly async |
| ADR-020 | Location-scoped RBAC | Every auth check needs role + location |

---

## NEXT STEPS AND ROADMAP

### Pre-Development Actions (Must complete before M1)

1. Confirm SSO provider JWKS endpoint availability and client_id/secret — blocks M2
2. Obtain Arms HR API contract and test credentials — blocks M2 (HR sync)
3. Confirm AWS Athena badge event schema and sample data — blocks M4
4. Confirm email provider API contract — blocks notification-service
5. Resolve Amazon MQ region availability (verify `mq.m5.large` active/standby in target region)
6. Schedule procurement discovery session — blocks M10 reorder integration

### Phase 1 Build Order

```
M1: Infrastructure
    ECS cluster, Amazon MQ, 8 RDS instances, ALB, Eureka Server,
    Spring Cloud Gateway, Prometheus + Grafana, Jaeger, Secrets Manager

M2: auth-service
    SSO integration, session management, user profiles, roles, HR sync
    (All other services depend on auth)

M3: audit-service + notification-service
    RabbitMQ consumers must be live before business services start producing events

M4: attendance-service
    Badge pipeline; publishes no_show events consumed by seating-service

M5: seating-service
    Consumes attendance events

M6: remote-service
    Publishes approved events consumed by attendance-service

M7: Phase 1 hardening, security review, performance testing, UAT, launch
```

### Phase 2 Build Order

```
M8:  workplace-service (visitor + event)
M9:  inventory-service (supplies) — after procurement contract confirmed
M10: inventory-service (assets) — can be delivered with M9 or separately
M11: Phase 2 hardening and launch
```

### Future Considerations (Post-Launch)

- CQRS per service where read/write traffic diverges significantly
- Redis caching for seat availability if query latency grows with seat count
- Real-time badge streaming (replace nightly batch if same-day attendance required)
- Spring Cloud Config Server for runtime config refresh without redeployment
- Mobile app (React Native — shares API contracts; no backend changes needed)
- Multi-region deployment if organisation expands beyond single AWS region

---

*End of System Architecture Document*  
*See [README.md](README.md) for the full documentation index.*
