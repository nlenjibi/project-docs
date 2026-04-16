# Deployment & Infrastructure
## Office Management System (OMS)

---

## Infrastructure Overview

```
Internet / Corporate Network
        ↓ HTTPS / TLS 1.3
  [API Gateway — AWS API Gateway or Kong]
  Auth · Rate Limiting · Routing · Logging
        ↓ Internal mTLS (Istio / AWS App Mesh)
  ┌─────────────────────────────────────────┐
  │         AWS EKS Kubernetes Cluster       │
  │  ┌──────────────────────────────────┐   │
  │  │  Per-service Namespace           │   │
  │  │  Deployment (min 2 replicas)     │   │
  │  │  HPA (scale on CPU > 70%)        │   │
  │  │  Liveness + Readiness probes     │   │
  │  └──────────────────────────────────┘   │
  └──────────────┬──────────────────────────┘
                 │
      ┌──────────┼────────────────┐
      │          │                │
 AWS RDS      AWS MSK          AWS S3
(PostgreSQL) (Kafka)         (audit archival)
11 isolated  multi-AZ         cold storage
instances    cluster          > 24 months
```

---

## Component Decisions

| Component | Technology | Justification |
|-----------|-----------|---------------|
| API Gateway | AWS API Gateway or Kong | Centralised auth, routing, rate limiting; zero business logic |
| Message broker | Apache Kafka (AWS MSK) | Durable, replayable event log; supports audit replay |
| Databases | AWS RDS PostgreSQL (per service) | JSONB; relational integrity; consistent tooling |
| Container registry | AWS ECR | Docker image storage per service |
| Orchestration | AWS EKS (Kubernetes) | Horizontal scaling, rolling deploys, health-probe availability |
| Service mesh | Istio or AWS App Mesh | mTLS on every service-to-service call |
| Secrets | AWS Secrets Manager | Runtime secret injection; never in code or config files |
| Logging | CloudWatch Logs / ELK | Centralised stdout aggregation |
| Metrics | Prometheus + Grafana | Dashboards and alerting |
| Tracing | OpenTelemetry + Jaeger | Distributed trace correlation |
| Audit archival | AWS S3 (Glacier Instant Retrieval) | Records > 24 months |

---

## Containerisation

Each service is a single Docker container. One `Dockerfile` per service.

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Rules:**
- Base image is `eclipse-temurin:21-jre-alpine` (minimal JRE, not full JDK).
- No secrets baked into the image — all via environment variables.
- Images are tagged with the Git commit SHA and pushed to AWS ECR.

---

## Kubernetes — Per-Service Resources

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seating-service
  namespace: oms-seating
spec:
  replicas: 2
  selector:
    matchLabels:
      app: seating-service
  template:
    spec:
      containers:
        - name: seating-service
          image: ecr.aws/oms/seating-service:${GIT_SHA}
          ports:
            - containerPort: 8080
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: seating-service-secrets
                  key: db-password
            - name: INTERNAL_JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: oms-internal-secrets
                  key: jwt-secret
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: seating-service-hpa
  namespace: oms-seating
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: seating-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Resource Allocation (All Services)

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 250m | 1000m |
| Memory | 512Mi | 1Gi |
| Min replicas | 2 | — |
| Max replicas | 10 | — |
| HPA trigger | CPU > 70% | — |

---

## Service Ports

| Service | Internal Port | External (via Gateway) |
|---------|--------------|----------------------|
| identity-service | 8081 | `/api/v1/auth/**` |
| user-service | 8082 | `/api/v1/users/**`, `/api/v1/locations/**` |
| attendance-service | 8083 | `/api/v1/attendance/**` |
| seating-service | 8084 | `/api/v1/seat-bookings/**`, `/api/v1/seats/**` |
| remote-service | 8085 | `/api/v1/remote-**/**`, `/api/v1/ooo-**/**` |
| notification-service | 8086 | `/api/v1/notifications/**` |
| audit-service | 8087 | `/api/v1/audit-logs/**` |
| visitor-service | 8088 | `/api/v1/visitors/**`, `/api/v1/visits/**` |
| event-service | 8089 | `/api/v1/events/**` |
| supplies-service | 8090 | `/api/v1/supplies/**`, `/api/v1/supply-**/**` |
| assets-service | 8091 | `/api/v1/assets/**`, `/api/v1/asset-**/**` |

---

## CI/CD Pipeline (Per Service)

Each service has its own independent pipeline. Deploying `seating-service` never blocks `attendance-service`.

```
Developer opens Pull Request
        ↓
[GitHub Actions — CI]
  1. Lint (Checkstyle)
  2. Unit tests (JUnit 5)
  3. Integration tests (TestContainers)
  4. Pact contract tests (consumer side)
  5. OWASP dependency security scan
  6. Docker build
  7. Push image to ECR (on merge only)
        ↓
Merge to develop    → Auto-deploy to DEV environment
Merge to staging    → Auto-deploy to STAGING + run Pact provider verification
Merge to main       → Manual approval gate → PRODUCTION
                       (rollback plan documented before each prod deploy)
```

### Environment Promotion Rules

| Step | Trigger | Gate |
|------|---------|------|
| DEV | Merge to `develop` | Automated |
| STAGING | Merge to `staging` | Automated + Pact provider tests |
| PRODUCTION | Merge to `main` | Manual approval + rollback plan |

---

## Database Infrastructure

- **One AWS RDS PostgreSQL instance per service** — 11 instances total.
- Separate DB user per service — minimum privilege (audit service: INSERT only).
- No cross-service DB credentials — a service cannot connect to another service's database.
- RDS encryption at rest: enabled on all instances.

### RDS Configuration (per service)

| Setting | Value |
|---------|-------|
| Instance class | `db.t3.small` (scale up based on measured load) |
| Multi-AZ | Enabled (staging and production) |
| Automated backups | Daily snapshots + 7-day point-in-time recovery |
| Encryption at rest | Enabled |
| SSL in transit | Required |

---

## Kafka Infrastructure (AWS MSK)

- Multi-AZ MSK cluster; topic replication factor ≥ 3.
- Topic partitions: start with 3 per topic; increase when consumer lag is measured.
- Kafka consumer group scaling: add pods to increase throughput (pods ≤ partitions for efficiency).

```
oms.audit.event topic (3 partitions)
  → audit-service pod 1  (partition 0)
  → audit-service pod 2  (partition 1)
  → audit-service pod 3  (partition 2)
```

---

## Secrets Management

All secrets injected at runtime via AWS Secrets Manager. Never in code, config files, or Docker images.

```yaml
# Kubernetes Secret sourced from AWS Secrets Manager via External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: seating-service-secrets
spec:
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: seating-service-secrets
  data:
    - secretKey: db-password
      remoteRef:
        key: oms/seating-service/db-password
```

---

## Scaling Model

Each service scales independently. A Monday morning booking spike does not affect the nightly attendance sync.

| Service | Scaling Driver | Expected Load Pattern |
|---------|--------------|----------------------|
| `seating-service` | High concurrent reads (real-time availability) | Peaks on Monday morning |
| `attendance-service` | High write volume during nightly sync | Peaks nightly 2–4am |
| `audit-service` | High Kafka consumer throughput | Continuous throughout the day |
| `notification-service` | Event bursts | Tied to business events |
| `identity-service` | High read volume (validate on every request) | Sustained daytime load |

---

## Environments

| Environment | Purpose | Deployment |
|-------------|---------|-----------|
| `dev` | Development and integration testing | Auto on `develop` merge |
| `staging` | Pre-production validation; Pact provider verification | Auto on `staging` merge |
| `prod` | Production | Manual approval after `main` merge |

Only environment variables differ between environments — the same Docker image is promoted through all three.

---

## Disaster Recovery

| Concern | Solution | Target |
|---------|---------|--------|
| RDS failure | Multi-AZ; automated daily snapshots; 7-day point-in-time recovery | RPO < 24h, RTO < 4h |
| Kafka failure | Multi-AZ MSK; replication factor ≥ 3; at-least-once delivery on reconnect | Near-zero data loss |
| Service pod failure | Kubernetes replaces failed pods; readiness probe prevents traffic to failed pods | Automatic |
| Audit archival | Records > 24 months moved to S3 Glacier Instant Retrieval | Continuous background job |
| Full cluster failure | EKS multi-AZ node groups; infra-as-code allows cluster rebuild | RTO < 4h per service |

### RTO / RPO Targets

| Target | Value |
|--------|-------|
| RTO (Recovery Time Objective) | < 4 hours per service |
| RPO (Recovery Point Objective) | < 24 hours (daily RDS backup cadence) |

---

## Cost Estimate (Monthly — AWS)

| Component | Estimated Monthly |
|-----------|-----------------|
| EKS cluster (control plane) | ~$75 |
| Application pods (11 services × 2 pods) | ~$300–600 |
| RDS PostgreSQL (11 × db.t3.small) | ~$300–500 |
| AWS MSK (2-broker multi-AZ) | ~$200–350 |
| AWS API Gateway | ~$15–30 |
| Audit archival (S3 Glacier) | ~$5–20 |
| Observability (CloudWatch + Prometheus) | ~$50–100 |
| Secrets Manager | ~$10 |
| **Total (indicative)** | **~$955–1,685** |

_Costs depend on actual traffic, data volume, and right-sizing after load testing._

### Cost Optimisation

- Start with `db.t3.small` per service; scale only where load is demonstrated.
- Use Spot instances for non-critical pods (`notification-service`, `audit-service`).
- Start with 3 Kafka partitions per topic; increase only where consumer lag is measured.
- Audit archival to S3 Glacier prevents unbounded RDS growth.
- Single Prometheus + Grafana deployment scrapes all 11 services.
