# Office Management System (OMS) — Documentation Index

**Version:** 3.0 — Spring Cloud Edition  
**Status:** Draft  
**Last Updated:** April 2026  
**Frontend:** React  
**Backend:** Java 21 + Spring Boot 3.x + Spring Cloud  
**Broker:** RabbitMQ (Amazon MQ)  
**Orchestration:** AWS ECS Fargate  

---

## What Is OMS?

The Office Management System (OMS) is a cloud-native, microservices platform that replaces manual, spreadsheet-driven office operations with a unified, location-aware system. It serves employees, managers, HR, and facilities staff across 2–5 office locations.

The platform is composed of **8 independently deployable Spring Boot services**, each owning exactly one business capability and its own PostgreSQL database. All external traffic enters through a Spring Cloud Gateway. Services discover each other via Netflix Eureka. Asynchronous communication uses RabbitMQ (Amazon MQ) with Topic Exchanges and durable queues.

---

## Technology Stack at a Glance

| Layer | Technology | Why |
|---|---|---|
| Frontend | React (TypeScript) | Component model, ecosystem, team expertise |
| Backend | Java 21, Spring Boot 3.x | Team expertise, Spring Cloud ecosystem |
| API Gateway | Spring Cloud Gateway | Native Eureka integration, Java-native auth filter |
| Service Discovery | Netflix Eureka | Native Spring Boot support, dynamic ECS IP resolution |
| Inter-service REST | Spring Cloud OpenFeign + Resilience4j | Declarative clients, circuit breakers, Eureka resolution |
| Messaging | RabbitMQ via Amazon MQ | Fan-out, Spring Cloud Stream binder, lower cost than Kafka |
| Messaging Abstraction | Spring Cloud Stream (RabbitMQ binder) | Decouples business code from broker |
| Databases | PostgreSQL via AWS RDS (8 instances) | One per service, relational, JSONB for audit snapshots |
| Containerisation | Docker | Consistent builds across all environments |
| Orchestration | AWS ECS Fargate | Simpler than Kubernetes, no control plane ops |
| Secrets | AWS Secrets Manager | Runtime injection into ECS Task Definitions |
| Logging | CloudWatch Logs (awslogs driver) | Native ECS integration, zero setup |
| Metrics | Prometheus + Grafana | Spring Actuator `/actuator/prometheus` on every service |
| Tracing | OpenTelemetry + Jaeger | Distributed trace correlation across all services |
| CI/CD | GitHub Actions + AWS ECR | Per-service independent pipelines |

---

## Service Inventory

| Service | Port | Phase | Replaces | Responsibility |
|---|---|---|---|---|
| `api-gateway` | 8080 | 1 | — | Routing, auth, rate limiting |
| `eureka-server` | 8761 | 1 | — | Service registry |
| `auth-service` | 8081 | 1 | identity + user | SSO, sessions, user profiles, roles, HR sync |
| `attendance-service` | 8083 | 1 | attendance | Badge ingestion, WorkSession, AttendanceRecord |
| `seating-service` | 8084 | 1 | seating | Floor plans, hot-desk booking, no-show release |
| `remote-service` | 8085 | 1 | remote | Remote/OOO requests, approval workflow, policy |
| `notification-service` | 8086 | 1 | notification | Event-driven in-app + email dispatch |
| `audit-service` | 8087 | 1 | audit | Immutable append-only audit log |
| `inventory-service` | 8090 | 2 | supplies + assets | Physical office inventory, two-stage approvals |
| `workplace-service` | 8088 | 2 | visitor + event | Visitor lifecycle, office events, RSVP |

---

## Documentation Map

### Architecture

| Document | Description |
|---|---|
| [architecture-system.md](architecture-system.md) | Full system architecture — requirements, domain model, service definitions, API design, infrastructure, security |
| [solution-system.md](solution-system.md) | 5-Layer solution architecture — business why, capability what, logical how, physical technology, execution |
| [adr.md](adr.md) | Architecture Decision Records — every major decision with options considered, rationale, and trade-offs accepted |

### Infrastructure & Operations

| Document | Description |
|---|---|
| [infrastructure.md](infrastructure.md) | ECS Fargate, Amazon MQ, RDS, ALB, CI/CD pipelines, deployment, disaster recovery |
| [observability.md](observability.md) | Logging, metrics, tracing, alerting, dashboards |
| [security.md](security.md) | Zero Trust, JWT auth, RBAC, data security, OWASP mitigations |

### Development Reference

| Document | Description |
|---|---|
| [communication.md](communication.md) | RabbitMQ exchanges/queues, Spring Cloud Stream config, OpenFeign patterns, Resilience4j |
| [api-reference.md](api-reference.md) | Full API contracts for all 8 services |
| [data-models.md](data-models.md) | Entity definitions, schema conventions, database indexes |
| [cost-analysis.md](cost-analysis.md) | AWS cost breakdown, trade-off justifications, optimization strategies |

---

## Local Development Quick Start

```bash
# Prerequisites: Docker Desktop, Java 21, Node 20+

# 1. Start infrastructure (Eureka, RabbitMQ, PostgreSQL databases)
docker compose up -d

# 2. Verify Eureka dashboard
open http://localhost:8761

# 3. Verify RabbitMQ management UI
open http://localhost:15672   # guest / guest

# 4. Start services (each in a separate terminal, or via IDE run configs)
cd auth-service && ./mvnw spring-boot:run
cd attendance-service && ./mvnw spring-boot:run
cd seating-service && ./mvnw spring-boot:run
cd remote-service && ./mvnw spring-boot:run
cd notification-service && ./mvnw spring-boot:run
cd audit-service && ./mvnw spring-boot:run

# 5. All traffic via gateway
open http://localhost:8080/api/v1/auth/login

# 6. Start React frontend
cd frontend && npm install && npm start
open http://localhost:3000
```

---

## Key Architectural Rules (Non-Negotiable)

1. **No shared databases.** Each service has its own PostgreSQL instance. No service reads another service's database.
2. **No direct calls to notification or audit services.** These services are triggered by RabbitMQ events only. No REST endpoint may write to them.
3. **Zero Trust.** Every inbound request to every service — from the gateway or another service — is validated via internal JWT and role/location check. Network position grants no trust.
4. **Parameterised SQL only.** No string concatenation in database queries. Ever.
5. **Idempotent consumers.** Every RabbitMQ consumer checks a `processed_events` table before processing. Duplicate delivery is safe.
6. **Pagination everywhere.** No list endpoint returns unbounded results.
7. **DTOs, not entities.** Entities never leave the service layer. Responses always use DTOs.

---

## Roles and Permissions Summary

| Role | Scope | Key Capabilities |
|---|---|---|
| Employee | Own data only | Self-service: attendance view, seat booking, remote requests, notifications |
| Manager | Team within location | All Employee + team visibility + approval authority |
| HR | Location-wide | All Manager + org-wide reporting + employee lifecycle |
| Facilities Admin | Location-wide | Space config, supply/asset management, visitor reception |
| Super Admin | All locations | Full system configuration, cross-location oversight |

---

## RabbitMQ Exchange Map

| Exchange | Type | Key Producers | Key Consumers |
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

*For questions or corrections, raise a PR against this repository.*
