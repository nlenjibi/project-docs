# Backend Documentation
## Office Management System (OMS)

**Version:** 3.0
**Last Updated:** 2026-04-17
**Stack:** Java 21 · Spring Boot 3.x · Spring Cloud 2023.x · PostgreSQL · RabbitMQ (Amazon MQ) · AWS ECS Fargate · Netflix Eureka · Spring Cloud Gateway · React (TypeScript)

---

## What is the OMS?

The Office Management System is a cloud-native, microservices platform that manages attendance, seating, remote-day scheduling, visitor management, office events, supply requests, and asset tracking across 2–5 office locations. It is built as **8 independently deployable Spring Boot services**, each owning exactly one business capability and its own PostgreSQL database.

**Why 8 services?** An earlier design used 11 services. Three merges were made after coupling analysis:

| Merged From | Merged Into | Rationale |
|-------------|-------------|-----------|
| `identity-service` + `user-service` | `auth-service` | Identity always calls user to resolve roles at login. They share the same data lifecycle, the same SSO dependency, and the same deployment cadence. Separating them added a synchronous hop on every single request with no independent scalability benefit. |
| `supplies-service` + `assets-service` | `inventory-service` | Both manage physical items at a location. They share the same approval workflow pattern, the same Facilities Admin role, the same stock/register paradigm. A single `inventory_db` avoids cross-service inventory joins. Merging saves ~$1,440/year in RDS costs. |
| `visitor-service` + `event-service` | `workplace-service` | Both are Phase 2, location-scoped, low-traffic features driven by Facilities Admin and HR. Shared deployment, shared database, shared agreement template infrastructure. Merging halves the operational overhead for two services that will rarely need independent scaling. |

---

## Service Inventory

| Service | Port | Phase | Primary Responsibility |
|---------|------|-------|----------------------|
| `auth-service` | 8081 | 1 | SSO / OAuth 2.0 authentication, session lifecycle, user profiles, per-location roles, HR sync |
| `attendance-service` | 8082 | 1 | Nightly badge ingestion (AWS Athena), WorkSession resolution, two-pass AttendanceRecord computation |
| `seating-service` | 8083 | 1 | Floor plans, hot-desk booking, block reservations, no-show auto-release |
| `remote-service` | 8084 | 1 | Remote-day / OOO requests, approval workflow, HARD_BLOCK / SOFT_WARNING policy enforcement |
| `notification-service` | 8085 | 1 | Event-driven in-app and email dispatch (RabbitMQ consumer only — never called directly via REST) |
| `audit-service` | 8086 | 1 | Immutable append-only audit log (RabbitMQ consumer only — INSERT privilege only at DB level) |
| `inventory-service` | 8087 | 2 | Supply catalogue, stock management, two-stage supply request Saga, asset register, assignment lifecycle, fault reporting |
| `workplace-service` | 8088 | 2 | Visitor pre-registration, check-in/out, agreement versioning, office events, RSVP, waitlist |

---

## Architecture at a Glance

```
Browser / Mobile (React SPA)
       │ HTTPS  (HTTP-only session cookie on every request)
       ▼
  Spring Cloud Gateway  (Port 8080)
  │  ─ Validates session cookie via auth-service /internal/validate
  │  ─ Injects X-User-Id, X-User-Roles, X-Location-Id headers
  │  ─ Generates X-Correlation-ID for every inbound request
  │  ─ Routes via lb:// prefix → Eureka service discovery
       │ Internal HTTP + JWT (HMAC-SHA256)
       ▼
  Netflix Eureka  (Service Registry — Port 8761)
  ├── auth-service        [auth_db]
  ├── attendance-service  [attendance_db]
  ├── seating-service     [seating_db]
  ├── remote-service      [remote_db]
  ├── notification-service[notification_db]
  ├── audit-service       [audit_db]
  ├── inventory-service   [inventory_db]
  └── workplace-service   [workplace_db]
       │
       ▼
  Amazon MQ (RabbitMQ)  ←── async events: state changes, workflows, audit, notifications
       │
  ├── notification-service  ← RabbitMQ consumer only (never receives REST calls)
  └── audit-service         ← RabbitMQ consumer only (INSERT-only DB privilege)
```

### Three Unbreakable Rules

1. **No shared databases** — every service owns its data exclusively. No cross-database SQL joins. Ever.
2. **No direct calls to `notification-service` or `audit-service`** — publish to RabbitMQ only. These two services react independently.
3. **Zero Trust** — every service validates the internal JWT on every request, regardless of origin.

---

## Technology Decisions and Trade-offs

### Why Spring Cloud Gateway over AWS API Gateway or Kong?

| Option | Decision | Reason |
|--------|----------|--------|
| **Spring Cloud Gateway** | ✅ Chosen | Native Eureka `lb://` routing. Session cookie auth filter in Java without Lambda Authorizer. Same Spring ecosystem — no new tooling. Full control over headers and filters. |
| AWS API Gateway | Rejected | Cannot route to Eureka-registered ECS tasks directly. Session cookie validation requires a Lambda Authorizer (extra latency, extra cost). Vendor lock-in on routing config. |
| Kong | Rejected | No native Eureka integration. Requires a separate Kong data plane cluster. Adds operational complexity and a new technology stack for the team to learn and maintain. |

### Why RabbitMQ (Amazon MQ) over Apache Kafka (MSK) or AWS SQS?

| Criterion | RabbitMQ | Kafka (MSK) | SQS |
|-----------|----------|-------------|-----|
| Spring Cloud Stream support | ✅ First-class binder | ✅ First-class binder | ❌ No official binder (SNS/SQS complexity) |
| Message replay | Not needed for OMS | ✅ (overkill — adds ~$200/mo) | ❌ |
| Fan-out to N consumers | ✅ Topic Exchange | ✅ Consumer groups | Requires SNS + multiple queues |
| Dead-letter queues | ✅ Native | ✅ | ✅ |
| Managed hosting cost | ~$30/mo (mq.m5.large) | ~$200–350/mo (MSK 2-broker) | Pay-per-message (~$20/mo at OMS volume) |
| Operational complexity | Low | High (partition management, consumer lag tuning) | Low |
| **Verdict** | ✅ Best fit | ❌ Cost vs benefit | ❌ No Spring Cloud Stream binder |

**Annual saving vs Kafka MSK:** ~$2,040–3,840/year.

### Why ECS Fargate over EKS (Kubernetes)?

| Criterion | ECS Fargate | EKS (Kubernetes) |
|-----------|-------------|-----------------|
| Cluster management | Zero — AWS managed | You manage control plane + node groups |
| Eureka registration | Works via ECS service discovery or direct registration | Works but adds complexity |
| Deployment YAML complexity | Simple task definition JSON | Full Kubernetes manifests, RBAC, namespaces, Helm charts |
| Team ramp-up | Days | Weeks–months for production-grade K8s |
| Cost (control plane) | $0 | ~$75/month (EKS control plane) |
| Auto-scaling | ECS Service Auto Scaling on CPU/memory | HPA (CPU), KEDA for advanced metrics |
| **Verdict** | ✅ Right for OMS team size | ❌ Operational overhead unjustified at this scale |

**Annual saving vs EKS:** ~$900/year (control plane alone) + engineering time.

### Why Netflix Eureka over AWS Cloud Map or DNS-based discovery?

ECS Fargate assigns dynamic IPs to tasks. Spring Cloud Gateway uses `lb://service-name` routing which requires a service registry to resolve live instances. Eureka is the native solution in the Spring Cloud ecosystem. AWS Cloud Map requires additional SDK integration and does not integrate with Spring Cloud LoadBalancer's `@LoadBalancerClient` pattern. DNS-based discovery has no health-check-driven removal. Eureka was already the team's known tool from previous Spring Cloud work.

---

## Documentation Index

| Document | Contents |
|----------|---------|
| [services.md](services.md) | Detailed definition of all 8 services — responsibilities, owned entities, APIs, RabbitMQ events, trade-offs |
| [api-reference.md](api-reference.md) | Full endpoint listing per service — method, path, roles, request/response shapes, error codes |
| [data-models.md](data-models.md) | Entity definitions, schema conventions, full SQL DDL, database indexes |
| [security.md](security.md) | Authentication flow, Zero Trust service-to-service, RBAC, OWASP mitigations, React frontend security |
| [communication.md](communication.md) | Feign client resilience patterns, RabbitMQ exchange/queue registry, Spring Cloud Stream config, Saga choreography |
| [cross-service-queries.md](cross-service-queries.md) | Every cross-service data dependency — patterns used (local read-model, API composition, event-driven), implementation |
| [observability.md](observability.md) | Structured logging, Prometheus metrics, OpenTelemetry + Jaeger tracing, health checks, alerting |
| [development.md](development.md) | Project structure, coding standards, naming conventions, testing strategy, git workflow |
| [deployment.md](deployment.md) | ECS Fargate task definitions, CI/CD pipeline, infrastructure, disaster recovery, cost breakdown |

---

## Quick Start (Local Development)

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Java | 21 (LTS) | Service runtime |
| Maven | 3.9+ | Build tool |
| Docker | 25+ | Containerisation |
| Docker Compose | 2.x | Local infrastructure (Postgres, RabbitMQ, Eureka) |

### Run a Single Service

```bash
# Clone the repo
git clone <repo-url>
cd oms/<service-name>

# Copy environment template — NEVER commit .env
cp .env.example .env

# Start with Docker Compose (spins up Postgres + RabbitMQ + Eureka locally)
docker-compose up --build

# Or run directly (requires infrastructure running separately)
./mvnw spring-boot:run
```

### Local Infrastructure (shared docker-compose.yml at repo root)

```yaml
services:
  postgres-auth:
    image: postgres:16
    environment: { POSTGRES_DB: auth_db, POSTGRES_USER: auth_svc, POSTGRES_PASSWORD: local_dev }
    ports: ["5432:5432"]

  rabbitmq:
    image: rabbitmq:3.13-management
    ports: ["5672:5672", "15672:15672"]   # AMQP + management UI

  eureka:
    image: springcloud/eureka   # or build locally
    ports: ["8761:8761"]
```

RabbitMQ management UI: `http://localhost:15672` (guest/guest in local only)
Eureka dashboard: `http://localhost:8761`

### Swagger UI

Every service exposes Swagger UI at its local port:

```
http://localhost:808{n}/swagger-ui/index.html
http://localhost:808{n}/v3/api-docs
```

---

## Key Conventions (Summary)

| Convention | Rule |
|------------|------|
| Base path | `/api/v1/` — all APIs versioned from day one |
| Response envelope | `{ "success": true/false, "data": {}, "error": null }` |
| All list endpoints | Paginated — `?page=0&size=20&sort=createdAt,desc` — never unbounded |
| Entity exposure | Entities are NEVER returned in API responses — always map to a DTO first |
| SQL safety | Parameterised SQL only — no string concatenation, ever |
| Deletes | Soft deletes (`deleted_at` timestamp) — hard deletes are forbidden |
| Primary keys | UUID (`gen_random_uuid()`) — never sequential integers |
| Location scoping | Every feature entity carries `location_id`; every query filters by it |
| Events | Published AFTER the database transaction commits — never before |
| Fallbacks | Every Feign call has a circuit breaker + a defined fallback — no fallback-less Feign calls |
| Internal JWT | Every service validates the internal JWT on every inbound request |

---

## Platform Tickets

These must be completed before any feature service work begins. Grouped by concern.

### Infra — AWS & ECS

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-I01 | Provision ECS Fargate cluster (dev / staging / prod) | AWS account |
| OMS-I02 | Create VPC, private subnets, ALB, security groups | OMS-I01 |
| OMS-I03 | Create AWS ECR repositories (one per service + gateway + eureka) | OMS-I01 |
| OMS-I04 | Provision 8 RDS PostgreSQL instances (one per service, multi-AZ prod) | OMS-I02 |
| OMS-I05 | Store all secrets in AWS Secrets Manager; create ECS execution role | OMS-I04 |

### Eureka Server

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-E01 | Create `eureka-server` Spring Boot project with `spring-cloud-netflix-eureka-server` | — |
| OMS-E02 | Configure `preferIpAddress: true`, self-preservation, peer replication | OMS-E01 |
| OMS-E03 | Write ECS task definition + ECS Service (2 tasks, private subnet) | OMS-I01, OMS-E01 |
| OMS-E04 | Deploy Eureka to dev; verify dashboard at `http://eureka.oms.internal:8761` | OMS-E03 |

### Spring Cloud Gateway

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-G01 | Create `api-gateway` project; add `spring-cloud-starter-gateway` + Eureka client | OMS-E04 |
| OMS-G02 | Configure `lb://` routes for all 8 services in `application.yml` | OMS-G01 |
| OMS-G03 | Implement `SessionAuthFilter` — calls `auth-service /internal/validate`; injects `X-User-Id`, `X-User-Roles`, `X-Location-Id` | OMS-G01 |
| OMS-G04 | Implement `CorrelationIdFilter` — generates `X-Correlation-ID` on every inbound request | OMS-G01 |
| OMS-G05 | Add `RequestRateLimiter` (100 req/s per user) | OMS-G01 |
| OMS-G06 | Write ECS task definition + Service; attach to ALB target group | OMS-I02, OMS-G03 |
| OMS-G07 | Deploy gateway to dev; verify routing to each service | OMS-E04, OMS-G06 |

### Amazon MQ (RabbitMQ)

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-M01 | Provision Amazon MQ broker (`mq.m5.large`, `ACTIVE_STANDBY_MULTI_AZ`) in private subnet | OMS-I02 |
| OMS-M02 | Create vhost `/oms`, service users, and store credentials in Secrets Manager | OMS-M01, OMS-I05 |
| OMS-M03 | Declare Topic Exchange `oms.events`, all queues, DLX `oms.dlx`, and all DLQs | OMS-M01, OMS-M02 |
| OMS-M04 | Verify queue bindings and DLQ routing with a test publisher | OMS-M03 |

### Auth Service (SSO + User + Identity)

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-A01 | Bootstrap `auth-service`; Flyway schema for `auth_db` (users, user_roles, locations, sessions) | OMS-I04 |
| OMS-A02 | Implement OIDC callback flow — exchange code, verify JWKS, issue session cookie | OMS-A01 |
| OMS-A03 | Implement `GET /internal/validate` — resolve session → inject user context headers | OMS-A02 |
| OMS-A04 | Implement `GET /api/v1/auth/me`, logout, user profile, role, location APIs | OMS-A02 |
| OMS-A05 | Implement `LocationRoleInterceptor` + `@RequiresRole` annotation | OMS-A01 |
| OMS-A06 | Implement `HrSyncJob` — Arms API poll, upsert users/roles, publish RabbitMQ events | OMS-A01, OMS-M03 |
| OMS-A07 | Register with Eureka; deploy to ECS dev | OMS-E04, OMS-A03 |

### Internal JWT (Zero Trust)

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-J01 | Implement `InternalJwtFilter` in `auth-service` — attach HMAC-SHA256 JWT to every outgoing Feign call | OMS-A02 |
| OMS-J02 | Configure `SecurityFilterChain` + `internalJwtDecoder()` in every service | OMS-J01 |
| OMS-J03 | Add `CorrelationIdHolder` (MDC ThreadLocal) and Feign `RequestInterceptor` in every service | OMS-J01 |

### Audit Service

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-AU01 | Bootstrap `audit-service`; Flyway schema for `audit_db` (INSERT-only DB user) | OMS-I04 |
| OMS-AU02 | Implement `oms.audit.event` RabbitMQ consumer (idempotent, `processed_events` table) | OMS-AU01, OMS-M03 |
| OMS-AU03 | Implement `GET /api/v1/audit-logs` with all filter params and local `user_summaries` JOIN | OMS-AU02 |
| OMS-AU04 | Implement `AuditEventPublisher` (shared library or copy) — wire into every service | OMS-M03 |

### CI/CD Pipeline

| Ticket | Task | Depends On |
|--------|------|-----------|
| OMS-C01 | Create GitHub Actions workflow per service: lint → unit → integration → Pact → build → push ECR | OMS-I03 |
| OMS-C02 | Add `deploy-dev` job (auto on `develop` merge); `deploy-staging` job; `deploy-prod` with manual approval | OMS-C01 |
| OMS-C03 | Add OWASP dependency-check step to CI pipeline | OMS-C01 |

---

*See individual files in this directory for full implementation details.*
