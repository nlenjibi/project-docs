# SYSTEM ARCHITECTURE: Office Management System (OMS)
**Version:** 4.0 — AWS-Native Edition  
**Status:** Draft  
**Last Updated:** April 2026  
**Frontend:** React | **Backend:** Java 21 + Spring Boot 3.x (core services) + FastAPI / Python 3.12 (async services)  
**Broker:** SNS + SQS | **Orchestration:** AWS ECS Fargate | **Discovery:** AWS Cloud Map

---

## EXECUTIVE SUMMARY

The Office Management System (OMS) is a cloud-native, event-driven microservices platform that replaces manual, spreadsheet-driven office operations with a unified, auditable, location-aware system. It serves employees, managers, HR, and facilities staff across 2–5 office locations.

The platform is composed of **8 independently deployable services** (Spring Boot or FastAPI), each owning exactly one business capability and its own PostgreSQL database. All external traffic enters through **AWS API Gateway (HTTP API)** backed by a **VPC Link**. Services discover each other via **AWS Cloud Map** private DNS. Asynchronous workflows use **Amazon SNS + SQS** (fan-out pattern). All infrastructure runs on AWS ECS Fargate. Authentication at the edge uses an **AWS Lambda Authorizer** that calls `auth-service /internal/validate` and returns user context as request context. CI/CD runs through **GitHub → Jenkins → ECR → ECS**.

**Three non-negotiable architectural rules drive every decision:**
1. **No shared databases** — each service owns its data exclusively; cross-service data flows through events and APIs
2. **No direct calls to notification or audit services** — these are event-driven only; no service waits for them
3. **Zero Trust** — every service validates every caller on every request via internal JWT, regardless of network position

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
| F-09 | Event-driven in-app and email notification dispatch via Amazon SES |
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
| Authorisation | Lambda Authorizer at API Gateway (coarse) + role + location check within each service (authoritative) |
| Internal security | Zero Trust; internal JWT on every service-to-service call |
| Data isolation | Database per service; no shared schemas; no cross-service DB access |
| SQL safety | Parameterised queries only; no string concatenation |
| Resilience | Circuit breaker, retry (max 3), timeout (3s), fallback on every REST call |
| Idempotency | All SQS consumers idempotent; duplicate delivery safe |
| Observability | Structured JSON logs → CloudWatch, CloudWatch Container Insights, CloudWatch Alarms |
| Audit retention | 24 months active; archived to S3 thereafter |

### Constraints

| Constraint | Detail |
|---|---|
| Technology | React frontend; Java 21 / Spring Boot 3.x for core services; Python 3.12 / FastAPI for async/data services |
| Auth | Must integrate with existing SSO provider (OAuth 2.0 / OIDC) |
| Employee data | Arms HR system is the source of truth; `auth-service` syncs from it |
| Badge data | Ingested from AWS Athena nightly batch; real-time streaming deferred |
| CI/CD | GitHub → Jenkins → ECR → ECS Fargate |
| Tracing | CloudWatch Container Insights + structured correlation IDs |

---

## DOMAIN MODEL

### Service-to-Entity Ownership

No service accesses another service's database. Cross-service relationships use ID values only.

| Service | Runtime | Owned Entities |
|---|---|---|
| `auth-service` | Spring Boot | Session, User, UserRole, Location, LocationConfig, PublicHoliday |
| `attendance-service` | **FastAPI** | BadgeEvent (immutable), WorkSession, AttendanceRecord, BadgeSyncJobLog |
| `seating-service` | Spring Boot | Floor, Zone, Seat, SeatBooking, BlockReservation, NoShowRecord |
| `remote-service` | Spring Boot | RemoteRequest, RecurringRemoteSchedule, OOORequest, ApprovalDelegate, RemoteDayPolicy |
| `notification-service` | **FastAPI** | Notification, NotificationTemplate |
| `audit-service` | Spring Boot | AuditLog (append-only, INSERT-only DB user) |
| `inventory-service` | Spring Boot | SupplyCategory, SupplyItem, SupplyStockEntry, SupplyRequest, ReorderLog, AssetCategory, Asset, AssetAssignment, AssetRequest, MaintenanceRecord, FaultReport |
| `workplace-service` | Spring Boot | VisitorProfile, ParentVisit, VisitRecord, AgreementTemplate, EventSeries, Event, EventInvite |

### Cross-Service Event Flow (SNS → SQS Fan-out)

```
auth-service         → publishes to SNS: oms-user
                         → SQS: oms-user-notification  (notification-service)
                         → SQS: oms-user-audit         (audit-service)

remote-service       → publishes to SNS: oms-remote
                         → SQS: oms-remote-attendance   (attendance-service)
                         → SQS: oms-remote-notification (notification-service)
                         → SQS: oms-remote-audit        (audit-service)

attendance-service   → publishes to SNS: oms-attendance
                         → SQS: oms-attendance-seating  (seating-service)
                         → SQS: oms-attendance-audit    (audit-service)

(all services)       → publish to SNS: oms-audit
                         → SQS: oms-audit-service       (audit-service)
```

---

## SERVICE ARCHITECTURE

### System Diagram

```
🌍 USER (Browser / Mobile)
        │  HTTPS / TLS 1.3
        ▼
┌───────────────────────────────────────────────────┐
│   Amazon CloudFront (CDN + SSL termination)        │
│   - Caches static React SPA assets               │
│   - Routes /api/** to API Gateway                │
└───────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────┐
│   AWS API Gateway (HTTP API)                       │
│   - Lambda Authorizer → auth-service /validate   │
│   - Rate limiting (usage plans)                  │
│   - CORS enforcement                             │
│   - X-Correlation-ID injection                   │
│   - Routes /api/v1/** → VPC Link                 │
└───────────────────────────────────────────────────┘
        │  VPC Link (private)
        ▼
┌──────────────────────────────────────────────────────────────┐
│                  AWS VPC — Private Subnets                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              ECS Cluster: oms-cluster                   │  │
│  │                                                        │  │
│  │  auth-service         :8081  Spring Boot               │  │
│  │  attendance-service   :8083  FastAPI                   │  │
│  │  seating-service      :8084  Spring Boot               │  │
│  │  remote-service       :8085  Spring Boot               │  │
│  │  notification-service :8086  FastAPI                   │  │
│  │  audit-service        :8087  Spring Boot               │  │
│  │  inventory-service    :8090  Spring Boot  (Phase 2)    │  │
│  │  workplace-service    :8088  Spring Boot  (Phase 2)    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │           AWS Cloud Map — oms.local namespace           │  │
│  │   auth-service.oms.local          :8081                │  │
│  │   attendance-service.oms.local    :8083                │  │
│  │   seating-service.oms.local       :8084                │  │
│  │   remote-service.oms.local        :8085                │  │
│  │   notification-service.oms.local  :8086                │  │
│  │   audit-service.oms.local         :8087                │  │
│  └────────────────────────────────────────────────────────┘  │
│                          │                                   │
│        ┌─────────────────┼──────────────────┐               │
│        │                 │                  │               │
│   Amazon SNS+SQS    AWS RDS            AWS S3              │
│   Fan-out queues    PostgreSQL          Audit archival       │
│   Standard queues   8 isolated          > 24 months         │
│   FIFO where        instances           Glacier IR           │
│   ordering needed                                           │
└──────────────────────────────────────────────────────────────┘
```

### Service Inventory

| Service | Runtime | Port | Phase | Min Tasks | Key Dependencies |
|---|---|---|---|---|---|
| `auth-service` | Spring Boot | 8081 | 1 | 2 | SSO provider, Arms HR API |
| `attendance-service` | **FastAPI** | 8083 | 1 | 2 | AWS Athena, auth-service (HTTP), SQS |
| `seating-service` | Spring Boot | 8084 | 1 | 2 | auth-service (HTTP), SQS |
| `remote-service` | Spring Boot | 8085 | 1 | 2 | auth-service (HTTP), SQS |
| `notification-service` | **FastAPI** | 8086 | 1 | 2 | SQS, Amazon SES |
| `audit-service` | Spring Boot | 8087 | 1 | 2 | SQS |
| `inventory-service` | Spring Boot | 8090 | 2 | 2 | auth-service (HTTP), SQS |
| `workplace-service` | Spring Boot | 8088 | 2 | 2 | auth-service (HTTP), SQS |

> **No `api-gateway` ECS service.** AWS API Gateway is a fully managed AWS service — no container to deploy. No `eureka-server` ECS service. AWS Cloud Map is a fully managed AWS service.

---

## DATA ARCHITECTURE

### Technology Choice

**PostgreSQL 16 (AWS RDS, Multi-AZ, `db.t3.small`)** — one isolated instance per service.

- JSONB support for audit state snapshots and raw badge event payloads
- Consistent tooling: Spring Data JPA / Hibernate (Spring Boot services), SQLAlchemy async / Alembic (FastAPI services)
- No polyglot overhead — all services share the same data model characteristics

### Database Isolation Guarantee

```
Service:         attendance-service
DB instance:     attendance_db (RDS)
DB user:         attendance_app (SELECT, INSERT, UPDATE, DELETE on attendance_* tables)
Firewall:        sg-rds-attendance allows inbound 5432 from sg-attendance-service ONLY

No other service's DB user has ANY privilege on attendance_db.
No other service's security group can reach the attendance_db instance.
Enforced at both application AND network level.
```

### Attendance Data Pipeline

```
AWS Athena (badge_events table)
        ↓ nightly BadgeSyncJob (2am, APScheduler, configurable per location)
BadgeEvent records (immutable; persisted once; never modified)
        ↓ WorkSession resolution engine (pure Python)
WorkSession (first_badge_in, last_badge_out, total_duration, is_late, crosses_midnight)
        ↓ Pass 1 (AttendancePass1Service)
AttendanceRecord: PRESENT or LATE (from badge data)
        ↓ Pass 2 (SQS consumers overlay status corrections)
Final AttendanceRecord: PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY
```

---

## API SPECIFICATIONS

### Global API Conventions

- **Base path:** `/api/v1/`
- **Auth:** HTTP-only session cookie validated by Lambda Authorizer at API Gateway on every request
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

All routes resolve via Cloud Map DNS (e.g. `http://attendance-service.oms.local:8083`) — the VPC Link bridges API Gateway to the private namespace.

### SNS Topic + SQS Queue Registry

| SNS Topic | Event Types Published | Producer | SQS Consumer Queues |
|---|---|---|---|
| `oms-user` | `user.created`, `user.updated`, `user.deactivated` | auth-service | `oms-user-notification`, `oms-user-audit` |
| `oms-attendance` | `record.resolved`, `no_show.detected`, `restamp.requested` | attendance-service | `oms-attendance-seating`, `oms-attendance-audit` |
| `oms-seating` | `booking.created`, `booking.cancelled`, `booking.released` | seating-service | `oms-seating-notification`, `oms-seating-attendance`, `oms-seating-audit` |
| `oms-remote` | `request.submitted`, `request.approved`, `request.rejected`, `ooo.request.approved` | remote-service | `oms-remote-attendance`, `oms-remote-notification`, `oms-remote-audit` |
| `oms-audit` | all event types | all services | `oms-audit-service` |

All queues have a corresponding DLQ: `{queue-name}-dlq`. Standard queues used unless ordering is critical (FIFO queues used only for `oms-audit-service`).

---

## INFRASTRUCTURE ARCHITECTURE

### Deployment Architecture

```
Internet
    ↓ HTTPS TLS 1.3
Amazon CloudFront
    ↓ /api/** forwarded
AWS API Gateway (HTTP API) — public endpoint
    ↓ VPC Link
ECS Cluster (8 services, private subnets, multi-AZ)
    ↓ service-to-service via Cloud Map DNS (oms.local)
Amazon SNS + SQS (async fan-out, private subnets)
AWS RDS PostgreSQL (Multi-AZ, private subnets, 8 instances)
AWS S3 (audit archival, Glacier Instant Retrieval)
```

### Key AWS Services

| Service | Configuration | Monthly Cost (est.) |
|---|---|---|
| ECS Fargate | 8 services × 2 tasks avg, 0.25–0.5 vCPU, 512–1024 MB | ~$80–130 |
| AWS API Gateway | HTTP API, ~1M req/month | ~$1–3 |
| Amazon CloudFront | Static assets + API proxy | ~$5–15 |
| Lambda Authorizer | ~1M invocations/month (cached 300s TTL) | ~$0.20 |
| Amazon SNS | ~2,500 messages/day | < $1 |
| Amazon SQS | ~2,500 messages/day | < $1 |
| Amazon SES | Email dispatch | ~$0.10/1000 emails |
| AWS RDS | 8 × `db.t3.small` Multi-AZ | ~$300–480 |
| ElastiCache | `cache.t3.micro` (API GW rate-limit state) | ~$15 |
| AWS Cloud Map | Private DNS namespace | ~$1–2 |
| **Total** | | **~$410–650/month** |

> Removing Amazon MQ (`mq.m5.large` at ~$230/month) and Spring Cloud Gateway ECS tasks saves ~$280–350/month vs v3.0.

### CI/CD Pipeline

```
Developer pushes to GitHub (feature branch)
    ↓
Pull Request opened → Jenkins pipeline triggered:
    1. Checkstyle / flake8 lint
    2. JUnit 5 / pytest unit tests
    3. Spring Boot Test + TestContainers / pytest + testcontainers integration tests
    4. Pact consumer contract tests
    5. OWASP Dependency Check / pip-audit (fail CVSS >= 7)
    6. Docker multi-stage build
    7. Push image to AWS ECR (on merge to develop/staging/main)
    8. aws ecs update-service --force-new-deployment
    9. aws ecs wait services-stable

Branches:
  develop → auto-deploy DEV
  staging → auto-deploy STAGING + Pact provider verify
  main    → manual approval gate → PRODUCTION + rollback plan
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

Per request (via Lambda Authorizer):
9.  Request hits CloudFront → forwarded to API Gateway
10. API Gateway invokes Lambda Authorizer (cache TTL: 300s)
11. Lambda Authorizer extracts SESSION cookie → calls auth-service /internal/validate
12. auth-service returns UserContext { userId, roles, locationIds } or 401
13. Lambda Authorizer returns IAM ALLOW policy + context (userId, roles, locationIds)
14. API Gateway injects context as request headers: X-User-Id, X-User-Roles, X-Location-Ids
15. API Gateway generates X-Correlation-ID, injects it as header
16. API Gateway generates internal JWT (signed with INTERNAL_JWT_SECRET), injects Authorization header
17. Request forwarded via VPC Link → Cloud Map DNS → ECS service
18. Service validates internal JWT via Spring Security / python-jose
19. Service performs authoritative role + location check
```

### Security Controls Summary

| Layer | Control |
|---|---|
| CDN | CloudFront WAF (OWASP managed rule set); TLS 1.3 |
| Edge | API Gateway rate limiting (usage plans); CORS enforcement; Lambda Authorizer (300s cache) |
| Identity | SSO only; HTTP-only cookies; immediate session revocation |
| Service-to-Service | Internal JWT (HMAC-SHA256, 300s TTL); validated on every inbound call |
| Authorisation | Role + location check in every service |
| Data | Parameterised SQL; soft deletes; separate DB user per service; INSERT-only for audit |
| Secrets | AWS Secrets Manager; injected as env vars; never in code or git |
| Transport | TLS 1.3 CloudFront→API GW; HTTPS VPC-internal |
| Audit | Immutable audit-service log via SQS consumer; no REST write endpoint |
| Network | VPC private subnets; Cloud Map DNS not publicly resolvable; security groups per service |

---

## OBSERVABILITY

All services emit **structured JSON logs** to **CloudWatch Logs** (`/oms/{service-name}` log group).

```json
{
  "timestamp": "2026-04-17T09:15:00Z",
  "level": "INFO",
  "service": "attendance-service",
  "correlationId": "abc-123",
  "userId": "uuid",
  "message": "Attendance record stamped",
  "recordDate": "2026-04-16",
  "status": "PRESENT"
}
```

| Tool | Purpose |
|---|---|
| CloudWatch Logs | Centralised log aggregation; log groups per service |
| CloudWatch Container Insights | ECS CPU, memory, task health metrics |
| CloudWatch Alarms | Alert on error rate spikes, DLQ depth > 0, ECS task failures |
| CloudWatch Dashboards | Per-service request rates, latency, error counts |

`X-Correlation-ID` generated at API Gateway, propagated via HTTP headers to all downstream services and included as `correlationId` in every SQS message — enables full request tracing via CloudWatch Logs Insights.

---

## ARCHITECTURE DECISION RECORDS (Summary)

Full ADR log is in [adr.md](adr.md). Key decisions for v4.0:

| ADR | Decision | Trade-off |
|---|---|---|
| ADR-005 | AWS Cloud Map over Netflix Eureka | No ECS Eureka service to operate; DNS-based; zero library config |
| ADR-006 | AWS API Gateway (HTTP API) over Spring Cloud Gateway | No ECS gateway service; Lambda Authorizer adds ~5ms cold start (mitigated by 300s cache) |
| ADR-007 | SNS + SQS over RabbitMQ (Amazon MQ) | No broker to operate; SNS fan-out replaces topic exchanges; saves ~$230/month |
| ADR-012 | CloudWatch over Prometheus + Jaeger | No Jaeger ECS service to operate; less granular trace UI; saves ~$15–30/month infra |
| ADR-021 | Polyglot runtimes: Spring Boot + FastAPI | Two build toolchains (Maven + pip); mitigated by shared Docker base image pattern |

---

## NEXT STEPS AND ROADMAP

### Pre-Development Actions (Must complete before M1)

1. Confirm SSO provider JWKS endpoint and client_id/secret — blocks M2
2. Obtain Arms HR API contract and test credentials — blocks M2 (HR sync)
3. Confirm AWS Athena badge event schema and sample data — blocks M4
4. Configure API Gateway VPC Link to Cloud Map namespace
5. Create SNS topics and SQS queues (IaC — Terraform or CDK)
6. Configure Lambda Authorizer deployment and 300s cache
7. Verify SES sending domain (DNS verification) — blocks notification-service

### Phase 1 Build Order

```
M1: Infrastructure
    ECS cluster, SNS topics + SQS queues, 8 RDS instances,
    API Gateway + VPC Link, Cloud Map namespace, Lambda Authorizer,
    CloudFront distribution, CloudWatch dashboards + alarms

M2: auth-service (Spring Boot)
    SSO integration, session management, user profiles, roles, HR sync
    Lambda Authorizer depends on /internal/validate being live

M3: audit-service (Spring Boot) + notification-service (FastAPI)
    SQS consumers must be live before business services publish events

M4: attendance-service (FastAPI)
    Badge pipeline; publishes no_show events consumed by seating-service

M5: seating-service (Spring Boot)
    Consumes attendance events

M6: remote-service (Spring Boot)
    Publishes approved events consumed by attendance-service

M7: Phase 1 hardening, security review, performance testing, UAT, launch
```

### Phase 2 Build Order

```
M8:  workplace-service (Spring Boot) — visitor + event
M9:  inventory-service (Spring Boot) — supplies
M10: inventory-service — assets
M11: Phase 2 hardening and launch
```

### Future Considerations (Post-Launch)

- CQRS per service where read/write traffic diverges significantly
- Redis caching for seat availability if query latency grows
- Real-time badge streaming (replace nightly Athena batch if same-day attendance required)
- Kafka (AWS MSK) if event replay or high-throughput streaming becomes a requirement
- Mobile app (React Native — shares API contracts; no backend changes)
- Multi-region if organisation expands beyond single AWS region

---

*End of System Architecture Document v4.0*  
*See [README.md](README.md) for the full documentation index.*
