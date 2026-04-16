# Backend Documentation
## Office Management System (OMS)

**Version:** 2.0  
**Stack:** Java 21 · Spring Boot 3.x · PostgreSQL · Apache Kafka · AWS SQS · AWS EKS

---

## What is the OMS?

The Office Management System is a cloud-native, microservices platform for managing attendance, seating, remote-day scheduling, visitors, events, supplies, and assets across 2–5 office locations. It is built as **11 independently deployable Spring Boot services**, each owning exactly one business capability and its own PostgreSQL database.

---

## Service Inventory

| Service | Port | Phase | Responsibility |
|---------|------|-------|---------------|
| `identity-service` | 8081 | 1 | SSO / OAuth 2.0 authentication, session lifecycle |
| `user-service` | 8082 | 1 | User profiles, per-location roles, Arms HR sync |
| `attendance-service` | 8083 | 1 | Nightly badge ingestion, WorkSession resolution, AttendanceRecord computation |
| `seating-service` | 8084 | 1 | Floor plans, hot-desk booking, block reservations, no-show auto-release |
| `remote-service` | 8085 | 1 | Remote-day / OOO requests, approval workflow, policy enforcement |
| `notification-service` | 8086 | 1 | Event-driven in-app and email dispatch (Kafka consumer only) |
| `audit-service` | 8087 | 1 | Immutable append-only audit log (Kafka consumer + SQS) |
| `visitor-service` | 8088 | 2 | Visitor pre-registration, check-in/out, agreement versioning |
| `event-service` | 8089 | 2 | Office events, RSVP, waitlist, recurring events |
| `supplies-service` | 8090 | 2 | Supply catalogue, batch stock, two-stage approval Saga |
| `assets-service` | 8091 | 2 | Asset register, assignment lifecycle, fault reporting, maintenance |

---

## Architecture at a Glance

```
Browser / Mobile
       │ HTTPS
       ▼
  API Gateway  ←── validates every session via identity-service /auth/validate
       │ Internal mTLS + JWT
       ▼
  Service Mesh (Istio / AWS App Mesh)
  ├── identity-service  [identity_db]
  ├── user-service      [user_db]
  ├── attendance-service[attendance_db]
  ├── seating-service   [seating_db]
  ├── remote-service    [remote_db]
  ├── visitor-service   [visitor_db]
  ├── event-service     [event_db]
  ├── supplies-service  [supplies_db]
  └── assets-service    [assets_db]
       │
       ▼
  Apache Kafka (AWS MSK)  ←── async events: state changes, workflows, audit, notifications
       │
  ├── notification-service  [notification_db]   ← Kafka + SQS consumer only
  └── audit-service         [audit_db]          ← Kafka + SQS consumer only
```

Three unbreakable rules:
1. **No shared databases** — every service owns its data exclusively.
2. **No direct calls to `notification-service` or `audit-service`** — publish to Kafka/SQS only.
3. **Zero Trust** — every service validates every caller on every request.

---

## Documentation Index

| Document | Contents |
|----------|---------|
| [services.md](services.md) | Detailed definition of all 11 services — responsibilities, owned entities, APIs, Kafka events |
| [api-reference.md](api-reference.md) | Full endpoint listing per service — method, path, roles, request/response shape |
| [data-models.md](data-models.md) | Entity definitions, schema conventions, database indexes |
| [security.md](security.md) | Authentication flow, Zero Trust service-to-service, RBAC, OWASP mitigations |
| [communication.md](communication.md) | REST resilience patterns, Kafka/SQS event contracts, topic registry |
| [observability.md](observability.md) | Structured logging, Prometheus metrics, distributed tracing, health checks, alerting |
| [development.md](development.md) | Project structure, coding standards, naming conventions, testing strategy, git workflow |
| [deployment.md](deployment.md) | Docker, Kubernetes, CI/CD pipeline, infrastructure, disaster recovery |

---

## Quick Start (Local)

### Prerequisites

| Tool | Version |
|------|---------|
| Java | 21 (LTS) |
| Maven | 3.9+ |
| Docker | 25+ |
| Docker Compose | 2.x |

### Run a single service

```bash
# Clone
git clone <repo-url>
cd oms/<service-name>

# Copy and populate environment variables (never commit .env)
cp .env.example .env

# Start with Docker Compose (spins up Postgres + Kafka locally)
docker-compose up --build

# Or run directly (requires Postgres + Kafka running separately)
./mvnw spring-boot:run
```

### Swagger UI

Every service exposes Swagger UI at its local port:

```
http://localhost:808{n}/swagger-ui/index.html
http://localhost:808{n}/v3/api-docs
```

---

## Key Conventions (Summary)

- **Base path:** `/api/v1/` — all APIs versioned from day one.
- **Response envelope:** `{ "success": true/false, "data": {}, "error": null }`
- **All list endpoints paginated** — no unbounded queries.
- **Entities never returned in API responses** — always map to a DTO first.
- **Parameterised SQL only** — no string concatenation ever.
- **Soft deletes** — `deleted_at` timestamp; `NULL` = active.
- **UUID primary keys** — `gen_random_uuid()` / `GenerationType.UUID`.
- **Location-scoped data** — every feature entity carries `location_id`; every query filters by it.

---

## Platform Tickets (Sprint 1 — must complete before any feature work)

| Ticket | Title | Depends On |
|--------|-------|-----------|
| ATT-P01 | SSO / OAuth 2.0 Integration | SSO provider credentials |
| ATT-P02 | Role + Location Interceptor | ATT-P01 |
| ATT-P03 | Audit Log Infrastructure | ATT-P01, ATT-P02 |

---

*See individual files in this directory for full details.*
