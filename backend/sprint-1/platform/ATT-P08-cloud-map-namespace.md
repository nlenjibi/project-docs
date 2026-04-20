# ATT-P08 — AWS Cloud Map Namespace Provisioning

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P08 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, cloud-map, service-discovery, terraform, infrastructure, aws |
| **Perspective** | Infrastructure |

---

## Story

As the platform, I want the `oms.local` AWS Cloud Map private DNS namespace and all service discovery records provisioned via Terraform so that ECS tasks can register and resolve each other by name — with no registry container to run, no self-registration code, and no client library dependency.

---

## Background

This ticket covers the **Terraform infrastructure provisioning** for AWS Cloud Map. Service-side adoption (removing Eureka client code from Spring Boot services, adding `httpx.AsyncClient` to FastAPI services, externalising `*_SERVICE_URL` env vars) is tracked in **ATT-P05**.

**Why this is a separate ticket from ATT-P05:**
ATT-P05 covers the code changes every developer makes in their service. This ticket is the infrastructure provisioning work done by one person (or a dedicated infrastructure PR) before any ECS service is deployed. Cloud Map namespace must exist before the first ECS service starts — it is a hard dependency for all M1 service tickets.

**How it works:**
- A single Terraform resource creates the `oms.local` private DNS namespace in the OMS VPC.
- One `aws_service_discovery_service` resource is created per ECS service — this is the registration target.
- Each ECS service definition includes a `service_registries` block pointing to the corresponding Cloud Map service.
- When an ECS task starts, ECS automatically creates an A record: `service-name.oms.local → task private IP`. No application code runs for this.
- When a task fails its health check, ECS automatically deregisters it from Cloud Map within one health check interval.

---

## Acceptance Criteria

### Cloud Map Namespace
- [ ] AWS Cloud Map **private DNS namespace** created: name `oms.local`, associated with the OMS VPC ID.
- [ ] Namespace type: `DNS_PRIVATE` — allows standard DNS resolution within the VPC without the Cloud Map SDK.
- [ ] Namespace description: `OMS internal service discovery namespace`.

### Service Discovery Services (one per ECS service)
- [ ] The following Cloud Map services are created within `oms.local`:

  | Cloud Map Service | DNS Name | Port | DNS TTL | Health Check Source |
  |---|---|---|---|---|
  | `auth-service` | `auth-service.oms.local` | 8081 | 10s | ECS task health check |
  | `attendance-service` | `attendance-service.oms.local` | 8083 | 10s | ECS task health check |
  | `seating-service` | `seating-service.oms.local` | 8084 | 10s | ECS task health check |
  | `remote-service` | `remote-service.oms.local` | 8085 | 10s | ECS task health check |
  | `notification-service` | `notification-service.oms.local` | 8086 | 10s | ECS task health check |
  | `audit-service` | `audit-service.oms.local` | 8087 | 10s | ECS task health check |
  | `workplace-service` | `workplace-service.oms.local` | 8088 | 10s | ECS task health check |
  | `inventory-service` | `inventory-service.oms.local` | 8090 | 10s | ECS task health check |

- [ ] DNS record type: **A** (not SRV) — plain `http://` URL resolution works without parsing SRV records.
- [ ] DNS TTL: **10 seconds** — ensures routing updates are reflected within one health check interval when tasks are replaced.
- [ ] Routing policy: `MULTIVALUE` — returns all healthy task IPs; clients resolve to any healthy instance.
- [ ] `healthCheckCustomConfig.failureThreshold = 1` — Cloud Map deregisters the task after 1 ECS health check failure.

### ECS Service Integration (per service Terraform)
- [ ] Every `aws_ecs_service` resource includes a `service_registries` block:
  ```hcl
  service_registries {
    registry_arn = aws_service_discovery_service.<name>.arn
    port         = <port>
  }
  ```
- [ ] The ECS cluster security group `sg-services` allows **inbound TCP on ports 8081–8090** from:
  - `sg-services` itself (service-to-service traffic)
  - `sg-api-gateway-vpclink` (API Gateway VPC Link → ECS traffic)
- [ ] No ECS service or task is directly reachable from the public internet.

### Health Check Configuration (per service)
- [ ] Every Spring Boot ECS task definition health check:
  ```json
  {
    "command": ["CMD-SHELL", "curl -f http://localhost:PORT/actuator/health/readiness || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
  ```
- [ ] Every FastAPI ECS task definition health check:
  ```json
  {
    "command": ["CMD-SHELL", "curl -f http://localhost:PORT/health/ready || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
  ```

### Tests
- [ ] `nslookup auth-service.oms.local` resolves from within the VPC (run from an ECS task or EC2 instance in the same VPC).
- [ ] Deploy two tasks of the same service; verify both IPs returned in DNS response (`MULTIVALUE` routing).
- [ ] Stop one task; verify it is deregistered from Cloud Map within 40 seconds (TTL 10s + `failureThreshold=1` × interval 30s).
- [ ] Smoke test: `curl -f http://auth-service.oms.local:8081/actuator/health` returns 200 from within the VPC.

---

## Terraform Resource Outline

```hcl
# oms.local private DNS namespace
resource "aws_service_discovery_private_dns_namespace" "oms" {
  name        = "oms.local"
  description = "OMS internal service discovery namespace"
  vpc         = var.vpc_id
}

# Reusable module for Cloud Map service + ECS service_registries
locals {
  services = {
    auth-service        = { port = 8081 }
    attendance-service  = { port = 8083 }
    seating-service     = { port = 8084 }
    remote-service      = { port = 8085 }
    notification-service = { port = 8086 }
    audit-service       = { port = 8087 }
    workplace-service   = { port = 8088 }
    inventory-service   = { port = 8090 }
  }
}

resource "aws_service_discovery_service" "services" {
  for_each = local.services

  name = each.key

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.oms.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }

  health_check_custom_config {
    failure_threshold = 1
  }
}

# Security group — service-to-service + API Gateway VPC Link
resource "aws_security_group" "services" {
  name        = "sg-oms-services"
  description = "ECS service-to-service and API Gateway VPC Link access"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 8081
    to_port         = 8090
    protocol        = "tcp"
    self            = true                          # service-to-service
  }

  ingress {
    from_port       = 8081
    to_port         = 8090
    protocol        = "tcp"
    security_groups = [aws_security_group.apigw_vpclink.id]   # API Gateway → ECS
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ECS service with Cloud Map registration (example: attendance-service)
resource "aws_ecs_service" "attendance" {
  name            = "oms-attendance-service"
  cluster         = aws_ecs_cluster.oms.id
  task_definition = aws_ecs_task_definition.attendance.arn
  desired_count   = 2

  service_registries {
    registry_arn = aws_service_discovery_service.services["attendance-service"].arn
    port         = 8083
  }

  network_configuration {
    subnets         = var.private_subnet_ids
    security_groups = [aws_security_group.services.id]
    assign_public_ip = false
  }
}
```

---

## Terraform Outputs

```hcl
output "cloud_map_namespace_id" {
  value = aws_service_discovery_private_dns_namespace.oms.id
}

output "cloud_map_namespace_arn" {
  value = aws_service_discovery_private_dns_namespace.oms.arn
}

output "service_discovery_arns" {
  value = { for k, v in aws_service_discovery_service.services : k => v.arn }
}
```

---

## Definition of Done

- [ ] Terraform code reviewed and merged
- [ ] `oms.local` private DNS namespace created and associated with the OMS VPC
- [ ] All 8 Cloud Map services created with A record TTL = 10s and `MULTIVALUE` routing
- [ ] All 8 ECS service definitions include `service_registries` block
- [ ] `sg-services` security group allows inbound 8081–8090 from self and `sg-api-gateway-vpclink`
- [ ] DNS resolution smoke test: `auth-service.oms.local` resolves from within VPC
- [ ] Task deregistration confirmed within 40s of ECS health check failure
- [ ] Health check endpoints confirmed: `/actuator/health/readiness` (Spring Boot) and `/health/ready` (FastAPI)
- [ ] Jenkins pipeline green for Terraform plan and apply
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **No service code dependencies** — this ticket provisions infrastructure only. It must be the first M1 infrastructure ticket completed.
- VPC and **private subnets** must exist before the namespace can be associated.
- **ATT-P05** (service code changes) depends on this ticket — services cannot resolve Cloud Map URLs until the namespace exists.
- **ATT-P04 / ATT-P09** (API Gateway + VPC Link) depends on this ticket — the Lambda Authorizer calls `auth-service.oms.local` and all route integrations use Cloud Map DNS.
