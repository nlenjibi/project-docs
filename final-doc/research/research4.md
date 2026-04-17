# ECS + Eureka + API Gateway Research

> **Context:** OMS switching from EKS (Kubernetes) to AWS ECS for container orchestration.  
> Team: Angular frontend, Java/Spring Boot backend, AWS infrastructure.  
> Service discovery: Netflix Eureka (Spring Cloud).  
> API Gateway candidates: Spring Cloud Gateway vs AWS API Gateway vs Kong.

---

## Part 1 — AWS ECS: What You Need to Know

### 1.1 What Is ECS?

Amazon Elastic Container Service (ECS) is a **fully managed container orchestration service** from AWS. You describe your containers in a **Task Definition** and ECS handles running, restarting, and scaling them. You do not manage a Kubernetes control plane.

Two launch modes:
- **Fargate** — serverless containers. AWS manages the underlying servers. You pay per task CPU/memory. **Recommended for OMS.**
- **EC2** — you provision and manage the EC2 instances that run your containers. More control, more ops work. Only worth it at very high scale.

### 1.2 ECS vs EKS — Why ECS Is the Right Call for OMS

| Dimension | EKS (Kubernetes) | ECS (Fargate) |
|---|---|---|
| **Control plane** | You pay for and manage K8s control plane ($75/month + ops) | AWS manages it — zero ops |
| **Deployment unit** | Pod + Deployment + HPA + Service YAML | Task Definition + ECS Service |
| **Scaling** | HorizontalPodAutoscaler (YAML config) | ECS Service Auto Scaling (Target Tracking) |
| **Networking** | K8s CNI (complex) | AWS VPC networking — each task gets its own ENI |
| **Service discovery** | K8s DNS (built-in) | AWS Cloud Map or Eureka |
| **Service mesh** | Istio or AWS App Mesh | AWS App Mesh or handled at app level |
| **Secrets** | K8s Secrets or Secrets Manager | Native AWS Secrets Manager injection |
| **Logging** | Needs fluentd/fluentbit setup | Native CloudWatch Logs via `awslogs` driver |
| **Learning curve** | High — kubectl, YAML, K8s concepts | Low — ECS console, CLI, or CDK |
| **Team expertise required** | Kubernetes knowledge | AWS knowledge (most teams already have this) |
| **Cost at OMS scale** | Higher (control plane + nodes) | Lower (pay per task second, Fargate) |

**Bottom line:** EKS is powerful but complex — it makes sense for teams with dedicated platform engineers managing Kubernetes. For OMS (8 services, hundreds of employees, a product team without a dedicated K8s operator), ECS Fargate removes the operational burden and lets the team focus on building features.

---

### 1.3 Core ECS Concepts (What Replaces K8s Concepts)

| Kubernetes | ECS Equivalent | Notes |
|---|---|---|
| Pod | **Task** | One running instance of your container(s) |
| Deployment | **ECS Service** | Manages desired task count, restarts, rolling deploys |
| HPA (autoscaler) | **ECS Service Auto Scaling** | Scale based on CPU, memory, or custom CloudWatch metric |
| Namespace | **ECS Cluster** | Logical grouping of services |
| ConfigMap | **ECS Task Definition env vars + Parameter Store** | Environment config |
| Secret | **AWS Secrets Manager** | Injected as env vars at task start |
| Liveness probe | **ECS health check** | Defined in Task Definition |
| Readiness probe | **ALB target group health check** | ALB removes unhealthy tasks from routing |
| Service YAML | **Task Definition (JSON)** | Defines container image, CPU, memory, env vars, ports |

---

### 1.4 What Each OMS Service Needs in ECS

Every service gets:

```
ECS Cluster: oms-cluster
    │
    ├── ECS Service: auth-service
    │     Task Definition: oms-auth-service:latest
    │     Desired Count: 2
    │     Launch Type: FARGATE
    │     CPU: 256 (.25 vCPU)   Memory: 512 MB
    │     Secrets: DB_PASSWORD → Secrets Manager
    │     Env Vars: KAFKA_BOOTSTRAP_SERVERS, EUREKA_SERVER_URL, ...
    │     Health Check: GET /actuator/health
    │     Auto Scaling: Target CPU 70% → min 2, max 10
    │
    ├── ECS Service: attendance-service
    │     CPU: 512 (.5 vCPU)    Memory: 1024 MB  ← heavier (nightly sync job)
    │     ...
    │
    └── (one ECS Service per OMS service)
```

**Task Definition example (attendance-service):**
```json
{
  "family": "oms-attendance-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "attendance-service",
      "image": "123456789.dkr.ecr.eu-west-1.amazonaws.com/oms/attendance-service:v1.0.0",
      "portMappings": [{ "containerPort": 8080 }],
      "environment": [
        { "name": "KAFKA_BOOTSTRAP_SERVERS", "value": "broker1:9092,broker2:9092" },
        { "name": "EUREKA_SERVER_URL", "value": "http://eureka-service:8761/eureka" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:...attendance-db-password" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/oms/attendance-service",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

### 1.5 Networking in ECS

With `awsvpc` networking mode (required for Fargate):
- Each task gets its **own private IP address** inside the VPC
- Tasks communicate with each other over the VPC using private IPs or DNS names
- This is where **Eureka** becomes critical — tasks have dynamic IPs, so services need a registry to find each other

**Security groups:**
- Each ECS service gets its own security group
- `auth-service` SG: allows inbound 8080 from API Gateway SG only
- `attendance-service` SG: allows inbound 8080 from API Gateway SG + internal services
- `audit-service` SG: allows inbound Kafka only (no REST callers from outside)
- Database SGs: allow inbound 5432 from their respective service SG only — no other service can reach it

---

### 1.6 ECS Auto Scaling

Replaces Kubernetes HPA:

```
ECS Service Auto Scaling
  Target Tracking Policy:
    Metric: ECSServiceAverageCPUUtilization
    Target: 70%
    Min capacity: 2
    Max capacity: 10
    Scale-out cooldown: 60s
    Scale-in cooldown: 300s
```

For `attendance-service` you may want a **scheduled scaling action** instead — scale up at 01:45 before the nightly badge sync job, scale down after it completes.

---

### 1.7 CI/CD with ECS

Replaces the K8s rollout pipeline:

```
GitHub Actions:
  1. Lint + unit tests + integration tests (TestContainers)
  2. Pact contract tests
  3. OWASP scan
  4. Docker build
  5. Push image to AWS ECR
  6. aws ecs update-service --cluster oms-cluster
                            --service attendance-service
                            --force-new-deployment
     ↓
  ECS performs rolling deployment:
    - Starts new tasks with new image
    - Waits for health check to pass
    - Drains and stops old tasks
    - No downtime
```

---

## Part 2 — Netflix Eureka: Service Discovery for ECS

### 2.1 What Eureka Does

Netflix Eureka is a **service registry**. When a Spring Boot service starts up, it registers itself with the Eureka Server, announcing: "I am `attendance-service`, my IP is 10.0.1.45, my port is 8080."

When `seating-service` wants to call `attendance-service`, it asks Eureka: "Give me the address of `attendance-service`" — and Eureka returns the current live instances. Spring Cloud LoadBalancer then picks one (round-robin by default).

```
attendance-service starts
    → POST http://eureka:8761/eureka/apps/ATTENDANCE-SERVICE
    → { ip: "10.0.1.45", port: 8080, status: UP }

seating-service wants to call attendance-service
    → asks Eureka: GET /eureka/apps/ATTENDANCE-SERVICE
    → gets [ { ip: "10.0.1.45", port: 8080 }, { ip: "10.0.2.11", port: 8080 } ]
    → Spring Cloud LoadBalancer picks one
    → calls http://10.0.1.45:8080/api/v1/attendance/...
```

Heartbeats are sent every 30 seconds. If a service stops sending heartbeats, Eureka removes it from the registry after ~90 seconds.

### 2.2 Why Eureka Is a Good Fit with ECS

- **ECS Fargate tasks get dynamic private IPs** — every task restart may get a new IP. Eureka handles this automatically via the registration/heartbeat cycle.
- **Spring Boot has first-class Eureka support** via `spring-cloud-netflix-eureka-client` — add one dependency and two config lines and the service self-registers.
- **Client-side load balancing** — the calling service picks the instance, not a central load balancer. This means no single load balancer bottleneck for inter-service calls.
- **Spring Cloud OpenFeign + Eureka** — inter-service REST calls can use Feign clients that resolve service names via Eureka automatically:

```java
@FeignClient(name = "user-service")  // resolves via Eureka
public interface UserServiceClient {
    @GetMapping("/api/v1/users/{id}")
    UserResponse getUser(@PathVariable UUID id);
}
```

### 2.3 Running Eureka Server on ECS

Eureka Server is itself a Spring Boot app — it runs as its own ECS service:

```
ECS Service: eureka-server
  Desired Count: 2  (for HA)
  CPU: 256  Memory: 512MB
  Port: 8761
  Internal ALB (not public-facing)
  Health Check: GET /actuator/health
```

**application.yml for Eureka Server:**
```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: eureka-server
  client:
    register-with-eureka: false   # Server does not register itself
    fetch-registry: false
  server:
    enable-self-preservation: false   # Disable in dev; enable in prod
```

**application.yml for every OMS service (Eureka client):**
```yaml
spring:
  application:
    name: attendance-service   # This is the name other services use to find it

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true    # Required in ECS — use IP, not hostname
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

### 2.4 Eureka High Availability

Run **2 Eureka Server instances** that peer-replicate with each other. If one goes down, the other continues serving the registry.

```yaml
# eureka-server-1
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server-2:8761/eureka/

# eureka-server-2
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server-1:8761/eureka/
```

Both are behind an internal Application Load Balancer — all services point to the ALB DNS name, not individual instances.

---

## Part 3 — API Gateway: Spring Cloud Gateway vs AWS API Gateway vs Kong

### 3.1 What the API Gateway Must Do for OMS

Before comparing options, here is what OMS needs its gateway to do:

| Requirement | Detail |
|---|---|
| **Session validation** | Call `auth-service /api/v1/auth/validate` on every request |
| **Routing** | Route `/api/v1/attendance/**` → `attendance-service`, etc. |
| **Rate limiting** | Per-user and per-IP limits |
| **Request logging** | Correlation ID generated and propagated |
| **CORS** | Per-environment config |
| **Zero business logic** | Gateway authenticates and routes only |
| **Eureka integration** | Discover downstream services by name, not hardcoded IP |
| **Health** | Gateway must be highly available |

---

### 3.2 Option A — Spring Cloud Gateway

Spring Cloud Gateway is an **API Gateway built in Java/Spring Boot**. It runs as a regular Spring Boot application on ECS. It integrates natively with Eureka and Spring Security.

**How it works with OMS:**

```
Angular SPA → Spring Cloud Gateway (ECS Service)
                │
                ├── Spring Security: validate session cookie
                │     → calls auth-service via Eureka
                │
                ├── Route: /api/v1/attendance/** → lb://attendance-service
                │     (lb:// prefix = resolve via Eureka LoadBalancer)
                │
                ├── Rate limiting filter (Redis or in-memory)
                │
                └── Correlation ID filter: generate X-Correlation-ID header
```

**application.yml (Spring Cloud Gateway):**
```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: attendance-service
          uri: lb://attendance-service          # Eureka name
          predicates:
            - Path=/api/v1/attendance/**
          filters:
            - name: CircuitBreaker
              args:
                name: attendance-cb
                fallbackUri: forward:/fallback/attendance

        - id: seating-service
          uri: lb://seating-service
          predicates:
            - Path=/api/v1/seat**,/api/v1/seats/**

        - id: auth-service
          uri: lb://auth-service
          predicates:
            - Path=/api/v1/auth/**

      default-filters:
        - AddRequestHeader=X-Correlation-ID, #{T(java.util.UUID).randomUUID().toString()}
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 100
            redis-rate-limiter.burstCapacity: 200

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
```

**Authentication filter (calls auth-service):**
```java
@Component
public class SessionValidationFilter implements GlobalFilter {

    @Autowired
    private WebClient authClient;  // calls auth-service via Eureka

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String sessionCookie = exchange.getRequest().getCookies()
            .getFirst("session")
            .getValue();

        return authClient.get()
            .uri("http://auth-service/api/v1/auth/validate")
            .cookie("session", sessionCookie)
            .retrieve()
            .bodyToMono(UserContext.class)
            .flatMap(user -> {
                ServerHttpRequest mutated = exchange.getRequest()
                    .mutate()
                    .header("X-User-Id", user.getId().toString())
                    .header("X-User-Roles", user.getRoles())
                    .header("X-Location-Id", user.getLocationId().toString())
                    .build();
                return chain.filter(exchange.mutate().request(mutated).build());
            })
            .onErrorResume(e -> unauthorized(exchange));
    }
}
```

**Pros:**
- Java/Spring Boot — same language and framework as all OMS services. The team already knows it.
- Native Eureka integration — routes resolve service names via Eureka automatically. No hardcoded IPs or URLs.
- Resilience4j built-in — circuit breakers and retries on gateway-level routes use the same library already in every service.
- Full control — auth logic, correlation ID generation, rate limiting are all written in Java and testable.
- Runs as an ECS service — no extra infrastructure, no new tooling.
- Low cost — just another ECS Fargate task.

**Cons:**
- Requires writing and maintaining gateway code (filters, route config)
- Java process memory overhead (~256–512MB per pod)
- Rate limiting requires Redis for distributed state across gateway instances

---

### 3.3 Option B — AWS API Gateway

AWS API Gateway is a **fully managed, serverless API Gateway** from AWS. You configure routes and integrations in the AWS console or via CDK/CloudFormation. No code to deploy.

Two products:
- **REST API** — full-featured, more expensive, supports request/response transformation
- **HTTP API** — newer, simpler, ~70% cheaper, supports JWT authorizers

**How it works with OMS:**

```
Angular SPA → AWS API Gateway
                │
                ├── JWT Authorizer (or Lambda Authorizer for session cookie validation)
                ├── Route: /api/v1/attendance/** → VPC Link → attendance-service ALB
                └── Logs to CloudWatch automatically
```

**The challenge with OMS session cookies:**
AWS API Gateway's built-in JWT Authorizer works with **Bearer tokens in the Authorization header**. OMS uses **HTTP-only session cookies** validated by `auth-service`. This means you need a **Lambda Authorizer** — a small Lambda function that extracts the session cookie, calls `auth-service /validate`, and returns an IAM policy. This adds latency (~50–100ms cold start) and cost ($0.20 per million invocations).

**VPC Link requirement:**
To route to ECS services running in a private VPC, AWS API Gateway needs a **VPC Link** connected to a Network Load Balancer (NLB) in front of each ECS service. This means:
- 1 NLB per service (or 1 shared NLB with multiple listeners) — NLBs cost ~$16/month each
- For 8 services: potentially significant additional cost

**Pros:**
- Zero infrastructure to manage — fully serverless
- Native CloudWatch logging and metrics
- Built-in DDoS protection (AWS Shield Standard)
- AWS WAF can be attached

**Cons:**
- **Session cookie auth requires a Lambda Authorizer** — extra latency, extra cost, extra Lambda to maintain
- **VPC Link + NLB required** for private ECS services — adds cost and configuration
- **No Eureka integration** — AWS API Gateway routes to fixed NLB endpoints, not Eureka-registered service names. Every ECS service needs its own NLB or target group.
- Less flexible for custom filters (correlation ID injection, request transformation) without Lambda
- Per-request pricing adds up at scale: ~$3.50/million requests (REST API) or ~$1.00/million (HTTP API)

---

### 3.4 Option C — Kong

Kong is an **open-source API Gateway** that runs as a container. It is configured via a declarative YAML file or its Admin API. It has a rich plugin ecosystem.

**How it works with OMS:**

```
Angular SPA → Kong Gateway (ECS Service)
                │
                ├── Plugin: custom-auth (calls auth-service to validate session)
                ├── Plugin: rate-limiting
                ├── Plugin: correlation-id
                ├── Route: /api/v1/attendance/** → http://attendance-service:8080
                └── Plugin: prometheus (metrics)
```

**Kong with Eureka:** Kong does not integrate with Eureka natively. You would need to either:
- Use static upstream URLs (hardcoded ECS service discovery DNS via AWS Cloud Map)
- Use a custom Kong plugin to query Eureka
- Route through internal ALBs per service

**Pros:**
- Rich plugin ecosystem (rate limiting, auth, transformations, observability)
- Declarative config (kong.yml) — version controlled
- Prometheus plugin exports gateway metrics
- No per-request cost (runs on Fargate)

**Cons:**
- **No native Eureka integration** — since OMS uses Eureka, Kong cannot discover services dynamically via the registry. You need static upstream definitions or internal ALBs.
- New technology for the team — Kong configuration and plugin development is not Java/Spring Boot
- Requires running Kong's database (PostgreSQL) or using DB-less mode
- More ops work than AWS API Gateway

---

### 3.5 Comparison Table

| Dimension | Spring Cloud Gateway | AWS API Gateway | Kong |
|---|---|---|---|
| **Eureka integration** | Native (`lb://service-name`) | None (needs static NLBs) | None (needs static upstreams) |
| **Team expertise** | Java/Spring Boot — already known | AWS console/CDK — known | New tooling |
| **Session cookie auth** | Clean Java filter → auth-service | Lambda Authorizer (extra latency + cost) | Custom plugin (Lua or Go) |
| **Correlation ID** | Java filter — 5 lines | Lambda or mapping template | Plugin |
| **Circuit breaker** | Resilience4j built-in | Not supported | Not supported |
| **Rate limiting** | Redis-backed filter | Built-in | Built-in plugin |
| **Cost** | ECS Fargate task (~$5–10/month) | Per-request + Lambda + NLBs (~$50–200/month at scale) | ECS Fargate task (~$5–10/month) |
| **Ops overhead** | Low (Java app on ECS) | Near-zero | Medium (Kong config + DB) |
| **Flexibility** | Full (Java code) | Limited without Lambda | High (plugin ecosystem) |
| **Observability** | Spring Actuator + Prometheus | CloudWatch built-in | Prometheus plugin |
| **Resilience4j** | Yes | No | No |

---

### 3.6 Recommendation: Spring Cloud Gateway

**For OMS on ECS + Eureka + Java/Spring Boot team — Spring Cloud Gateway is the correct choice.**

Here is why:

**1. Eureka integration is the deciding factor.**
Spring Cloud Gateway resolves service names via Eureka using `lb://service-name`. When `attendance-service` restarts and gets a new IP, the gateway automatically gets the new address from Eureka. With AWS API Gateway or Kong, you would need static NLBs or AWS Cloud Map entries for every service — adding cost and removing the benefit of Eureka.

**2. The team writes Java.**
The gateway's auth filter, correlation ID filter, and rate limiting are all written in Java using Spring patterns the team already knows. Zero learning curve. The gateway is just another Spring Boot service in the repo.

**3. Session cookie auth is clean.**
OMS uses HTTP-only session cookies validated by `auth-service`. A Spring Security `GlobalFilter` does this in ~30 lines of Java. In AWS API Gateway, the same thing requires a Lambda Authorizer with a cold-start latency penalty. In Kong, it requires writing a Lua or Go plugin.

**4. Resilience4j circuit breakers on routes.**
Gateway routes in Spring Cloud Gateway support Resilience4j circuit breakers natively — the same library every OMS service already uses. If `attendance-service` is degraded, the gateway's circuit breaker opens and returns a configured fallback response. Neither AWS API Gateway nor Kong supports this without custom code.

**5. Cost.**
Spring Cloud Gateway runs as 2 ECS Fargate tasks (~$5–10/month total). AWS API Gateway adds per-request costs + Lambda Authorizer costs + NLB costs. At OMS scale these are manageable but avoidable.

---

## Part 4 — Complete ECS Architecture (Post-Decision)

### Updated Infrastructure

```
Internet / Corporate Network
        ↓ HTTPS / TLS 1.3
  [Application Load Balancer — public-facing]
        ↓
  [Spring Cloud Gateway — 2 ECS Tasks]
    Auth filter → auth-service (via Eureka)
    Routing     → downstream services (via Eureka lb://)
    Rate limit  → Redis (ElastiCache)
    Correlation → X-Correlation-ID header
        ↓ Internal HTTP (VPC private subnets)
  ┌─────────────────────────────────────────┐
  │         ECS Cluster: oms-cluster         │
  │                                         │
  │  eureka-server (2 tasks — HA)           │
  │  auth-service                           │
  │  attendance-service                     │
  │  seating-service                        │
  │  remote-service                         │
  │  notification-service                   │
  │  audit-service                          │
  │  inventory-service (Phase 2)            │
  │  workplace-service (Phase 2)            │
  └──────────┬──────────────────────────────┘
             │
  ┌──────────┼───────────────┐
  │          │               │
AWS RDS   AWS MSK        AWS S3
(PostgreSQL) (Kafka)    (audit archival)
8 isolated  multi-AZ    cold storage
instances   cluster
```

### What Replaces What

| Old (EKS) | New (ECS) |
|---|---|
| K8s Deployment YAML | ECS Task Definition (JSON) |
| K8s Service | ECS Service + internal ALB (only for Spring Cloud Gateway) |
| HorizontalPodAutoscaler | ECS Service Auto Scaling (Target Tracking) |
| Kubernetes DNS | Netflix Eureka (service registry) |
| Istio service mesh | mTLS handled at application layer (internal JWT over HTTPS) |
| kubectl | AWS CLI / AWS Console / CDK |
| K8s Secrets | AWS Secrets Manager (injected as env vars in Task Definition) |
| K8s ConfigMap | ECS Task Definition env vars + AWS Parameter Store |
| Helm charts | AWS CDK or CloudFormation |

### mTLS Without Istio

With ECS, Istio is not available. For Zero Trust between services, OMS uses:
- **Internal JWT** — every inter-service HTTP call carries a short-lived JWT in the `Authorization` header. Each service validates this JWT on every inbound request.
- **Security groups** — at the AWS network level, each service's security group only allows inbound traffic from trusted sources (the gateway SG, and specific service SGs).
- **HTTPS within VPC** — services communicate over HTTPS using internal certificates (AWS Certificate Manager Private CA or self-signed certs loaded via Secrets Manager).

This is the **application-layer Zero Trust** approach, which is slightly less automated than Istio (mutual cert rotation is manual) but fully sufficient for OMS's scale and is what most Java microservices teams use in practice.

---

## Summary

| Decision | Choice | Reason |
|---|---|---|
| Container orchestration | **AWS ECS Fargate** | Simpler than EKS; no K8s control plane; right size for OMS |
| Service discovery | **Netflix Eureka** | Native Spring Boot integration; handles dynamic ECS IPs |
| API Gateway | **Spring Cloud Gateway** | Native Eureka integration; Java (team expertise); clean session cookie auth; Resilience4j built-in; low cost |
| Service mesh | **Internal JWT + security groups** | No Istio on ECS; application-level Zero Trust is sufficient |
| Load balancer (public) | **AWS ALB** | Sits in front of Spring Cloud Gateway; terminates TLS |
| Secrets | **AWS Secrets Manager** | Native ECS Task Definition injection |
| Logs | **CloudWatch Logs** | Native ECS `awslogs` driver; zero setup |
