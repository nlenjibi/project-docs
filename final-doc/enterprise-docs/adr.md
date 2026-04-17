# Architecture Decision Records (ADR)
## Office Management System (OMS)

**Version:** 3.0  
**Status:** Active  
**Last Updated:** April 2026  

Every major architectural choice is documented here. Format per record:
- **Options Considered** — what was evaluated
- **Decision** — what was chosen
- **Rationale** — why
- **Trade-offs Accepted** — what was sacrificed
- **Consequences** — what this decision forces downstream

---

## ADR-001 — Frontend Framework: React over Angular

**Options Considered:** React, Angular, Vue.js

**Decision:** React (TypeScript)

**Rationale:**
- Angular was the original choice but the team's expertise shifted during planning — the frontend developers have stronger React experience
- React's component model (functional components + hooks) is well-suited to the complex UI requirements (real-time floor plan, dynamic approval workflows, attendance dashboards)
- Larger ecosystem for data visualisation (Recharts, Victory) needed for attendance and inventory reporting
- Faster iteration: React's flexibility allows the team to adopt libraries best-suited to each UI problem without framework opinion

**Trade-offs Accepted:**
- Less opinionated structure than Angular — requires the team to establish its own conventions for state management (Zustand or Redux Toolkit), routing (React Router), and API layer (React Query or SWR)
- No built-in dependency injection (unlike Angular) — manual service layer composition required

**Consequences:** All frontend tooling (bundler, linting, testing) targets the React ecosystem. Backend APIs must expose proper CORS headers for the React dev server.

---

## ADR-002 — Architecture Style: Microservices over Modular Monolith

**Options Considered:** Modular monolith, microservices, serverless

**Decision:** Microservices (8 independently deployable services)

**Rationale:**
- OMS has 8 distinct business capabilities (after merges) with fundamentally different scaling, failure, and deployment requirements:
  - `attendance-service` has a heavy nightly batch job — should not compete with the seat booking API for resources
  - `audit-service` handles high-volume append-only writes — should scale independently
  - `seating-service` serves real-time booking queries — needs independent scaling during peak hours
- Team can own individual services without coordinating deployments across the entire system
- A failure in `inventory-service` (Phase 2) must not affect attendance or seating (Phase 1)

**Trade-offs Accepted:**
- Distributed system complexity: network latency, eventual consistency, distributed tracing required
- More infrastructure to provision and operate (8 databases, 8 service deployments)
- Integration testing across services requires TestContainers or contract testing

**Consequences:** Every design decision is evaluated against: "does this increase coupling between services? If yes — reject."

---

## ADR-003 — Service Merge: 11 → 8 Services

**Options Considered:** Keep all 11 services, merge strategically

**Decision:** Merge to 8 services

**Merges performed:**
1. `identity-service` + `user-service` → `auth-service`
2. `supplies-service` + `assets-service` → `inventory-service`
3. `visitor-service` + `event-service` → `workplace-service`

**Rationale:**

*auth-service merge:*
- `identity-service` called `user-service` synchronously on every single auth validation request — the tightest coupling in the entire system
- `identity-service` owned only one table (sessions) and three endpoints — insufficient to justify a separate service, database, and CI/CD pipeline
- Together they are one bounded context: Identity and Access Management

*inventory-service merge:*
- Both services manage physical office inventory under the same stakeholder (Facilities Admin)
- Identical two-stage approval workflow (Manager → Facilities Admin)
- Both are Phase 2 — simplifies delivery

*workplace-service merge:*
- Both are Phase 2 with the same communication pattern (user REST + notification/audit Kafka only)
- Low individual complexity; merging avoids duplicating boilerplate for minimal gain

**Trade-offs Accepted:**
- Merged services are larger codebases — requires discipline to keep domain modules clean within the service
- `auth-service` cannot scale identity validation independently from user profile management (acceptable at hundreds-of-employees scale)

**Consequences:** 3 fewer RDS instances, 3 fewer CI/CD pipelines, 3 fewer ECS task definitions. Estimated ~20% infrastructure cost reduction.

---

## ADR-004 — Container Orchestration: ECS Fargate over EKS

**Options Considered:** AWS EKS (Kubernetes), AWS ECS Fargate, AWS ECS EC2, self-managed K8s

**Decision:** AWS ECS Fargate

**Rationale:**
- OMS team does not have a dedicated platform engineer to manage a Kubernetes control plane
- EKS adds ~$75/month for the control plane plus significant operational overhead (kubectl, YAML complexity, CNI management, K8s upgrades)
- ECS Fargate provides the same isolation and auto-scaling guarantees without K8s expertise
- Task Definitions, ECS Services, and Service Auto Scaling cover all OMS deployment requirements
- AWS Secrets Manager injection, CloudWatch Logs, and ALB health checks are all native to ECS — zero additional configuration

**Trade-offs Accepted:**
- No Istio service mesh — mTLS must be handled at the application layer (internal JWT + HTTPS within VPC)
- Less ecosystem richness than Kubernetes (no Helm, no K8s-native operators)
- Migrating to EKS later is straightforward (same Docker images) but requires new deployment config

**Cost impact:** ECS Fargate eliminates the $75/month EKS control plane charge. At 8 services × 2 tasks × $0.04048/vCPU-hour (0.25 vCPU), container cost is ~$120/month.

**Consequences:** All deployment manifests are ECS Task Definitions (JSON), not Kubernetes YAML. Auto-scaling uses ECS Service Auto Scaling with Target Tracking policies.

---

## ADR-005 — Service Discovery: Netflix Eureka over AWS Cloud Map

**Options Considered:** Netflix Eureka (Spring Cloud), AWS Cloud Map, Kubernetes DNS (rejected with EKS), hardcoded ALB per service

**Decision:** Netflix Eureka

**Rationale:**
- ECS Fargate tasks get dynamic private IPs on every restart — service discovery is required
- Spring Boot has first-class Eureka support via `spring-cloud-netflix-eureka-client` — two config lines and the service self-registers
- Eureka integrates natively with Spring Cloud Gateway (`lb://service-name` routing) and Spring Cloud OpenFeign (`@FeignClient(name = "auth-service")`) — no hardcoded URLs anywhere in the codebase
- The team's Java/Spring expertise means Eureka is a known quantity

**Trade-offs Accepted:**
- Eureka Server is a Spring Boot application — it must be deployed as an ECS service and maintained as part of the system
- Eureka uses a self-preservation mode that can cause stale registrations under network partitions — requires tuning

**Consequences:** Every service runs a Eureka client. Spring Cloud Gateway resolves all routes via Eureka. OpenFeign clients resolve targets via Eureka. No NLBs needed for inter-service communication.

---

## ADR-006 — API Gateway: Spring Cloud Gateway over AWS API Gateway and Kong

**Options Considered:** Spring Cloud Gateway, AWS API Gateway (REST + HTTP API), Kong (self-hosted)

**Decision:** Spring Cloud Gateway

**Rationale:**

*vs AWS API Gateway:*
- OMS uses HTTP-only session cookies for auth. AWS API Gateway's built-in JWT authorizer only supports Bearer tokens in headers. Session cookie validation requires a Lambda Authorizer — adding ~50–100ms cold-start latency and $0.20/million invocations cost
- AWS API Gateway cannot resolve service names via Eureka — it needs static NLB endpoints for each ECS service (~$16/month per NLB × 8 services = ~$128/month additional)
- Spring Cloud Gateway needs zero extra infrastructure and integrates with Eureka natively

*vs Kong:*
- Kong has no native Eureka integration — would require static upstream definitions or AWS Cloud Map
- Kong introduces a new technology stack (Lua/Go plugins, Admin API) outside the team's Java expertise
- Spring Cloud Gateway is a Spring Boot app — the team already knows how to build, test, and deploy it

**Trade-offs Accepted:**
- Spring Cloud Gateway requires writing and maintaining gateway code (filters, route config) in Java
- Rate limiting requires Redis (ElastiCache) for distributed state across gateway instances
- Gateway is a single potential bottleneck — mitigated by running 2+ ECS tasks with ALB in front

**Consequences:** Gateway is deployed as an ECS service behind a public ALB. Auth filter, correlation ID filter, and rate-limiting filter are written as Spring `GlobalFilter` implementations.

---

## ADR-007 — Message Broker: RabbitMQ (Amazon MQ) over Kafka (AWS MSK) and SQS

**Options Considered:** Apache Kafka via AWS MSK, Amazon SQS + SNS, RabbitMQ via Amazon MQ

**Decision:** RabbitMQ via Amazon MQ (active/standby, `mq.m5.large`)

**Cost comparison:**

| Broker | Monthly Cost (OMS scale) |
|---|---|
| Kafka (MSK, 3-broker m5.large) | ~$463/month |
| RabbitMQ (Amazon MQ, active/standby) | ~$230/month |
| SQS (hundreds of employees, ~2,500 msgs/day) | ~$0.09/month |

**Rationale:**

*RabbitMQ over Kafka:*
- Kafka's primary advantages over RabbitMQ are replay capability and extreme throughput (millions/sec). OMS needs neither at this scale (hundreds of employees, ~2,500 events/day)
- Kafka requires: 3-broker cluster, partition strategy, consumer group offset management, Schema Registry (optional) — significant operational overhead
- RabbitMQ is ~50% cheaper ($230 vs $463/month) for equivalent HA
- DLQs are built-in and auto-configured by Spring Cloud Stream (`auto-bind-dlq: true`)

*RabbitMQ over SQS:*
- SQS has no native fan-out — requires SNS + one SQS queue per consumer per event type → 15 SNS topics + 45 SQS queues for OMS event topology
- SQS has no native Spring Cloud Stream binder — stepping outside the Spring Cloud ecosystem already committed to
- RabbitMQ Topic Exchange delivers to all bound queues natively: publish once, all consumers receive

**Trade-offs Accepted:**
- **Replay is not possible.** Once a message is consumed it is gone. Mitigation: `audit-service` runs with 2 HA tasks and durable queues — if it restarts, unprocessed messages wait in the queue
- Lower throughput ceiling than Kafka (sufficient for OMS; can migrate if scale demands it)
- Requires Eureka Server and RabbitMQ as additional infrastructure components (vs SQS's zero-infra model)

**Consequences:** All async communication uses Spring Cloud Stream with the RabbitMQ binder. Topic Exchanges per domain. One durable queue per service per exchange. DLQ auto-created for every consumer queue.

---

## ADR-008 — API Design: REST over GraphQL

**Options Considered:** REST + JSON, GraphQL (Apollo Federation or GraphQL Mesh)

**Decision:** REST with JSON, versioned at `/api/v1/`

**Rationale:**
- OMS uses Pact consumer-driven contract testing to guarantee API compatibility across 8 services. Pact has no GraphQL support — switching would require replacing the entire contract testing strategy
- The API Gateway mandate is: "zero business logic — authenticate, route, log." GraphQL federation makes the gateway a data orchestration layer (schema stitching, resolver chains) — a direct violation of this principle
- Zero Trust requires every service to validate each caller independently. With REST, the gateway forwards requests with user context headers and each service validates. With GraphQL federation, a resolver chain makes multiple downstream calls in one execution — the auth model becomes complex and hard to audit
- HTTP caching (GET requests on seat availability, user profiles) works naturally with REST; GraphQL's single POST endpoint breaks CDN and browser caching

**The one genuine REST weakness:** The manager dashboard requires parallel calls to attendance, remote, and seating services (API Composition). This is solved with a BFF endpoint — ~50 lines of Java — not a full federation layer.

**Trade-offs Accepted:**
- Dashboard views require API Composition (BFF endpoint or client-side `forkJoin`)
- Some over-fetching on certain list responses

**Consequences:** All APIs follow the same conventions: `/api/v1/` prefix, `{ success, data, error }` response envelope, paginated lists, DTOs only (entities never exposed), OpenAPI 3.0 documentation mandatory.

---

## ADR-009 — Database Per Service: PostgreSQL throughout

**Options Considered:** Shared database, database per service (polyglot), database per service (single technology)

**Decision:** PostgreSQL (AWS RDS) per service — same technology for all 8 services

**Rationale:**
- A shared database creates the tightest possible coupling between services — a schema change in one domain requires coordinating deployments across all services. This directly violates the independence requirement.
- Polyglot persistence (different DB per service based on data model) adds operational overhead without benefit at OMS scale — all services have the same data characteristics (relational, JSONB for snapshots, UUID keys, soft deletes)
- PostgreSQL's JSONB support handles audit state snapshots and raw badge event payloads
- Consistent tooling: Spring Data JPA / Hibernate, Flyway migrations, TestContainers — same stack for all services

**Trade-offs Accepted:**
- 8 separate RDS instances (~$300–500/month combined for db.t3.small)
- Cross-service queries require API Composition (no cross-database joins)

**Consequences:** Each service has its own DB user with minimum privilege. `audit_db` user has INSERT only — no UPDATE, no DELETE at the database level.

---

## ADR-010 — Security Model: Zero Trust with Internal JWT

**Options Considered:** Network-trust (trust internal VPC traffic), mTLS via Istio, application-level internal JWT

**Decision:** Application-level Zero Trust — internal JWT on every service-to-service call, enforced by Spring Security

**Rationale:**
- Istio (the natural mTLS solution) requires Kubernetes — rejected with EKS (ADR-004)
- Network-level trust ("if it's inside the VPC, trust it") is insufficient: a compromised pod inside the VPC could call any service without restriction
- Spring Security's `@oauth2ResourceServer` with a shared internal JWT signing key provides Zero Trust at the application layer with no external dependencies
- AWS security groups provide the network-layer backstop: each service's SG only allows inbound from known sources

**Trade-offs Accepted:**
- Certificate rotation for mTLS is manual (vs Istio's automatic rotation)
- Each service must include Spring Security JWT validation configuration
- Shared JWT signing key must be rotated carefully (rolling update across all services)

**Consequences:** Every service validates `Authorization: Bearer <internal-jwt>` on every inbound call. The API Gateway generates the internal JWT and injects it into forwarded requests. ECS security groups enforce network-level isolation as a second layer.

---

## ADR-011 — Async Communication: Choreography Saga over Orchestration Saga

**Options Considered:** Orchestration-based Saga (central orchestrator service), Choreography-based Saga (event-driven reactions)

**Decision:** Choreography-based Saga

**Rationale:**
- An orchestration saga requires a central coordinator service that knows the state of every step. This coordinator is a single point of failure and becomes a bottleneck for all multi-service workflows
- Choreography keeps each service autonomous — it reacts to events it cares about without any central coordinator knowing about it
- OMS's supply request workflow (PENDING_MANAGER → PENDING_FACILITIES → FULFILLED) flows naturally as a chain of Kafka events where each step publishes the next event

**Trade-offs Accepted:**
- Harder to visualise the overall workflow — requires tracing correlation IDs across multiple service logs
- Compensation (rollback) must be explicitly modelled as compensating events (e.g., `supply.request.rejected`)

**Consequences:** No orchestrator service exists. Each service in a workflow reacts to the previous step's event, processes its local state, and publishes the next event. The full workflow is only visible via distributed tracing (Jaeger).

---

## ADR-012 — Tracing: OpenTelemetry + Jaeger (no AWS X-Ray)

**Options Considered:** AWS X-Ray, OpenTelemetry + Jaeger, Spring Cloud Sleuth (deprecated)

**Decision:** OpenTelemetry SDK (all services) → Jaeger (trace store + UI)

**Rationale:**
- AWS X-Ray is explicitly excluded per team decision
- OpenTelemetry is the industry-standard, vendor-neutral tracing SDK — no lock-in to AWS
- Jaeger is a mature open-source distributed tracing system with a clean UI
- Spring Boot 3.x has native OpenTelemetry support via Micrometer Tracing
- The same OTel collector can export to multiple backends in future (Tempo, Zipkin) without code changes

**Trade-offs Accepted:**
- Jaeger must be deployed and operated (Docker container in dev, ECS task in production)
- No AWS-native tracing integration (X-Ray service map, X-Ray insights)

**Consequences:** Every service includes `opentelemetry-spring-boot-starter`. `X-Correlation-ID` is generated at the gateway and propagated via HTTP headers and RabbitMQ message headers to all downstream calls.

---

## ADR-013 — Session Management: HTTP-Only Cookie over Bearer Token

**Options Considered:** JWT Bearer token (stored in localStorage or memory), HTTP-only session cookie

**Decision:** HTTP-only session cookie managed by `auth-service`

**Rationale:**
- JWTs stored in localStorage are accessible to JavaScript — any XSS vulnerability can steal the token
- HTTP-only cookies cannot be accessed by JavaScript (`document.cookie` returns nothing for HTTP-only cookies) — XSS cannot steal the session
- `spring-session-jdbc` stores sessions server-side in `auth_db` — session revocation is immediate (logout works instantly, not "wait for JWT expiry")
- API Gateway calls `auth-service /validate` on every request — session state is always fresh

**Trade-offs Accepted:**
- Slightly more complex auth flow (API Gateway must forward cookie and call validate endpoint per request)
- CSRF protection required (mitigated by `SameSite=Strict` cookie attribute)
- `auth-service` becomes a dependency on every request — mitigated by circuit breaker

**Consequences:** The React frontend never handles tokens directly. All auth state is managed server-side. The `SameSite=Strict` cookie attribute prevents CSRF. The Spring Cloud Gateway auth filter extracts the cookie and calls `auth-service` before every routed request.

---

## ADR-014 — Notification and Audit: Event-Driven Only (Never Called Directly)

**Options Considered:** Direct REST calls to notification/audit services, event-driven via RabbitMQ

**Decision:** Event-driven via RabbitMQ — notification and audit services are never called directly by REST

**Rationale:**
- If notification and audit were called directly, every business service would need to handle their availability — circuit breakers, retries, fallbacks for non-critical operations
- With event-driven dispatch, a business operation (e.g., booking a seat) publishes an event and returns immediately — it does not wait for notification delivery or audit persistence
- `audit-service`'s INSERT-only database user model requires that write access is restricted to the consumer — no REST endpoint can write audit records directly

**Trade-offs Accepted:**
- Audit records are written slightly after the fact (async) — there is a small window where an action has occurred but is not yet in the audit log
- Notification delivery failures require DLQ monitoring rather than REST error responses

**Consequences:** `notification-service` and `audit-service` expose no write APIs. They subscribe to RabbitMQ queues. All business services publish events; they never import or depend on notification or audit service interfaces.

---

## ADR-015 — Idempotency: processed_events Deduplication Table

**Options Considered:** No deduplication (risk duplicate processing), message ID deduplication table, business-logic idempotency

**Decision:** `processed_events` deduplication table in every consumer

**Rationale:**
- RabbitMQ guarantees at-least-once delivery — duplicate messages can arrive after a consumer crash before acknowledgment
- Every business event carries a UUID `eventId` — checking for this UUID before processing is simple and reliable
- The check is wrapped in the same database transaction as the business operation — deduplication is atomic

**Trade-offs Accepted:**
- `processed_events` table grows over time — requires periodic cleanup of old records
- Adds one SELECT per consumed message (negligible at OMS scale)

**Consequences:** Every `@Bean Consumer<Event>` in every service checks `processedEventRepository.existsById(event.getEventId())` before processing. If found, the message is acknowledged and ignored.

---

## ADR-016 — Attendance Data: Nightly Batch over Real-Time Streaming

**Options Considered:** Real-time badge event streaming, nightly batch via AWS Athena

**Decision:** Nightly batch — `BadgeSyncJob` runs at 2am per location, queries AWS Athena

**Rationale:**
- Stakeholders accepted T+1 day attendance data — real-time was not a stated requirement
- Nightly batch avoids continuous Athena query costs and stream processing infrastructure (Kafka Streams, Spark Streaming)
- The Athena badge event table is already the authoritative source — no parallel streaming infrastructure needed
- Real-time streaming is listed as a future consideration if same-day attendance becomes a requirement

**Trade-offs Accepted:**
- Attendance data is always one day behind — managers cannot see today's attendance in real-time
- Nightly job failure means no attendance data for that night's run — mitigated by alerting and retry on next run

**Consequences:** `attendance-service` runs a scheduled `@Scheduled` job. The job is isolated in a bulkhead thread pool so sync failure does not affect the API request threads.

---

## ADR-017 — Seat Availability: Real-Time Query over Pre-Computed Cache

**Options Considered:** Pre-computed cache (Redis), real-time query from `seat_bookings` table

**Decision:** Real-time query with proper database indexing

**Rationale:**
- At hundreds of seats per location, a properly indexed query on `(booking_date, location_id)` is sub-200ms — no cache required
- A cache would need near-constant invalidation on bookings, cancellations, and no-show releases — adding complexity for no measurable gain at current scale
- Redis can be introduced as a service-internal change later if latency degrades at scale — the architecture does not need to change

**Trade-offs Accepted:**
- Slightly higher per-query database load than a cached approach
- Latency may degrade if seat count per location grows beyond the current scale assumption

**Consequences:** `seating-service` has a compound index on `seat_bookings(booking_date, location_id) WHERE deleted_at IS NULL`. Availability is always accurate — no cache invalidation logic needed.

---

## ADR-018 — Configuration Management: Environment Variables Only

**Options Considered:** Spring Cloud Config Server, AWS AppConfig, environment variables in ECS Task Definitions, YAML files in source control

**Decision:** Environment variables via ECS Task Definitions + AWS Secrets Manager

**Rationale:**
- 12-Factor App principle: configuration in the environment, not code
- Secrets (DB passwords, JWT signing key, RabbitMQ credentials) are stored in AWS Secrets Manager and injected as environment variables at task startup — never in source control
- Non-secret config (CRON schedules, thresholds, timeouts) is managed in ECS Task Definition environment variables — visible in the AWS console, auditable via CloudTrail
- Avoids running a Config Server as another dependency

**Trade-offs Accepted:**
- No runtime config refresh — changing a config value requires a new task deployment
- Config is spread across ECS Task Definitions rather than centralised in one place

**Consequences:** No `.properties` or `.yml` files contain sensitive values. CI/CD pipelines read secrets from AWS Secrets Manager and inject them into deployments. All config keys are documented per service.

---

## ADR-019 — Testing Strategy: Pact Contract Testing

**Options Considered:** Full integration test environment (deploy all services), Pact consumer-driven contract tests, manual API testing

**Decision:** Pact consumer-driven contract tests in CI, supplemented by TestContainers integration tests

**Rationale:**
- Full integration environments for 8 services are expensive to maintain and slow to spin up in CI
- Pact allows each consumer service to define a contract (what it expects from a provider) and each provider to verify that contract independently — no shared environment required
- TestContainers provides real PostgreSQL and RabbitMQ instances for integration tests — no mocking of infrastructure

**Trade-offs Accepted:**
- Pact tests require careful setup — consumers write contracts, providers must run contract verifications in their CI pipeline
- Contract tests do not replace end-to-end tests entirely — a Postman collection is maintained for full flow testing

**Consequences:** Every inter-service REST call has a Pact contract. Every service's CI pipeline includes `pact:verify` against the Pact Broker. Breaking API changes are caught before deployment.

---

## ADR-020 — Role Scoping: Location-Scoped RBAC

**Options Considered:** Global roles (one role per user), location-scoped roles (role + location per user)

**Decision:** Location-scoped roles — every `UserRole` record links a user, a role enum, and a `location_id`

**Rationale:**
- An organisation with 2–5 offices commonly has employees who are Managers in one location and Employees in another
- Global roles would either over-privilege (Manager everywhere) or under-privilege (Employee everywhere) multi-location users
- Super Admin is the only exception: their `UserRole` records have `NULL` `location_id` — the system interprets this as cross-location access

**Trade-offs Accepted:**
- Every authorisation check requires two dimensions: role AND location — slightly more complex than a simple role check
- API Gateway and every service must enforce both dimensions

**Consequences:** Role check pattern is always: `hasRole(X) && hasLocationAccess(locationId)`. The `auth-service` returns the full role+location list in the session. API Gateway injects `X-User-Roles` and `X-Location-Id` headers. Each service performs the authoritative check internally.

---

## Summary Table

| ADR | Decision | Key Trade-off |
|---|---|---|
| 001 | React frontend | Less opinionated — team sets conventions |
| 002 | Microservices | Distributed complexity accepted |
| 003 | 8 services (merged from 11) | Larger merged codebases |
| 004 | ECS Fargate | No Istio — app-level mTLS instead |
| 005 | Netflix Eureka | Eureka Server is another service to run |
| 006 | Spring Cloud Gateway | Requires Java filter code and Redis for rate limiting |
| 007 | RabbitMQ (Amazon MQ) | No event replay (vs Kafka) |
| 008 | REST not GraphQL | Dashboard composition via BFF endpoint |
| 009 | PostgreSQL per service | 8 RDS instances, no cross-service joins |
| 010 | Zero Trust — internal JWT | No Istio; manual cert rotation |
| 011 | Choreography Saga | Harder to visualise full workflow |
| 012 | OpenTelemetry + Jaeger | Jaeger must be deployed and operated |
| 013 | HTTP-only session cookie | CSRF protection required; validate on every request |
| 014 | Notification/audit event-driven only | Audit is slightly async after the fact |
| 015 | processed_events deduplication | Table grows; needs periodic cleanup |
| 016 | Nightly badge sync | T+1 attendance data |
| 017 | Real-time seat availability | No cache; DB must be properly indexed |
| 018 | Env vars + Secrets Manager | Config change requires redeployment |
| 019 | Pact contract testing | Pact infrastructure required in CI |
| 020 | Location-scoped RBAC | Every auth check needs role + location |
