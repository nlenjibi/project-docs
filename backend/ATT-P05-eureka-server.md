# ATT-P05 — AWS Cloud Map (Service Discovery)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P05 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, cloud-map, service-discovery, infrastructure, aws |
| **Perspective** | Infrastructure / Backend |

---

## Story

As the platform, I want AWS Cloud Map provisioned as a private DNS namespace so that all ECS services can discover each other by name at `service-name.oms.local` — with no registry container to deploy, no client library to configure, and no self-preservation quirks.

---

## Background

AWS Cloud Map replaces Netflix Eureka. There is **no ECS container to run** for this component — Cloud Map is a fully managed AWS service provisioned via Terraform / CDK as part of M1 infrastructure.

ECS services are configured with **Service Discovery** in their task/service definitions. When an ECS task starts, ECS automatically registers it into Cloud Map with the task's private IP. When a task stops or fails its health check, ECS deregisters it automatically. There is no client-side registration code.

Services call each other using plain HTTP URLs with the Cloud Map DNS name:
- Spring Boot services: `RestTemplate` or `WebClient` with URL `http://auth-service.oms.local:8081`
- FastAPI services: `httpx.AsyncClient` with base URL `http://auth-service.oms.local:8081`

No Eureka client library. No `@FeignClient(name = "auth-service")` Eureka lookup. Just DNS.

**How the VPC Link uses it:** API Gateway's HTTP proxy integration targets `http://service-name.oms.local:PORT` via the VPC Link. Cloud Map resolves this DNS to the healthy ECS task IP, and the traffic is load-balanced across all healthy tasks via the SRV record round-robin.

---

## Acceptance Criteria

### Cloud Map Namespace
- [ ] AWS Cloud Map **private DNS namespace** created: `oms.local`, associated with the OMS VPC.
- [ ] Namespace type: **DNS private** (not HTTP namespace) — so services can resolve names using standard DNS without any Cloud Map SDK dependency.

### Service Discovery Registration (per ECS service)
- [ ] The following Cloud Map services are created within the `oms.local` namespace, one per ECS service:

  | Cloud Map Service Name | DNS Name | Port | Health Check |
  |---|---|---|---|
  | `auth-service` | `auth-service.oms.local` | 8081 | `/actuator/health/readiness` |
  | `attendance-service` | `attendance-service.oms.local` | 8083 | `/health/ready` |
  | `seating-service` | `seating-service.oms.local` | 8084 | `/actuator/health/readiness` |
  | `remote-service` | `remote-service.oms.local` | 8085 | `/actuator/health/readiness` |
  | `notification-service` | `notification-service.oms.local` | 8086 | `/health/ready` |
  | `audit-service` | `audit-service.oms.local` | 8087 | `/actuator/health/readiness` |
  | `inventory-service` | `inventory-service.oms.local` | 8090 | `/actuator/health/readiness` |
  | `workplace-service` | `workplace-service.oms.local` | 8088 | `/actuator/health/readiness` |

- [ ] Each ECS **service definition** includes a `serviceRegistries` block referencing the corresponding Cloud Map service.
- [ ] ECS task health check failure triggers automatic Cloud Map **deregistration** within 30 seconds (controlled by `healthCheckCustomConfig.failureThreshold = 1` + ECS health check interval of 30s).
- [ ] DNS TTL set to **10 seconds** — ensures load balancer rotation is quick when tasks are replaced.
- [ ] DNS record type: **A** record (maps service name to task private IP). Not SRV — simpler for plain `http://` URL resolution.

### Service-to-Service Call Pattern (Spring Boot)
- [ ] Every Spring Boot service that calls another service uses a `RestTemplate` or `WebClient` bean with the service URL injected as an environment variable:
  ```bash
  AUTH_SERVICE_URL=http://auth-service.oms.local:8081
  ```
- [ ] No Eureka client dependency (`spring-cloud-starter-netflix-eureka-client`) in any `pom.xml`.
- [ ] Resilience4j circuit breaker + retry + timeout configuration remains (wraps the `WebClient` / `RestTemplate` calls) — the resilience layer is unchanged; only the URL resolution mechanism changed.
- [ ] Example Spring Boot WebClient configuration:
  ```java
  @Bean
  public WebClient authServiceClient(@Value("${AUTH_SERVICE_URL}") String baseUrl) {
      return WebClient.builder()
          .baseUrl(baseUrl)
          .defaultHeader("Content-Type", "application/json")
          .build();
  }
  ```

### Service-to-Service Call Pattern (FastAPI)
- [ ] Every FastAPI service that calls another service uses an `httpx.AsyncClient` with the URL from an environment variable:
  ```python
  AUTH_SERVICE_URL = os.environ["AUTH_SERVICE_URL"]  # http://auth-service.oms.local:8081
  ```
- [ ] No `py-eureka-client` dependency in any `requirements.txt`.
- [ ] Example httpx client setup in FastAPI lifespan:
  ```python
  @asynccontextmanager
  async def lifespan(app: FastAPI):
      app.state.auth_client = httpx.AsyncClient(
          base_url=settings.AUTH_SERVICE_URL,
          timeout=3.0,
          headers={"Content-Type": "application/json"},
      )
      yield
      await app.state.auth_client.aclose()
  ```

### Security Group
- [ ] Security group `sg-services` allows inbound on ports 8081–8090 from `sg-services` itself (for service-to-service) and from `sg-api-gateway-vpclink` (for API Gateway routing).
- [ ] No service is directly reachable from the internet — only via the VPC Link.

### Health Check Endpoint Requirements
- [ ] Every Spring Boot service exposes `/actuator/health/readiness` returning `{ "status": "UP" }` — this is the ECS + Cloud Map health check target.
- [ ] Every FastAPI service exposes `GET /health/ready` returning `{ "status": "ok" }`. Registered as ECS health check:
  ```json
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -f http://localhost:PORT/health/ready || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
  ```

### Tests
- [ ] Integration test: deploy two ECS tasks (or use LocalStack for local validation); verify `auth-service.oms.local` resolves to a task private IP via `nslookup` inside the VPC.
- [ ] Integration test: stop one ECS task; verify DNS is updated within 40 seconds (TTL 10s + deregistration delay); verify traffic routes only to remaining healthy task.
- [ ] Smoke test: `curl -f http://auth-service.oms.local:8081/actuator/health` returns 200 from within the VPC.

---

## Terraform Resource Outline

```hcl
# Cloud Map namespace
resource "aws_service_discovery_private_dns_namespace" "oms" {
  name = "oms.local"
  vpc  = aws_vpc.oms.id
}

# One service per ECS service
resource "aws_service_discovery_service" "attendance" {
  name = "attendance-service"
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

# ECS service — service discovery registration
resource "aws_ecs_service" "attendance" {
  ...
  service_registries {
    registry_arn = aws_service_discovery_service.attendance.arn
    port         = 8083
  }
}
```

---

## Environment Variables (per service)

```bash
# Added to every ECS task definition that calls auth-service
AUTH_SERVICE_URL=http://auth-service.oms.local:8081

# attendance-service specific (calls auth-service only)
AUTH_SERVICE_URL=http://auth-service.oms.local:8081

# seating-service (calls auth-service)
AUTH_SERVICE_URL=http://auth-service.oms.local:8081

# remote-service (calls auth-service)
AUTH_SERVICE_URL=http://auth-service.oms.local:8081
```

---

## Definition of Done

- [ ] Terraform / CDK code reviewed and merged
- [ ] `oms.local` private DNS namespace created and associated with the OMS VPC
- [ ] All 8 Cloud Map services created with correct A record TTL (10s)
- [ ] All 8 ECS service definitions updated with `serviceRegistries` block
- [ ] No `spring-cloud-starter-netflix-eureka-client` in any Spring Boot `pom.xml`
- [ ] No `py-eureka-client` in any FastAPI `requirements.txt`
- [ ] All service-to-service URLs externalised to `*_SERVICE_URL` env vars
- [ ] DNS resolution smoke test passing inside VPC
- [ ] Health check endpoints confirmed working (`/actuator/health/readiness` for Spring Boot; `/health/ready` for FastAPI)
- [ ] Task deregistration from Cloud Map confirmed within 40s of task failure
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **No service dependencies** — Cloud Map namespace must be provisioned first in M1, before any ECS service is deployed.
- VPC and private subnets must exist before the namespace can be associated.
- **ATT-P04** (API Gateway + VPC Link) depends on this ticket — the Lambda Authorizer and all API Gateway route integrations resolve DNS via Cloud Map.
- All service tickets (auth, attendance, seating, remote, notification, audit) depend on this — all inter-service `*_SERVICE_URL` env vars are not resolvable until Cloud Map is live.
