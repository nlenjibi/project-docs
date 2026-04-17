# Infrastructure & Deployment
## Office Management System (OMS)

**Version:** 3.0 | **Platform:** AWS | **Orchestration:** ECS Fargate

---

## 1. Infrastructure Overview

```
Internet / Corporate Network
        ↓ HTTPS / TLS 1.3
[AWS ALB — public-facing]
  TLS termination · HTTPS redirect · Health check routing
        ↓
[Spring Cloud Gateway — 2 ECS Tasks, port 8080]
  Session validation → auth-service (Eureka)
  Route: lb://service-name (Eureka resolution)
  Rate limiting → Redis (ElastiCache)
  Correlation ID injection
        ↓ HTTPS internal (VPC private subnets)
┌────────────────────────────────────────────────┐
│           ECS Cluster: oms-cluster              │
│                                                 │
│  eureka-server        (2 tasks — HA)  :8761    │
│  auth-service         (2–10 tasks)    :8081    │
│  attendance-service   (2–6 tasks)     :8083    │
│  seating-service      (2–10 tasks)    :8084    │
│  remote-service       (2–6 tasks)     :8085    │
│  notification-service (2–6 tasks)     :8086    │
│  audit-service        (2–8 tasks)     :8087    │
│  inventory-service    (2–6 tasks)     :8090    │ ← Phase 2
│  workplace-service    (2–6 tasks)     :8088    │ ← Phase 2
└──────────────┬────────────────────────────────┘
               │
   ┌───────────┼──────────────────┐
   │           │                  │
Amazon MQ   AWS RDS           AWS S3
RabbitMQ    PostgreSQL        audit archival
active/     8 isolated        >24 months
standby     instances         Glacier IR
```

---

## 2. AWS Services Used

| Service | Purpose | Configuration |
|---|---|---|
| **ECS Fargate** | Container orchestration | `oms-cluster`, 8 services + gateway + Eureka |
| **ALB** | Public-facing load balancer | HTTPS :443 → gateway :8080 |
| **Amazon MQ** | RabbitMQ broker | Active/standby, `mq.m5.large`, multi-AZ |
| **AWS RDS** | PostgreSQL databases | 8 instances, `db.t3.small`, Multi-AZ |
| **ECR** | Docker image registry | One repository per service |
| **Secrets Manager** | Runtime secrets injection | DB passwords, JWT key, RabbitMQ credentials |
| **ElastiCache (Redis)** | Gateway rate-limiting state | `cache.t3.micro`, single-node |
| **S3** | Audit log archival | Glacier Instant Retrieval |
| **CloudWatch Logs** | Centralised log aggregation | `/oms/{service-name}` log group per service |
| **Certificate Manager** | TLS certificates | ALB HTTPS termination |
| **VPC** | Network isolation | Private subnets for all services; public subnet for ALB only |

---

## 3. ECS Task Definitions

Every service shares the same pattern. Differences are in image name, port, CPU/memory, and environment variables.

### Standard Task Definition (example: seating-service)

```json
{
  "family": "oms-seating-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "seating-service",
      "image": "ACCOUNT.dkr.ecr.eu-west-1.amazonaws.com/oms/seating-service:v1.0.0",
      "portMappings": [{ "containerPort": 8084, "protocol": "tcp" }],
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
        { "name": "EUREKA_CLIENT_SERVICEURL_DEFAULTZONE", "value": "http://eureka.oms.internal:8761/eureka/" },
        { "name": "RABBITMQ_HOST", "value": "b-xxx.mq.eu-west-1.amazonaws.com" },
        { "name": "BOOKING_WINDOW_DAYS", "value": "14" },
        { "name": "NO_SHOW_RELEASE_CRON", "value": "0 10 * * *" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT:secret:oms/seating/db-password" },
        { "name": "RABBITMQ_PASSWORD", "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT:secret:oms/rabbitmq/password" },
        { "name": "INTERNAL_JWT_SECRET", "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT:secret:oms/internal-jwt-secret" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8084/actuator/health/readiness || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/oms/seating-service",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### CPU and Memory Per Service

| Service | CPU | Memory | Reason |
|---|---|---|---|
| `api-gateway` | 512 | 1024 MB | All traffic passes through; higher load |
| `eureka-server` | 256 | 512 MB | Registry only; low compute |
| `auth-service` | 256 | 512 MB | Stateless JWT validation; low compute |
| `attendance-service` | 512 | 1024 MB | Nightly batch job is CPU/memory intensive |
| `seating-service` | 256 | 512 MB | Real-time queries; moderate |
| `remote-service` | 256 | 512 MB | Policy engine; moderate |
| `notification-service` | 256 | 512 MB | Kafka consumer + email dispatch |
| `audit-service` | 256 | 512 MB | High write volume; append-only |
| `inventory-service` | 256 | 512 MB | Phase 2; similar to remote |
| `workplace-service` | 256 | 512 MB | Phase 2; low compute |

---

## 4. ECS Service Auto Scaling

Every service uses Target Tracking on `ECSServiceAverageCPUUtilization` at 70%.

```
Policy: Target Tracking
Metric: ECSServiceAverageCPUUtilization
Target: 70%
Scale-out cooldown: 60s
Scale-in cooldown: 300s

Per-service limits:
  api-gateway:          min 2, max 8
  auth-service:         min 2, max 10
  attendance-service:   min 2, max 6 (+ scheduled scale-up before nightly sync)
  seating-service:      min 2, max 10
  remote-service:       min 2, max 6
  notification-service: min 2, max 6
  audit-service:        min 2, max 8
```

### Scheduled Scaling for attendance-service

```
Scale-up action:  01:45 UTC — set desired count to 4
Scale-down action: 04:00 UTC — return to min 2
```

The nightly badge sync job runs at 02:00 UTC. Pre-scaling ensures compute is available before the job starts.

---

## 5. Networking and Security Groups

All services run in **private subnets**. Only the ALB is in a public subnet.

```
VPC: 10.0.0.0/16

Public subnets (ALB only):
  10.0.1.0/24  eu-west-1a
  10.0.2.0/24  eu-west-1b

Private subnets (all ECS tasks + RDS + Amazon MQ):
  10.0.10.0/24  eu-west-1a
  10.0.11.0/24  eu-west-1b
  10.0.12.0/24  eu-west-1c
```

**Security Group Rules:**

| Security Group | Inbound | Outbound |
|---|---|---|
| `sg-alb` | 443 from 0.0.0.0/0 | 8080 to `sg-gateway` |
| `sg-gateway` | 8080 from `sg-alb` | 8081–8091 to `sg-services` |
| `sg-services` | 8081–8091 from `sg-gateway` + `sg-services` | 5432 to `sg-rds`, 5671 to `sg-mq`, 8761 to `sg-eureka` |
| `sg-eureka` | 8761 from `sg-services` + `sg-gateway` | — |
| `sg-rds` | 5432 from matching `sg-services` only | — |
| `sg-mq` | 5671 from `sg-services` | — |
| `sg-elasticache` | 6379 from `sg-gateway` | — |

Each service's security group only allows inbound from specific other service groups — no `sg-services` allows blanket access to all ports.

---

## 6. Amazon MQ (RabbitMQ) Configuration

```
Deployment mode: ACTIVE_STANDBY_MULTI_AZ
Broker instance type: mq.m5.large
Engine version: RabbitMQ 3.12.x
Public accessibility: false (VPC-only)
TLS: enabled (port 5671)
Auto minor version upgrades: true
Maintenance window: Sunday 03:00–05:00 UTC

Virtual host: /oms

Exchanges (Topic type):
  oms.user, oms.attendance, oms.seating, oms.remote,
  oms.visitor, oms.event, oms.inventory, oms.audit
  oms.dlx (dead-letter exchange)

Queue naming: {exchange}.{consumer-service}
  e.g., oms.remote.attendance-service
        oms.remote.notification-service
        oms.remote.audit-service

All queues: durable=true, autoDelete=false
All queues: x-dead-letter-exchange=oms.dlx
```

---

## 7. RDS PostgreSQL Configuration

One RDS instance per service. All use the same configuration template.

```
Engine: PostgreSQL 16
Instance class: db.t3.small
Storage: 20 GB gp3 (auto-scaling to 100 GB)
Multi-AZ: true (production)
Automated backups: daily, 7-day retention
Point-in-time recovery: enabled
Encryption at rest: enabled (AWS KMS)
Performance Insights: enabled
Parameter group: oms-postgres16 (log_min_duration_statement=1000)

Database users (per service):
  {service}_app  → SELECT, INSERT, UPDATE, DELETE on service tables
  audit_app      → INSERT only on audit_logs (no UPDATE, no DELETE)
  {service}_mig  → Used by Flyway migration only; dropped after migration
```

---

## 8. CI/CD Pipeline

Each service has an independent GitHub Actions pipeline. Deploying `seating-service` never affects `attendance-service`.

```
Trigger: Pull Request opened / updated
         Push to develop, staging, main

Pipeline steps:
  1. Lint (Checkstyle — Google Java Style)
  2. Unit tests (JUnit 5 + Mockito)
  3. Integration tests (Spring Boot Test + TestContainers)
     → Real PostgreSQL container
     → Real RabbitMQ container
  4. Pact consumer contract tests
  5. OWASP Dependency Check (fail on CVSS >= 7)
  6. Docker build (multi-stage)
  7. Docker push to ECR (on merge only)
  8. Pact provider verification (on push to staging branch)

Environments:
  develop → auto-deploy to DEV
  staging → auto-deploy to STAGING + Pact provider verify
  main    → manual approval gate → PRODUCTION + rollback plan documented
```

### Dockerfile (all services — same template)

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw -B -ntp package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S oms && adduser -S oms -G oms
USER oms
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### ECS Deployment Step (GitHub Actions)

```yaml
- name: Deploy to ECS
  run: |
    aws ecs update-service \
      --cluster oms-cluster \
      --service ${{ env.SERVICE_NAME }} \
      --force-new-deployment \
      --region eu-west-1
    aws ecs wait services-stable \
      --cluster oms-cluster \
      --services ${{ env.SERVICE_NAME }}
```

---

## 9. Disaster Recovery

| Component | Strategy | RTO | RPO |
|---|---|---|---|
| RDS | Multi-AZ + daily snapshots + PITR | < 30 min (failover) | < 24 hours (daily snapshot) |
| Amazon MQ | Active/standby multi-AZ | < 2 min (AWS-managed failover) | Zero (messages in durable queues survive) |
| ECS | Multi-AZ task placement; ALB health-check-driven | < 5 min (tasks restart automatically) | Zero (stateless) |
| S3 (audit archival) | Standard S3 durability (11 nines) | Immediate | Zero |
| ECR (Docker images) | AWS-managed; multi-AZ | Immediate | Zero |

---

## 10. Environment Summary

| Environment | Purpose | Deployment | RDS | MQ |
|---|---|---|---|---|
| `dev` | Active development | Auto on `develop` merge | Single-AZ, db.t3.micro | Single broker |
| `staging` | Pre-production, Pact verify | Auto on `staging` merge | Single-AZ, db.t3.small | Single broker |
| `production` | Live system | Manual gate from `main` | Multi-AZ, db.t3.small | Active/standby |
