# RabbitMQ + Spring Cloud Stack — Recommendation for OMS

> **Context:** OMS switching from Apache Kafka (AWS MSK) to RabbitMQ as the message broker.  
> Full stack: ECS Fargate + Eureka + Spring Cloud Gateway + RabbitMQ + Spring Cloud Stream.  
> 8 services (post-merge): auth, attendance, seating, remote, notification, audit, inventory, workplace.

---

## 1. What Changes from Kafka to RabbitMQ

This is the most important section. Understand these differences before writing a single line of code.

| Concept | Kafka | RabbitMQ |
|---|---|---|
| **Core model** | Distributed log (messages stored durably, consumers read at their own pace) | Message broker (messages delivered to queues, consumed and removed) |
| **Message lifetime** | Configurable retention (hours → forever) | Until consumed or TTL expires |
| **Fan-out** | Multiple consumer groups read the same topic independently | Multiple queues bound to one exchange — each service gets its own queue |
| **Replay** | Full replay from any offset at any time | Not possible — consumed messages are gone |
| **Ordering** | Per-partition ordering | Per-queue FIFO ordering |
| **Consumer model** | Pull — consumer controls pace via offset | Push — broker delivers to consumer |
| **Routing** | By topic name | By exchange type + routing key |
| **Protocol** | Kafka protocol | AMQP 0-9-1 |
| **Spring abstraction** | Spring Cloud Stream (Kafka binder) | Spring Cloud Stream (RabbitMQ binder) |
| **AWS managed service** | AWS MSK | Amazon MQ for RabbitMQ |

### The one major trade-off you must accept

**Audit event replay is no longer possible.**

In the Kafka design, if `audit-service` went down for 2 hours, it could rewind its offset and reprocess every missed event from the log. With RabbitMQ, messages sit in a durable queue while `audit-service` is down, and are delivered when it recovers — but there is no "replay from the beginning." This means:

- `audit-service` must be highly available (2 tasks, durable queues, DLQ configured)
- If the audit queue fills up and messages are lost, those audit records are gone
- Use **durable queues + persistent messages** for audit to survive RabbitMQ restarts

Everything else (fan-out, notification dispatch, cross-service workflows) works cleanly with RabbitMQ.

---

## 2. RabbitMQ Core Concepts You Need

### Exchanges, Queues, Bindings

```
Producer (remote-service)
    │
    │  publishes message with routing key: "remote.request.approved"
    ▼
Exchange: oms.events  (type: topic)
    │
    ├── Binding: routing key "remote.request.approved"
    │       ▼
    │   Queue: attendance.remote.request.approved  → attendance-service consumes
    │
    ├── Binding: routing key "remote.request.approved"
    │       ▼
    │   Queue: notification.remote.request.approved → notification-service consumes
    │
    └── Binding: routing key "remote.request.approved"
            ▼
        Queue: audit.remote.request.approved → audit-service consumes
```

This is how fan-out works in RabbitMQ. One message published to the exchange is **copied to all bound queues**. Each service has its own queue and consumes independently — the same logical outcome as Kafka consumer groups.

### Exchange Types

| Type | Behaviour | Use in OMS |
|---|---|---|
| **Topic** | Routes by routing key pattern (`*` = one word, `#` = zero or more words) | Recommended — one exchange per domain |
| **Fanout** | Delivers to ALL bound queues regardless of routing key | Simple but less flexible |
| **Direct** | Routes only to queues with an exact matching binding key | Point-to-point events |
| **Headers** | Routes based on message headers | Not needed for OMS |

### Recommended Design: One Topic Exchange Per Domain

```
Exchange: oms.user       (type: topic)
Exchange: oms.attendance (type: topic)
Exchange: oms.seating    (type: topic)
Exchange: oms.remote     (type: topic)
Exchange: oms.visitor    (type: topic)
Exchange: oms.event      (type: topic)
Exchange: oms.inventory  (type: topic)
Exchange: oms.audit      (type: topic)
```

Each service publishes to its domain exchange with a routing key. Consumers bind queues to the relevant exchange with the routing keys they care about.

**Example — `remote-service` publishes, three services consume:**

```
Exchange: oms.remote  (topic)

remote-service publishes:
  routing key: "request.approved"
  payload: { requestId, userId, dates, locationId }

Bound queues:
  Queue: attendance.oms.remote.request.approved
    ← attendance-service (overlays REMOTE status)

  Queue: notification.oms.remote.request.approved
    ← notification-service (sends email + in-app)

  Queue: audit.oms.remote.request.approved
    ← audit-service (persists immutably)
```

### Dead Letter Queues (DLQ) — Required for Every Queue

If a consumer throws an exception and nacks a message, it goes to the DLQ instead of being requeued indefinitely.

```
Queue: attendance.oms.remote.request.approved
  → on failure: dead-letter-exchange: oms.dlx
                dead-letter-routing-key: attendance.remote.request.approved.failed

Queue: oms.dlx.attendance.remote.request.approved.failed
  ← consumed by: ops team / alerting / manual reprocessing
```

---

## 3. Full Spring Cloud Stack for OMS

Here is the complete Spring Cloud component map:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Cloud Stack (OMS)                      │
│                                                                   │
│  ┌─────────────────────┐   ┌──────────────────────────────────┐  │
│  │  Spring Cloud        │   │  Spring Cloud Netflix Eureka     │  │
│  │  Gateway             │   │  (Service Registry)              │  │
│  │  (API Gateway)       │   │  All services self-register      │  │
│  └──────────┬───────────┘   └──────────────────────────────────┘  │
│             │                                                      │
│             │ lb://service-name (Eureka resolution)               │
│             ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │               OMS Services (8 services)                   │     │
│  │                                                           │     │
│  │  Spring Cloud OpenFeign  ← inter-service REST calls       │     │
│  │  Spring Cloud LoadBalancer ← client-side load balancing   │     │
│  │  Resilience4j ← circuit breaker, retry, timeout          │     │
│  │  Spring Cloud Stream ← RabbitMQ producer/consumer         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                          │                                         │
│                          ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │           RabbitMQ (Amazon MQ)                            │     │
│  │  Topic Exchanges + Durable Queues + DLQs                  │     │
│  └──────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Spring Cloud Libraries Per Service (pom.xml)

```xml
<!-- Service Discovery — every service -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- Load-balanced REST calls via Feign — every service that calls others -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<!-- Messaging — every service that uses RabbitMQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>

<!-- Circuit breakers + retry — every service -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>

<!-- API Gateway only -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

---

## 4. Spring Cloud Stream with RabbitMQ

Spring Cloud Stream is the abstraction layer that lets your services publish and consume messages without writing raw RabbitMQ code. You write functions; Spring wires them to exchanges and queues.

### Publishing a Message (Producer)

```java
// In remote-service
@Service
@RequiredArgsConstructor
public class RemoteEventPublisher {

    private final StreamBridge streamBridge;

    public void publishRequestApproved(RemoteRequest request) {
        RemoteApprovedEvent event = RemoteApprovedEvent.builder()
            .eventId(UUID.randomUUID())
            .requestId(request.getId())
            .userId(request.getUserId())
            .dates(request.getDates())
            .locationId(request.getLocationId())
            .occurredAt(Instant.now())
            .build();

        // "remote-request-approved-out-0" maps to exchange oms.remote, key "request.approved"
        streamBridge.send("remote-request-approved-out-0", event);
    }
}
```

### Consuming a Message (Consumer)

```java
// In attendance-service — consumes remote.request.approved
@Configuration
public class AttendanceConsumers {

    @Autowired
    private AttendanceOverlayService overlayService;

    @Bean
    public Consumer<RemoteApprovedEvent> remoteRequestApproved() {
        return event -> {
            // Idempotency check first
            if (processedEventRepo.existsById(event.getEventId())) {
                return;  // duplicate — skip silently
            }
            overlayService.overlayRemoteStatus(event.getUserId(), event.getDates(), event.getLocationId());
            processedEventRepo.save(new ProcessedEvent(event.getEventId()));
        };
    }
}
```

### application.yml (Spring Cloud Stream + RabbitMQ)

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST}           # Amazon MQ broker endpoint
    port: 5671                        # AMQPS (TLS)
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}
    ssl:
      enabled: true

  cloud:
    stream:
      # --- PRODUCERS ---
      bindings:
        remote-request-approved-out-0:
          destination: oms.remote        # exchange name
          content-type: application/json

      # --- CONSUMERS ---
      bindings:
        remoteRequestApproved-in-0:
          destination: oms.remote                              # exchange name
          group: attendance-service                            # queue name = oms.remote.attendance-service
          content-type: application/json

      # --- RabbitMQ-specific config ---
      rabbit:
        bindings:
          remote-request-approved-out-0:
            producer:
              exchange-type: topic
              routing-key-expression: "'request.approved'"

          remoteRequestApproved-in-0:
            consumer:
              exchange-type: topic
              binding-routing-key: request.approved
              durableSubscription: true         # queue survives broker restart
              auto-bind-dlq: true               # auto-create DLQ
              dlq-ttl: 604800000                # DLQ messages live 7 days
              requeue-rejected: false           # failed messages go to DLQ, not requeued
              max-attempts: 3
              backoff-initial-interval: 1000
              backoff-max-interval: 10000
              backoff-multiplier: 2.0
```

### Naming Convention for Queues

Spring Cloud Stream + RabbitMQ auto-creates queues named: `{destination}.{group}`

For OMS:
```
Exchange oms.remote, group attendance-service  → Queue: oms.remote.attendance-service
Exchange oms.remote, group notification-service → Queue: oms.remote.notification-service
Exchange oms.remote, group audit-service        → Queue: oms.remote.audit-service
```

Clean, predictable, one queue per service per exchange.

---

## 5. RabbitMQ Exchange and Queue Map for All OMS Events

### Complete Binding Table

| Exchange | Routing Key | Producer | Consumer Queues |
|---|---|---|---|
| `oms.user` | `user.created` | auth-service | `oms.user.notification-service`, `oms.user.audit-service` |
| `oms.user` | `user.deactivated` | auth-service | `oms.user.attendance-service`, `oms.user.seating-service`, `oms.user.audit-service` |
| `oms.attendance` | `record.resolved` | attendance-service | `oms.attendance.audit-service` |
| `oms.attendance` | `no_show.detected` | attendance-service | `oms.attendance.seating-service` |
| `oms.seating` | `booking.created` | seating-service | `oms.seating.notification-service`, `oms.seating.audit-service` |
| `oms.seating` | `booking.released` | seating-service | `oms.seating.notification-service`, `oms.seating.audit-service` |
| `oms.remote` | `request.approved` | remote-service | `oms.remote.attendance-service`, `oms.remote.notification-service`, `oms.remote.audit-service` |
| `oms.remote` | `request.rejected` | remote-service | `oms.remote.notification-service`, `oms.remote.audit-service` |
| `oms.visitor` | `checked_in` | workplace-service | `oms.visitor.notification-service`, `oms.visitor.audit-service` |
| `oms.event` | `rsvp.promoted` | workplace-service | `oms.event.notification-service`, `oms.event.audit-service` |
| `oms.event` | `event.cancelled` | workplace-service | `oms.event.notification-service`, `oms.event.audit-service` |
| `oms.inventory` | `request.fulfilled` | inventory-service | `oms.inventory.notification-service`, `oms.inventory.audit-service` |
| `oms.inventory` | `asset.assigned` | inventory-service | `oms.inventory.notification-service`, `oms.inventory.audit-service` |
| `oms.audit` | `#` (all keys) | all services | `oms.audit.audit-service` |

---

## 6. Inter-Service REST Calls with Spring Cloud OpenFeign + Eureka

For synchronous calls (e.g. `seating-service` calling `auth-service` to resolve a user), use OpenFeign. Services are resolved by name from Eureka — no hardcoded URLs.

```java
// In seating-service
@FeignClient(name = "auth-service")   // Eureka name
public interface AuthServiceClient {

    @GetMapping("/api/v1/users/{id}")
    UserResponse getUser(@PathVariable UUID id);
}
```

Combine with Resilience4j:

```java
@FeignClient(name = "auth-service", fallback = AuthServiceFallback.class)
public interface AuthServiceClient {
    @GetMapping("/api/v1/users/{id}")
    @CircuitBreaker(name = "auth-service")
    UserResponse getUser(@PathVariable UUID id);
}

@Component
public class AuthServiceFallback implements AuthServiceClient {
    @Override
    public UserResponse getUser(UUID id) {
        return UserResponse.defaultResponse(id);  // degraded response
    }
}
```

application.yml config for Feign + Resilience4j:
```yaml
feign:
  circuitbreaker:
    enabled: true

resilience4j:
  circuitbreaker:
    instances:
      auth-service:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
  retry:
    instances:
      auth-service:
        max-attempts: 3
        wait-duration: 500ms
  timelimiter:
    instances:
      auth-service:
        timeout-duration: 3s
```

---

## 7. Amazon MQ for RabbitMQ — The Managed Option

Do not self-host RabbitMQ on ECS. Use **Amazon MQ for RabbitMQ** — the AWS-managed service.

### Why Amazon MQ

| Concern | Self-hosted on ECS | Amazon MQ for RabbitMQ |
|---|---|---|
| Broker management | You manage RabbitMQ container, upgrades, config | AWS manages it |
| High availability | You configure clustering manually | Multi-AZ active/standby built-in |
| Storage | EBS volume management | Managed EBS, automatic |
| CloudWatch integration | Manual setup | Built-in metrics |
| Security | VPC, SGs, TLS setup manually | VPC, TLS, IAM integrated natively |
| Backups | Manual | Automated daily snapshots |

### Cost (Amazon MQ for RabbitMQ)

| Deployment | Instance | Estimated Monthly |
|---|---|---|
| Single instance (dev) | `mq.m5.large` | ~$115/month |
| Active/standby HA (prod) | `mq.m5.large` × 2 | ~$230/month |
| Storage | $0.10/GB-month | ~$5–10/month |

**Amazon MQ is significantly cheaper than AWS MSK (~$450–500/month baseline for Kafka).**  
This is a meaningful cost reduction for OMS.

### Connection in Spring Boot

Amazon MQ provides an AMQP endpoint. Use AMQPS (TLS port 5671):

```yaml
spring:
  rabbitmq:
    host: b-xxxxxxxx-xxxx.mq.eu-west-1.amazonaws.com
    port: 5671
    username: ${RABBITMQ_USERNAME}    # from AWS Secrets Manager
    password: ${RABBITMQ_PASSWORD}    # from AWS Secrets Manager
    ssl:
      enabled: true
```

---

## 8. Idempotency (Still Required)

RabbitMQ delivers **at-least-once** — duplicate messages can arrive (e.g. after a consumer crash before ack). Every consumer must still be idempotent:

```java
@Bean
public Consumer<RemoteApprovedEvent> remoteRequestApproved() {
    return event -> {
        if (processedEventRepo.existsById(event.getEventId())) {
            log.info("Duplicate event ignored: {}", event.getEventId());
            return;
        }
        // process...
        processedEventRepo.save(new ProcessedEvent(event.getEventId()));
    };
}
```

This is the same `processed_events` deduplication table pattern from the Kafka design. Nothing changes here.

---

## 9. What Drops, What Stays

### What drops compared to the Kafka architecture

| Kafka Feature | Replaced By | Notes |
|---|---|---|
| Consumer group offset management | Queue acknowledgment (AMQP ack/nack) | Simpler mental model |
| Topic partitions | RabbitMQ queue consumers | RabbitMQ handles concurrency differently |
| Retention / replay | Durable queues (until consumed) | Replay from beginning is not possible |
| AWS MSK | Amazon MQ for RabbitMQ | Cheaper, simpler |
| Kafka Schema Registry | None needed | Use versioned event DTOs instead |
| Kafka consumer lag metric | RabbitMQ queue depth metric (CloudWatch) | Different metric, same purpose |

### What stays exactly the same

- **Event envelope structure** — same JSON shape with `eventId`, `correlationId`, `occurredAt`, `locationId`, `payload`
- **Idempotency** — `processed_events` deduplication table in every consumer
- **Audit-service** — still Kafka-consumer-only (now RabbitMQ-consumer-only), INSERT-only DB user, 24-month retention, S3 archival
- **Notification-service** — still event-driven only, never called directly via REST
- **Fan-out** — still works (one exchange → many queues)
- **Choreography Saga** — supply request approval workflow still works via RabbitMQ events
- **Correlation ID propagation** — unchanged, still in message headers and HTTP headers

---

## 10. Complete Updated Architecture

```
Internet
    ↓ HTTPS
[AWS ALB — public]
    ↓
[Spring Cloud Gateway — 2 ECS Tasks]
  ├── Eureka client (discovers all services)
  ├── Session validation filter → auth-service (via Feign + Eureka)
  ├── Routing: lb://service-name
  ├── Rate limiting (Redis / ElastiCache)
  └── Correlation ID filter
    ↓
[ECS Cluster: oms-cluster]
  ├── eureka-server          (2 tasks — HA, internal ALB)
  ├── auth-service           (2 tasks min)
  ├── attendance-service     (2 tasks min — scales for nightly sync)
  ├── seating-service        (2 tasks min — scales for booking traffic)
  ├── remote-service         (2 tasks min)
  ├── notification-service   (2 tasks min)
  ├── audit-service          (2 tasks min)
  ├── inventory-service      (2 tasks min — Phase 2)
  └── workplace-service      (2 tasks min — Phase 2)
    │                │                │
[Amazon MQ       [AWS RDS         [AWS S3]
 RabbitMQ]        PostgreSQL       Audit
 Topic Exchanges  8 isolated       archival
 Durable Queues   instances
 DLQs per queue
 Active/standby HA
```

---

## 11. Summary — Final Technology Decisions

| Component | Technology | Why |
|---|---|---|
| Container orchestration | AWS ECS Fargate | Simpler than EKS; no K8s control plane |
| Service discovery | Netflix Eureka (Spring Cloud) | Native Spring Boot integration; handles dynamic ECS IPs |
| API Gateway | Spring Cloud Gateway | Eureka-native routing; Java team expertise; session cookie auth in code |
| Message broker | RabbitMQ (Amazon MQ) | Cheaper than Kafka; sufficient for OMS fan-out needs; Spring Cloud Stream support |
| Messaging abstraction | Spring Cloud Stream (RabbitMQ binder) | Same code works if broker changes; functional consumer model |
| Inter-service REST | Spring Cloud OpenFeign + Eureka | Declarative HTTP clients; Eureka name resolution; Resilience4j fallbacks |
| Client-side load balancing | Spring Cloud LoadBalancer | Built into Feign + Spring Cloud Gateway |
| Circuit breaker / retry | Resilience4j | Spring Boot 3 native; works with Feign and Gateway |
| Databases | AWS RDS PostgreSQL (8 instances) | Unchanged |
| Secrets | AWS Secrets Manager | Injected into ECS Task Definitions |
| Logs | CloudWatch Logs (awslogs driver) | Native ECS integration |
| Tracing | OpenTelemetry + Jaeger | Confirmed — no AWS X-Ray |
| Metrics | Prometheus + Grafana | Spring Actuator /actuator/prometheus on every service |
