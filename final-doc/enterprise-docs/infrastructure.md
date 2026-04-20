# Infrastructure & Deployment
## Office Management System (OMS)

**Version:** 4.0 | **Platform:** AWS | **Orchestration:** ECS Fargate

---

## 1. Infrastructure Overview

```
Internet / Corporate Network
        ↓ HTTPS / TLS 1.3
[Amazon CloudFront]
  Static asset CDN · /api/** forwarded to API Gateway
        ↓
[AWS API Gateway HTTP API]
  Lambda Authorizer: SESSION cookie → auth-service → internal JWT
  VPC Link: routes to ECS via Cloud Map DNS
  CORS · Rate limiting (Usage Plans) · Correlation ID injection
        ↓ VPC Link (private subnets)
┌────────────────────────────────────────────────┐
│           ECS Cluster: oms-cluster              │
│                                                 │
│  auth-service         (2–10 tasks)    :8081    │
│  attendance-service   (2–6 tasks)     :8083    │  ← FastAPI / Python 3.12
│  seating-service      (2–10 tasks)    :8084    │
│  remote-service       (2–6 tasks)     :8085    │
│  notification-service (2–6 tasks)     :8086    │  ← FastAPI / Python 3.12
│  audit-service        (2–8 tasks)     :8087    │
│  inventory-service    (2–6 tasks)     :8090    │  ← Phase 2
│  workplace-service    (2–6 tasks)     :8088    │  ← Phase 2
└──────────────┬────────────────────────────────┘
               │ Cloud Map DNS (oms.local)
   ┌───────────┼──────────────────┐
   │           │                  │
Amazon SNS   AWS RDS           AWS S3
+ SQS        PostgreSQL        audit archival
(replaces MQ) 8 isolated        >24 months
             instances         Glacier IR
```

---

## 2. AWS Services Used

| Service | Purpose | Configuration |
|---|---|---|
| **ECS Fargate** | Container orchestration | `oms-cluster`, 8 services |
| **AWS API Gateway HTTP API** | Public-facing API routing + auth | Lambda Authorizer, VPC Link, Usage Plans |
| **Amazon CloudFront** | CDN + React SPA delivery | Cache SPA assets; forward /api/** to API Gateway |
| **AWS Lambda** | Lambda Authorizer | Python 3.12; calls auth-service; 300s cache |
| **AWS Cloud Map** | Service discovery | `oms.local` private DNS; A records, 10s TTL |
| **Amazon SNS** | Async event fanout | 6 topics: oms-user, oms-attendance, oms-seating, oms-remote, oms-inventory, oms-audit |
| **Amazon SQS** | Async message queuing per consumer | One queue per consumer per topic; DLQ via Terraform RedrivePolicy (maxReceiveCount=3) |
| **Amazon SES** | Transactional email | notification-service only; eu-west-1 |
| **AWS RDS** | PostgreSQL databases | 8 instances, `db.t3.small`, Multi-AZ |
| **ECR** | Docker image registry | One repository per service |
| **Secrets Manager** | Runtime secrets injection | DB passwords, JWT key, SSO client secret |
| **S3** | Audit log archival | Glacier Instant Retrieval |
| **CloudWatch Logs** | Centralised log aggregation | `/oms/{service-name}` log group per service |
| **CloudWatch Metrics + Alarms** | Metrics + alerting | Container Insights; custom metrics; DLQ depth alarms |
| **Certificate Manager** | TLS certificates | CloudFront + API Gateway HTTPS |
| **VPC + VPC Link** | Network isolation | Private subnets for ECS; VPC Link bridges API GW → ECS |
| **AWS WAF** | Edge security | OWASP Core Rule Set on API Gateway |

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
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole-seating",
  "containerDefinitions": [
    {
      "name": "seating-service",
      "image": "ACCOUNT.dkr.ecr.eu-west-1.amazonaws.com/oms/seating-service:v1.0.0",
      "portMappings": [{ "containerPort": 8084, "protocol": "tcp" }],
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
        { "name": "AUTH_SERVICE_URL", "value": "http://auth-service.oms.local:8081" },
        { "name": "AWS_REGION", "value": "eu-west-1" },
        { "name": "SNS_TOPIC_ARN_SEATING", "value": "arn:aws:sns:eu-west-1:ACCOUNT:oms-seating" },
        { "name": "SQS_QUEUE_URL_NOTIFICATIONS", "value": "https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-notifications-queue" },
        { "name": "BOOKING_WINDOW_DAYS", "value": "14" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:eu-west-1:ACCOUNT:secret:oms/seating/db-password" },
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

> **Note:** ECS registers tasks with Cloud Map automatically via `serviceRegistries` in the ECS Service definition (not the task definition). The Terraform snippet below shows this pattern:

```hcl
resource "aws_ecs_service" "seating" {
  name            = "seating-service"
  cluster         = aws_ecs_cluster.oms.id
  task_definition = aws_ecs_task_definition.seating.arn
  desired_count   = 2

  service_registries {
    registry_arn = aws_service_discovery_service.seating.arn
  }
  ...
}
```

### CPU and Memory Per Service

| Service | CPU | Memory | Reason |
|---|---|---|---|
| `auth-service` | 256 | 512 MB | Stateless JWT validation; moderate load |
| `attendance-service` | 512 | 1024 MB | FastAPI; nightly batch job CPU/memory intensive |
| `seating-service` | 256 | 512 MB | Real-time queries; moderate |
| `remote-service` | 256 | 512 MB | Policy engine; moderate |
| `notification-service` | 256 | 512 MB | FastAPI; SQS consumer + SES dispatch |
| `audit-service` | 256 | 512 MB | High write volume; append-only |
| `inventory-service` | 256 | 512 MB | Phase 2 |
| `workplace-service` | 256 | 512 MB | Phase 2 |

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

All services run in **private subnets**. VPC Link ENIs also reside in private subnets; there are no public-facing load balancers.

```
VPC: 10.0.0.0/16

Public subnets (no ALB — VPC Link ENIs are in private subnets):
  10.0.1.0/24  eu-west-1a
  10.0.2.0/24  eu-west-1b

Private subnets (all ECS tasks + RDS + VPC Link ENIs):
  10.0.10.0/24  eu-west-1a
  10.0.11.0/24  eu-west-1b
  10.0.12.0/24  eu-west-1c
```

**Security Group Rules:**

| Security Group | Inbound | Outbound |
|---|---|---|
| `sg-apigw-vpclink` | Managed by API Gateway VPC Link | 8081–8091 to `sg-services` |
| `sg-services` | 8081–8091 from `sg-apigw-vpclink` + `sg-services` (inter-service) | 5432 to `sg-rds`; 443 to SNS/SQS/SES endpoints |
| `sg-rds` | 5432 from matching `sg-services` only | — |
| `sg-lambda` | — | 443 to `sg-services` (Lambda Authorizer → auth-service) |

Each service's security group only allows inbound from specific other security groups — no group grants blanket access across all ports.

---

## 6. Amazon SNS + SQS Configuration

```
SNS Topics (6 total):
  oms-user, oms-attendance, oms-seating, oms-remote, oms-inventory, oms-audit
  All: Standard type (not FIFO); eu-west-1

SQS Queues:
  Naming: oms-{topic}-{consumer-service}
  e.g.: oms-remote-attendance-service
         oms-remote-notification-service
         oms-remote-audit-service

All queues:
  Visibility timeout: 30s
  Message retention: 4 days
  Long polling: WaitTimeSeconds=20 (set at consumer)

DLQ per queue:
  Name: {queue-name}-dlq
  Configured via Terraform aws_sqs_queue_redrive_policy
  maxReceiveCount: 3

SNS → SQS subscription filter policy (example):
  oms-remote → oms-remote-attendance-service:
  { "eventType": ["request.approved", "ooo.request.approved"] }
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

Each service has an independent Jenkins pipeline. Deploying `seating-service` never affects `attendance-service`.

```
Trigger: Pull Request opened / updated
         Push to develop, staging, main

Jenkins pipeline (Jenkinsfile — same structure for all services):

pipeline {
  agent any
  environment {
    ECR_REGISTRY = 'ACCOUNT.dkr.ecr.eu-west-1.amazonaws.com'
    SERVICE_NAME = 'seating-service'
    AWS_REGION = 'eu-west-1'
  }
  stages {
    stage('Lint') { steps { sh './mvnw checkstyle:check' } }
    stage('Test') { steps { sh './mvnw test' } }
    stage('OWASP') { steps { sh './mvnw org.owasp:dependency-check-maven:check -Dfail.build.on.cvss=7' } }
    stage('Docker Build') {
      steps {
        sh 'docker build -t $ECR_REGISTRY/oms/$SERVICE_NAME:$BUILD_NUMBER .'
      }
    }
    stage('Push to ECR') {
      when { branch 'main' }
      steps {
        sh 'aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY'
        sh 'docker push $ECR_REGISTRY/oms/$SERVICE_NAME:$BUILD_NUMBER'
      }
    }
    stage('Deploy to ECS') {
      when { branch 'main' }
      steps {
        sh '''
          aws ecs update-service --cluster oms-cluster --service $SERVICE_NAME --force-new-deployment --region $AWS_REGION
          aws ecs wait services-stable --cluster oms-cluster --services $SERVICE_NAME
        '''
      }
    }
  }
}
```

### Dockerfile — Spring Boot Services

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

### Dockerfile — FastAPI Services (attendance-service, notification-service)

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
RUN addgroup --system oms && adduser --system --ingroup oms oms
USER oms
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY . .
EXPOSE 8083
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8083", "--workers", "2"]
```

---

## 9. Disaster Recovery

| Component | Strategy | RTO | RPO |
|---|---|---|---|
| RDS | Multi-AZ + daily snapshots + PITR | < 30 min | < 24 hours |
| SQS | AWS-managed; multi-AZ; messages retained 4 days | Immediate | Zero (at-least-once delivery) |
| ECS | Multi-AZ task placement; API GW health-check-driven | < 5 min | Zero (stateless) |
| S3 (audit archival) | Standard S3 durability | Immediate | Zero |
| ECR | AWS-managed; multi-AZ | Immediate | Zero |
| API Gateway | AWS-managed; multi-AZ; 99.99% SLA | Immediate | Zero |

---

## 10. Environment Summary

| Environment | Purpose | Deployment | RDS |
|---|---|---|---|
| `dev` | Active development | Auto on `develop` merge | Single-AZ, db.t3.micro |
| `staging` | Pre-production | Auto on `staging` merge | Single-AZ, db.t3.small |
| `production` | Live system | Manual gate from `main` | Multi-AZ, db.t3.small |
