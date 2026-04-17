# Deployment & Infrastructure
## Office Management System (OMS)

**Version:** 3.0 — AWS ECS Fargate · Spring Cloud Gateway · Netflix Eureka · Amazon MQ (RabbitMQ)

---

## Infrastructure Overview

```
Internet / Corporate Network
        ↓ HTTPS / TLS 1.3
  [Application Load Balancer]
  AWS ACM certificate · HTTPS-only (HTTP redirects to HTTPS)
        ↓
  [Spring Cloud Gateway — ECS Fargate Task]
  Session auth filter · Correlation ID injection
  Eureka-based lb:// routing · Rate limiting
        ↓
  [Netflix Eureka Server — ECS Fargate Task]
  Service registry for all 8 OMS services
        ↓
  ┌──────────────────────────────────────────────────────────┐
  │                  AWS ECS Fargate Cluster                  │
  │  ┌─────────────────────────────────────────────────────┐ │
  │  │  Per-service ECS Service (min 2 tasks, max 10)      │ │
  │  │  CPU: 512 vCPU | Memory: 1 GB (per task)            │ │
  │  │  Health check: /actuator/health/readiness           │ │
  │  │  Service Auto Scaling on CPU > 70%                  │ │
  │  └─────────────────────────────────────────────────────┘ │
  └──────────────┬───────────────────────────────────────────┘
                 │
      ┌──────────┼────────────────────┐
      │          │                    │
  AWS RDS      Amazon MQ           AWS S3
(PostgreSQL)  (RabbitMQ)       (audit archival)
8 isolated    mq.m5.large         > 24 months
instances     multi-AZ            Glacier IR
```

---

## Technology Decision Rationale

### ECS Fargate over EKS (Kubernetes)

**Decision:** AWS ECS Fargate
**Rejected:** AWS EKS

| Criterion | ECS Fargate | EKS |
|-----------|-------------|-----|
| Cluster management | Zero — AWS managed entirely | Self-managed control plane + node groups |
| Deployment config complexity | Task definition JSON (~50 lines) | Deployment YAML + RBAC + namespaces + Helm |
| Service mesh requirement | None needed at this scale | Istio adds ~3 months ramp-up |
| Eureka integration | Direct task registration via ENV vars | Works but adds Kubernetes ConfigMap complexity |
| Control plane cost | $0 | ~$75/month |
| Auto-scaling | ECS Service Auto Scaling (CPU/memory) | HPA with more configuration |
| Team ramp-up | Days | Weeks to months for production-grade K8s |
| **Annual saving** | **~$900/year** vs EKS control plane alone | — |

**Trade-off accepted:** ECS Fargate does not have Kubernetes' ecosystem richness (CRDs, advanced scheduling, etc.). If the OMS team grows significantly and needs advanced workload scheduling, migration to EKS is feasible — the Docker images and ECS task definitions map directly to Kubernetes Deployments.

### Amazon MQ (RabbitMQ) over AWS MSK (Kafka)

| Criterion | Amazon MQ RabbitMQ | AWS MSK Kafka |
|-----------|-------------------|---------------|
| Spring Cloud Stream | ✅ First-class binder | ✅ First-class binder |
| Managed cost | ~$30/month (mq.m5.large, multi-AZ) | ~$200–350/month (2-broker cluster) |
| Event replay | Not needed for OMS | ✅ (overkill) |
| Fan-out | ✅ Topic Exchange | ✅ Consumer groups |
| Operational complexity | Low | High (partition tuning, consumer lag management) |
| **Annual saving** | **~$2,040–3,840/year** | — |

### Spring Cloud Gateway over AWS API Gateway or Kong

| Criterion | Spring Cloud Gateway | AWS API Gateway | Kong |
|-----------|---------------------|-----------------|------|
| Eureka lb:// routing | ✅ Native | ❌ Requires Lambda Authorizer | ❌ Requires custom plugin |
| Session cookie auth filter | ✅ Java filter in same codebase | ❌ Lambda Authorizer + extra latency | ❌ Custom plugin required |
| Team tooling | ✅ Same Spring ecosystem | ❌ New AWS service to operate | ❌ New platform to operate |
| Cost | ECS task (~$15/month) | Pay-per-request (~$3.50/M requests) | Kong cluster (~$50–100/month) |

---

## Containerisation

Each service is a single Docker image. Multi-stage build minimises image size and excludes build tools from the runtime image.

```dockerfile
# Multi-stage Dockerfile — identical for all 8 services
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests -q

# Stage 2: Runtime — JRE only, no JDK
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Non-root user for security
RUN addgroup -S oms && adduser -S oms -G oms
USER oms

COPY --from=build /build/target/*.jar app.jar

# Health check for ECS container-level health
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health/liveness || exit 1

EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

**Rules:**
- Base image: `eclipse-temurin:21-jre-alpine` — JRE only, Alpine Linux (~170MB vs ~700MB JDK-based)
- No secrets in the image — all via ECS Task Definition environment variables sourced from Secrets Manager
- Images tagged with Git commit SHA: `ecr.aws/oms/seating-service:${GIT_SHA}`
- Multi-stage build: build artifacts not included in runtime image

---

## ECS Fargate Task Definition

One task definition per service. Example for `seating-service`:

```json
{
  "family": "oms-seating-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/oms-ecs-execution-role",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/oms-seating-task-role",
  "containerDefinitions": [
    {
      "name": "seating-service",
      "image": "ACCOUNT_ID.dkr.ecr.eu-west-1.amazonaws.com/oms/seating-service:GIT_SHA",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
        { "name": "EUREKA_CLIENT_SERVICEURL_DEFAULTZONE",
          "value": "http://eureka.oms.internal:8761/eureka/" },
        { "name": "RABBITMQ_HOST", "value": "amqp.oms.internal" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT_ID:secret:oms/seating-service/db-password"
        },
        {
          "name": "INTERNAL_JWT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT_ID:secret:oms/internal-jwt-secret"
        },
        {
          "name": "RABBITMQ_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT_ID:secret:oms/rabbitmq-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/oms/seating-service",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL",
          "wget -qO- http://localhost:8080/actuator/health/readiness || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

---

## ECS Service Configuration (Per Service)

| Setting | Value |
|---------|-------|
| Launch type | FARGATE |
| Min tasks | 2 (always 2 for high availability) |
| Max tasks | 10 |
| Auto Scaling policy | `ECSServiceAverageCPUUtilization > 70%` → scale out |
| Deployment type | Rolling update — `minimumHealthyPercent: 100`, `maximumPercent: 200` |
| Health check grace period | 60 seconds (Spring Boot startup time) |
| VPC | Private subnets only (no public IP) |

---

## Service Resource Allocation

| Service | CPU (vCPU) | Memory (MB) | Min Tasks | Scaling Driver |
|---------|-----------|------------|-----------|----------------|
| `auth-service` | 512 | 1024 | 2 | High sustained daytime load (every request validates here) |
| `attendance-service` | 512 | 1024 | 2 | Nightly spike (2–4am badge sync batch writes) |
| `seating-service` | 512 | 1024 | 2 | Monday morning concurrent read spikes |
| `remote-service` | 256 | 512 | 2 | Moderate daytime load |
| `notification-service` | 256 | 512 | 2 | Event-driven bursts |
| `audit-service` | 256 | 512 | 2 | Continuous steady throughput |
| `inventory-service` | 256 | 512 | 2 | Low; Phase 2 |
| `workplace-service` | 256 | 512 | 2 | Low; Phase 2 |
| `eureka-server` | 256 | 512 | 2 | Registry — steady, critical |
| `spring-cloud-gateway` | 512 | 1024 | 2 | Every inbound request passes here |

**Why min 2 tasks for all services:** A single task failure would make the service fully unavailable. Two tasks allow rolling deployments (one stops, one serves) and absorb task restarts without downtime.

---

## Netflix Eureka Server

Eureka is deployed as a pair of ECS Fargate tasks (2 replicas for HA). All OMS services register on startup.

```yaml
# eureka-server application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: eureka.oms.internal
  client:
    registerWithEureka: false    # Server does not register with itself
    fetchRegistry: false
  server:
    enableSelfPreservation: true   # Do not remove services during brief network interruptions
    expectedClientRenewalIntervalSeconds: 30
    renewalPercentThreshold: 0.85
```

```yaml
# All 8 OMS services — Eureka client config
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka.oms.internal:8761/eureka/
    registerWithEureka: true
    fetchRegistry: true
    registryFetchIntervalSeconds: 30
  instance:
    preferIpAddress: true          # ECS Fargate uses dynamic IPs — must register by IP
    leaseRenewalIntervalInSeconds: 30
    leaseExpirationDurationInSeconds: 90
    healthCheckUrlPath: /actuator/health
    statusPageUrlPath: /actuator/info
```

**Why `preferIpAddress: true`:** ECS Fargate tasks get a dynamic private IP from the VPC subnet on each launch. They do not have stable hostnames. Registering by IP ensures the gateway can route to live task IPs. Eureka automatically removes stale IPs when a task stops sending heartbeats.

---

## Spring Cloud Gateway Configuration

```yaml
# gateway application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: auth-route
          uri: lb://auth-service
          predicates: [Path=/api/v1/auth/**, /api/v1/users/**, /api/v1/locations/**]
        - id: attendance-route
          uri: lb://attendance-service
          predicates: [Path=/api/v1/attendance/**]
        - id: seating-route
          uri: lb://seating-service
          predicates: [Path=/api/v1/seat-bookings/**, /api/v1/seats/**]
        - id: remote-route
          uri: lb://remote-service
          predicates: [Path=/api/v1/remote-requests/**, /api/v1/ooo-requests/**,
                        /api/v1/teams/**, /api/v1/approval-delegates/**,
                        /api/v1/remote-day-policies/**]
        - id: notification-route
          uri: lb://notification-service
          predicates: [Path=/api/v1/notifications/**]
        - id: audit-route
          uri: lb://audit-service
          predicates: [Path=/api/v1/audit-logs/**]
        - id: inventory-route
          uri: lb://inventory-service
          predicates: [Path=/api/v1/supplies/**, /api/v1/supply-requests/**,
                        /api/v1/assets/**, /api/v1/asset-assignments/**,
                        /api/v1/asset-requests/**, /api/v1/fault-reports/**,
                        /api/v1/maintenance-records/**]
        - id: workplace-route
          uri: lb://workplace-service
          predicates: [Path=/api/v1/visitors/**, /api/v1/visits/**,
                        /api/v1/agreement-templates/**, /api/v1/events/**]
      default-filters:
        - name: SessionAuthFilter      # Validates session cookie via auth-service /internal/validate
        - name: CorrelationIdFilter    # Generates X-Correlation-ID on every inbound request
        - name: RequestRateLimiter     # Redis-backed rate limiter (100 req/s per user)
```

---

## Database Infrastructure

- **One AWS RDS PostgreSQL instance per service** — 8 instances.
- Separate DB user per service — minimum privilege (audit-service: INSERT only on `audit_logs`).
- No cross-service DB credentials — physically enforced.
- All instances: encryption at rest enabled.

### RDS Configuration

| Setting | Value |
|---------|-------|
| Engine | PostgreSQL 16 |
| Instance class | `db.t3.small` (scale up based on load test results) |
| Multi-AZ | Enabled in staging and production |
| Automated backups | Daily snapshots + 7-day point-in-time recovery |
| Encryption at rest | AWS KMS — enabled on all instances |
| SSL in transit | Required (`sslmode=require` in all JDBC URLs) |
| Deletion protection | Enabled in production |

### Instance Sizing Rationale

`db.t3.small` (2 vCPU, 2 GB RAM) is the starting point for all services. Scale up only where load testing demonstrates sustained CPU > 70% or slow query counts rising. Typical Spring Boot + PostgreSQL services for OMS volumes will not exhaust `db.t3.small` in Phase 1.

---

## Amazon MQ (RabbitMQ) Configuration

| Setting | Value |
|---------|-------|
| Instance type | `mq.m5.large` |
| Deployment mode | `ACTIVE_STANDBY_MULTI_AZ` |
| Protocol | AMQP over TLS (port 5671) |
| Virtual host | `/oms` |
| Authentication | Service-specific credentials from Secrets Manager |

**Why ACTIVE_STANDBY_MULTI_AZ:** Automatic failover to the standby broker if the primary fails. Failover takes ~30 seconds. RabbitMQ clients reconnect automatically. Messages queued in durable queues are preserved.

**Exchange setup (run once at deployment, idempotent):**
```
Exchange: oms.events       (type: topic, durable: true, auto-delete: false)
Exchange: oms.dlx          (type: direct, durable: true, auto-delete: false)
All queues: durable: true, auto-delete: false
DLQ binding: every queue → oms.dlx on x-dead-letter-exchange
```

---

## Secrets Management

All secrets injected at runtime via AWS Secrets Manager. No secrets in code, config files, Docker images, or environment variable defaults.

**Secrets store paths:**
```
oms/{service-name}/db-password         → RDS database password per service
oms/internal-jwt-secret                → Shared HMAC-SHA256 key for internal JWT
oms/rabbitmq-password                  → Amazon MQ service user password
oms/sso/jwks-uri                       → SSO provider JWKS endpoint
oms/sso/issuer-uri                     → SSO provider issuer URI
```

**ECS retrieval:** Task Definition `secrets` block references Secrets Manager ARN directly. ECS retrieves the value at task startup and injects it as an environment variable. The task role has `secretsmanager:GetSecretValue` permission only on the specific secrets it needs.

---

## CI/CD Pipeline (Per Service)

Each service has its own independent GitHub Actions pipeline. Deploying `seating-service` never blocks `inventory-service`.

```yaml
# .github/workflows/ci.yml (per service)
on:
  push:
    branches: [develop, staging, main]
  pull_request:
    branches: [develop]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }

      - name: Lint (Checkstyle)
        run: ./mvnw checkstyle:check

      - name: Unit Tests
        run: ./mvnw test -Dtest="**/*Test"

      - name: Integration Tests (TestContainers)
        run: ./mvnw test -Dtest="**/*IT"

      - name: Pact Consumer Contract Tests
        run: ./mvnw test -Dtest="**/*Pact*"

      - name: OWASP Dependency Security Scan
        run: ./mvnw org.owasp:dependency-check-maven:check

      - name: Build Docker Image
        run: docker build -t $ECR_REGISTRY/oms/$SERVICE_NAME:$GITHUB_SHA .

      - name: Push to ECR (on merge only)
        if: github.event_name == 'push'
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker push $ECR_REGISTRY/oms/$SERVICE_NAME:$GITHUB_SHA

  deploy-dev:
    needs: ci
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to DEV (ECS update-service)
        run: |
          aws ecs update-service \
            --cluster oms-cluster-dev \
            --service oms-$SERVICE_NAME \
            --force-new-deployment

  deploy-staging:
    needs: ci
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to STAGING
        run: aws ecs update-service --cluster oms-cluster-staging --service oms-$SERVICE_NAME --force-new-deployment

      - name: Run Pact Provider Verification
        run: ./mvnw test -Dtest="**/*PactProvider*" -Dpact.provider.version=$GITHUB_SHA

  deploy-prod:
    needs: ci
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://oms.yourorg.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to PRODUCTION (manual approval gate in GitHub environment)
        run: aws ecs update-service --cluster oms-cluster-prod --service oms-$SERVICE_NAME --force-new-deployment
```

### Environment Promotion Rules

| Environment | Trigger | Gate | Auto-deploy? |
|-------------|---------|------|-------------|
| `dev` | Merge to `develop` | All CI checks pass | Yes |
| `staging` | Merge to `staging` | CI + Pact provider verification | Yes |
| `production` | Merge to `main` | Manual approval in GitHub | No — human approval required |

**Same Docker image is promoted through all environments.** Only environment variables differ (injected at task startup). This guarantees that what was tested in staging is exactly what runs in production.

---

## Service Ports

| Service | Internal Container Port | External (via Gateway) |
|---------|------------------------|-----------------------|
| `spring-cloud-gateway` | 8080 | 443 (ALB → 8080) |
| `eureka-server` | 8761 | Internal only (VPC) |
| `auth-service` | 8080 | `/api/v1/auth/**`, `/api/v1/users/**`, `/api/v1/locations/**` |
| `attendance-service` | 8080 | `/api/v1/attendance/**` |
| `seating-service` | 8080 | `/api/v1/seat-bookings/**`, `/api/v1/seats/**` |
| `remote-service` | 8080 | `/api/v1/remote-requests/**`, etc. |
| `notification-service` | 8080 | `/api/v1/notifications/**` |
| `audit-service` | 8080 | `/api/v1/audit-logs/**` |
| `inventory-service` | 8080 | `/api/v1/supplies/**`, `/api/v1/assets/**`, etc. |
| `workplace-service` | 8080 | `/api/v1/visitors/**`, `/api/v1/events/**`, etc. |

All internal service-to-service communication uses Eureka-registered IPs on port 8080 via Feign `lb://` routing. No hardcoded service URLs.

---

## Environments

| Environment | Cluster | RDS | Amazon MQ | Purpose |
|-------------|---------|-----|-----------|---------|
| `dev` | `oms-cluster-dev` | `db.t3.micro` (single-AZ) | Shared dev instance | Development and integration testing |
| `staging` | `oms-cluster-staging` | `db.t3.small` (multi-AZ) | Dedicated staging | Pre-production; Pact provider verification |
| `production` | `oms-cluster-prod` | `db.t3.small` (multi-AZ) | `mq.m5.large` multi-AZ | Production |

Dev uses `db.t3.micro` (single-AZ, no Multi-AZ) to reduce cost. Staging mirrors production configuration to catch environment-specific issues before they reach production.

---

## Disaster Recovery

| Concern | Solution | Target |
|---------|---------|--------|
| ECS task failure | ECS replaces failed tasks automatically; ALB health check removes unhealthy targets | Automatic, <60s |
| RDS instance failure | Multi-AZ failover; standby promoted automatically | RPO ~0, RTO ~2min |
| Amazon MQ broker failure | ACTIVE_STANDBY; clients reconnect automatically | ~30s failover |
| Secrets Manager unavailability | Secrets cached by ECS for task duration; new tasks may fail to start | Negligible risk |
| Full AZ failure | Multi-AZ RDS + MQ; ECS tasks spread across AZs via placement constraints | Degraded capacity; no outage |
| Audit archival | `AuditArchivalJob` archives records > 24 months to S3 Glacier Instant Retrieval | Continuous |
| Complete region failure | Not in scope for Phase 1. Addressed in Phase 3 via Route 53 + multi-region RDS. | — |

### RTO / RPO Targets

| Target | Value |
|--------|-------|
| RTO (Recovery Time Objective) | < 4 hours per service |
| RPO (Recovery Point Objective) | < 1 hour (RDS automated backups + point-in-time recovery) |

---

## Monthly Cost Estimate (AWS — Production)

| Component | Configuration | Est. Monthly |
|-----------|--------------|-------------|
| ECS Fargate (10 services × 2 tasks × 0.5 vCPU, 1 GB) | ~0.5 vCPU, 1GB | ~$150–250 |
| ECS Fargate (gateway + eureka × 2 tasks each) | 0.5 vCPU, 1GB | ~$30–50 |
| AWS RDS PostgreSQL (8 × db.t3.small, multi-AZ) | 8 instances | ~$280–400 |
| Amazon MQ (mq.m5.large, multi-AZ) | Active+standby | ~$60–80 |
| Application Load Balancer | 1 ALB | ~$20–30 |
| AWS ECR (image storage) | 8 services × ~500MB | ~$5–10 |
| AWS Secrets Manager (12 secrets) | At-rest + API calls | ~$10–15 |
| CloudWatch Logs (all services) | Log ingestion + storage | ~$30–60 |
| S3 (audit archival — Glacier IR) | Grows ~1GB/month | ~$5–10 |
| Data transfer | Intra-AZ (minimal) | ~$10–20 |
| **Total (indicative)** | | **~$600–925/month** |

**vs. original 11-service EKS + MSK Kafka design:**

| Line item | Original | New | Annual saving |
|-----------|---------|-----|--------------|
| Kafka MSK (2-broker) | ~$275/month | Replaced by Amazon MQ (~$70/month) | ~$2,460/year |
| EKS control plane | ~$75/month | $0 (ECS Fargate) | ~$900/year |
| 3 extra RDS instances (11 vs 8) | ~$105/month | Saved by service merges | ~$1,260/year |
| **Total annual saving** | | | **~$4,620/year** |

---

## Cost Optimisation Strategies

1. **Start with `db.t3.small` for all RDS instances.** Scale to `db.t3.medium` only where load testing shows consistent CPU > 70% or query latency > 200ms.
2. **ECS Fargate Spot for non-critical services.** `notification-service` and `audit-service` can run on Fargate Spot (up to 70% cheaper) because they are RabbitMQ consumers — task interruption just pauses message processing until the replacement task starts.
3. **CloudWatch Logs retention policy.** Set 30-day retention for dev, 90-day for staging, 1-year for production. Older logs go to S3 at a fraction of CloudWatch cost.
4. **Right-size after 30 days of production load.** ECS Container Insights provides per-task CPU and memory utilisation. Down-size any service consistently below 30% utilisation.
5. **Reserved Instances for RDS.** After 3 months of stable production, purchase 1-year RDS Reserved Instances for `db.t3.small` — saves ~40% over on-demand.
