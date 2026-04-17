# Observability
## Office Management System (OMS)

**Version:** 3.0 — OpenTelemetry + Jaeger · Prometheus + Grafana · CloudWatch Logs

Every service implements the full observability stack. **No exceptions** — a service without all four pillars cannot go to production.

**Four pillars:**
1. Structured JSON logging → stdout → CloudWatch Logs
2. Prometheus metrics → `/actuator/prometheus`
3. Distributed tracing → OpenTelemetry + Jaeger
4. Health checks → `/actuator/health/liveness` and `/actuator/health/readiness`

---

## Why This Stack

### Why OpenTelemetry + Jaeger instead of AWS X-Ray

| Criterion | OpenTelemetry + Jaeger | AWS X-Ray |
|-----------|----------------------|-----------|
| Vendor neutrality | ✅ Standard; exportable to any backend | ❌ AWS-specific SDK and format |
| Spring Boot auto-configuration | ✅ `spring-boot-starter-opentelemetry` | ❌ Requires AWS X-Ray SDK + manual setup |
| RabbitMQ trace propagation | ✅ W3C TraceContext propagation | ❌ Not supported natively |
| Feign client auto-instrumentation | ✅ Automatic via OTel agent | ❌ Manual instrumentation |
| Local development | ✅ Run Jaeger locally in Docker | ❌ Requires AWS credentials |
| Cost | Jaeger storage on ECS or OpenSearch | AWS X-Ray $5 per 1M traces |
| **Decision** | ✅ Chosen | Rejected |

### Why CloudWatch Logs instead of ELK

CloudWatch Logs is the natural choice for ECS Fargate. The `awslogs` log driver sends stdout directly to CloudWatch with zero configuration. ELK requires a Logstash/Fluentd sidecar or log shipper, adding operational complexity and cost. CloudWatch Logs Insights provides sufficient query capability for OMS scale.

---

## 1. Structured JSON Logging

All logs output to **stdout only** — no local log files. ECS `awslogs` driver forwards stdout to CloudWatch Logs. No log files to rotate or manage.

### Log Format

```json
{
  "timestamp": "2026-01-18T14:30:42.123Z",
  "level": "INFO",
  "service": "seating-service",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "correlationId": "d4e2f9a1-1234-5678-abcd-ef1234567890",
  "userId": "user-uuid",
  "locationId": "location-uuid",
  "action": "SEAT_BOOKED",
  "bookingId": "booking-uuid",
  "message": "Hot-desk booking confirmed for 2026-01-20"
}
```

**Fields always present:**
- `timestamp` — ISO 8601 UTC
- `level` — INFO / WARN / ERROR / DEBUG
- `service` — matches `spring.application.name`
- `traceId`, `spanId` — injected automatically by OpenTelemetry
- `correlationId` — from `X-Correlation-ID` header (business trace)
- `message`

**Fields present when applicable:**
- `userId`, `locationId` — resolved from SecurityContext
- `action` — business operation label (e.g. `SEAT_BOOKED`, `BADGE_SYNC_STARTED`)
- Domain-specific fields (e.g. `bookingId`, `assetId`, `eventId`)

### Logback Configuration (logback-spring.xml — every service)

```xml
<configuration>
  <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <fieldNames>
        <timestamp>timestamp</timestamp>
        <version>[ignore]</version>
      </fieldNames>
      <customFields>{"service":"${spring.application.name}"}</customFields>
      <includeMdcKeyName>correlationId</includeMdcKeyName>
      <includeMdcKeyName>userId</includeMdcKeyName>
      <includeMdcKeyName>locationId</includeMdcKeyName>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="JSON_STDOUT"/>
  </root>

  <!-- Reduce noise from framework internals -->
  <logger name="org.springframework" level="WARN"/>
  <logger name="org.hibernate" level="WARN"/>
  <logger name="com.netflix.eureka" level="WARN"/>
</configuration>
```

### Log Levels

| Scenario | Level |
|----------|-------|
| HTTP request received | DEBUG (disabled in production) |
| Business operation completed successfully | INFO |
| Fallback triggered (circuit breaker OPEN) | WARN |
| Duplicate RabbitMQ event ignored (idempotency) | INFO |
| Scheduled job started | INFO |
| Scheduled job completed with stats | INFO |
| Scheduled job failed | ERROR |
| Unresolved `personnel_id` during badge sync | WARN |
| 403 Forbidden — role/location check failed | WARN |
| Internal JWT validation failure | WARN |
| Unhandled exception | ERROR |
| DLQ message received | ERROR |

### Log Rules

- **Never log:** passwords, raw tokens, JWT values, full session cookies, personally identifiable information (names, emails) at ERROR level, credit card or financial data.
- Every log entry in a request context MUST include `correlationId` (propagated from MDC).
- Sensitive fields are logged at most as IDs (UUIDs), never as values.

```java
// ✅ Safe
log.warn("Auth failure: userId={}, correlationId={}", userId, correlationId);
log.error("Badge sync failed: locationId={}, date={}", locationId, syncDate, ex);

// ❌ Never
log.debug("Token: {}", rawJwtToken);
log.info("User data: name={}, email={}", name, email);
```

---

## 2. Prometheus Metrics

Every service exposes `/actuator/prometheus`. Prometheus scrapes all services every 15 seconds. Grafana visualises.

### Spring Boot Actuator Configuration (every service)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, info
      base-path: /actuator
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never      # Never expose DB connection details externally
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true    # Enables p50, p95, p99 histograms
      sla:
        http.server.requests: 100ms, 500ms, 1s, 3s
```

### Required Metrics (All Services)

```
# API latency — per endpoint, per method, per status code
http_server_requests_seconds{service, uri, method, status}
  → p50, p95, p99 latency per endpoint

# Error rate
http_server_requests_seconds_count{status="5xx"}
  → Divide by total to get error rate %

# RabbitMQ consumer lag — how far behind is this consumer?
spring_rabbitmq_listener_queue_message_count{queue}
  → Alert when > 500 unprocessed messages

# Circuit breaker state
resilience4j_circuitbreaker_state{name, state}
  → 0 = CLOSED (healthy), 1 = OPEN (failing), 2 = HALF_OPEN

# DB connection pool
hikaricp_connections_active{pool}
hikaricp_connections_pending{pool}
  → Alert when active approaches pool max

# JVM health
jvm_memory_used_bytes{area}
jvm_gc_pause_seconds
jvm_threads_live_threads
```

### Custom Business Metrics (Per Service)

```java
// attendance-service — nightly sync metrics
@Component
@RequiredArgsConstructor
public class BadgeSyncMetrics {
    private final MeterRegistry meterRegistry;

    public void recordSync(int inserted, int skipped, int failed, Duration duration) {
        meterRegistry.counter("badge_sync_records_inserted_total",
            "service", "attendance-service").increment(inserted);
        meterRegistry.counter("badge_sync_records_failed_total",
            "service", "attendance-service").increment(failed);
        meterRegistry.timer("badge_sync_duration_seconds",
            "service", "attendance-service").record(duration);
    }
}

// seating-service — no-show release metrics
meterRegistry.counter("no_show_releases_total",
    "service", "seating-service",
    "locationId", locationId.toString()).increment(releasedCount);

// inventory-service — stock level gauge
Gauge.builder("supply_stock_level", supplyItemRepository,
    repo -> repo.getTotalStockBelowThreshold())
    .tag("service", "inventory-service")
    .description("Number of supply items below reorder threshold")
    .register(meterRegistry);
```

### Scheduled Job Metrics (attendance-service, seating-service, auth-service)

```
badge_sync_duration_seconds{job="badge-sync", locationId}
badge_sync_records_inserted_total{locationId}
badge_sync_records_failed_total{locationId}
no_show_releases_total{locationId}
hr_sync_users_upserted_total
hr_sync_failures_total
audit_archival_records_archived_total
```

---

## 3. Distributed Tracing (OpenTelemetry + Jaeger)

Every request carries both an OpenTelemetry `traceId` (W3C TraceContext format) and an OMS `correlationId` (UUID from `X-Correlation-ID` header). These serve different purposes:

| Identifier | Format | Purpose | Propagated via |
|-----------|--------|---------|----------------|
| `traceId` | W3C hex (128-bit) | Auto-generated by OTel; spans the entire request tree | `traceparent` header |
| `correlationId` | UUID v4 | Business-level trace; includes async RabbitMQ chains | `X-Correlation-ID` header + RabbitMQ message header |

**Why both?** `traceId` is for technical debugging (Jaeger shows every span). `correlationId` correlates a business operation across async RabbitMQ events that Jaeger spans cannot capture (events are async by definition). A single `correlationId` lets you find all CloudWatch logs from a booking request, including the notification sent 2 seconds later.

### OpenTelemetry Configuration (every service)

```yaml
# application.yml
spring:
  application:
    name: seating-service     # Appears in Jaeger service list

management:
  tracing:
    sampling:
      probability: 1.0        # 100% in dev/staging; reduce to 0.1 in production
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://jaeger:4318/v1/traces}
```

```xml
<!-- pom.xml — OTel auto-instrumentation -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

With auto-instrumentation:
- Every incoming HTTP request → new span
- Every Feign outbound call → child span
- Every Spring Data JPA query → child span
- `traceId` injected into every log entry automatically via MDC

### Trace Flow Example (booking request)

```
Span: POST /api/v1/seat-bookings [seating-service] 245ms
  ├── Span: GET /internal/validate [auth-service] 12ms (Feign)
  ├── Span: INSERT seat_bookings [seating-service DB] 8ms
  ├── Span: INSERT processed_events [seating-service DB] 2ms
  └── Span: RabbitMQ publish oms.seating.booking.created 3ms
          (trace continues asynchronously via correlationId in log)
```

### RabbitMQ Trace Propagation

OpenTelemetry does not automatically trace across RabbitMQ message boundaries. The `correlationId` fills this gap.

```java
// Publisher — include correlationId in message header
Message<OmsEvent<?>> message = MessageBuilder
    .withPayload(event)
    .setHeader("X-Correlation-ID", event.correlationId().toString())
    .setHeader("traceparent", Span.current().getSpanContext().toString())
    .build();

// Consumer — restore correlationId to MDC
@Bean
public Consumer<Message<OmsEvent<NoShowDetectedPayload>>> handleNoShow() {
    return message -> {
        String correlationId = message.getHeaders().get("X-Correlation-ID", String.class);
        MDC.put("correlationId", correlationId);
        // ... process
        MDC.clear();
    };
}
```

---

## 4. Health Checks

Every service exposes two health endpoints used by ECS task health checks:

```
GET /actuator/health/liveness   → Is the JVM process alive? (not deadlocked)
GET /actuator/health/readiness  → Is the service ready to serve traffic?
                                   (DB reachable, RabbitMQ connected, Eureka registered)
```

### ECS Health Check Configuration (in Task Definition)

```json
"healthCheck": {
  "command": ["CMD-SHELL",
    "wget -qO- http://localhost:8080/actuator/health/readiness || exit 1"],
  "interval": 30,
  "timeout": 10,
  "retries": 3,
  "startPeriod": 60
}
```

`startPeriod: 60` gives the Spring Boot application 60 seconds to complete startup before ECS begins health checks. This prevents premature task replacement during deployment.

A task failing its readiness check is removed from the ALB target group until it recovers — it receives no traffic.

### Spring Boot Health Indicators

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-components: never     # Never expose health detail externally
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
    db:
      enabled: true              # RDS connectivity check
    rabbit:
      enabled: true              # Amazon MQ connectivity check
```

---

## Grafana Dashboards

| Dashboard | Key Metrics Displayed |
|-----------|----------------------|
| **System Health** | Error rate (5xx/total) per service; p95 API latency per service; circuit breaker states |
| **RabbitMQ Health** | Queue message count per queue; DLQ depth (alert if > 0); consumer processing rate |
| **Scheduled Jobs** | Badge sync: records inserted/failed, duration, last run time; no-show release completion |
| **Database Health** | HikariCP active connections vs max; slow query count per service |
| **Deployment** | ECS task count per service (expected vs actual); recent deployments timeline |
| **Business Activity** | Bookings per hour; remote requests per day; inventory request volume |

---

## Alerting Rules

### PagerDuty (Critical — immediate response required)

| Alert | Threshold | Condition |
|-------|-----------|-----------|
| Badge sync job failure | Any failure | `badge_sync_duration_seconds_count{status="FAILED"} > 0` |
| Service 5xx error rate | > 1% over 5 minutes | `rate(http_server_requests_seconds_count{status=~"5.."}) / rate(http_server_requests_seconds_count) > 0.01` |
| Circuit breaker OPEN | Any service | `resilience4j_circuitbreaker_state{state="open"} == 1` |
| auth-service 401 rate | > 5% over 5 minutes | High auth failure rate (possible attack or identity-service failure) |
| ECS task count below minimum | < 2 for any service | `aws_ecs_service_running_task_count < 2` |
| DLQ message depth | > 0 for any queue | RabbitMQ DLQ depth alert |
| RDS CPU | > 90% for 5 minutes | `aws_rds_cpuutilization > 90` |

### Slack #oms-warnings (Warning — investigate within the day)

| Alert | Threshold | Condition |
|-------|-----------|-----------|
| RabbitMQ queue depth | > 500 messages | Consumer falling behind |
| DB connection pool usage | > 80% of pool max | `hikaricp_connections_active / hikaricp_connections_max > 0.8` |
| ECS task restart count | > 3 within 1 hour | Service instability |
| No-show release job failure | Any failure | `no_show_release_failures_total > 0` |
| HR sync job failure | Any failure | `hr_sync_failures_total > 0` |
| Badge sync partial failure | Any partial | `badge_sync_records_failed_total > 0` |
| API p99 latency | > 2000ms over 10 minutes | Performance degradation |
| Audit archival job failure | Any failure | Records not moving to S3 Glacier |

---

## Log Aggregation — CloudWatch Logs

All ECS tasks use the `awslogs` log driver (zero-config for ECS Fargate).

| Environment | CloudWatch Log Group | Retention |
|-------------|---------------------|-----------|
| Dev | `/oms/{service}/dev` | 7 days |
| Staging | `/oms/{service}/staging` | 30 days |
| Production | `/oms/{service}/prod` | 1 year → S3 after 90 days |

### Useful CloudWatch Logs Insights Queries

**Trace all logs for a correlation ID (entire business operation):**
```sql
fields @timestamp, service, level, action, message
| filter correlationId = "d4e2f9a1-1234-5678-abcd-ef1234567890"
| sort @timestamp asc
```

**Find all 5xx errors in the last hour:**
```sql
fields @timestamp, service, message
| filter level = "ERROR" and message like /5[0-9][0-9]/
| sort @timestamp desc
| limit 50
```

**Badge sync failures across all locations today:**
```sql
fields @timestamp, locationId, message
| filter action = "BADGE_SYNC_FAILED"
| stats count(*) as failureCount by locationId
| sort failureCount desc
```

**Circuit breaker events:**
```sql
fields @timestamp, service, message
| filter message like /circuit breaker/ or message like /fallback/
| sort @timestamp desc
| limit 100
```

**DLQ message processing:**
```sql
fields @timestamp, service, message
| filter action = "DLQ_MESSAGE_RECEIVED"
| sort @timestamp desc
```

---

## Observability Checklist (Per Service — Before Production)

- [ ] `logstash-logback-encoder` in pom.xml; `logback-spring.xml` configured
- [ ] `correlationId` added to MDC on every request (via `CorrelationIdFilter`)
- [ ] `traceId` and `spanId` appearing in log output (OTel auto-instrumentation)
- [ ] No passwords, tokens, or PII in any log line
- [ ] `/actuator/prometheus` reachable and scraped by Prometheus
- [ ] All required base metrics emitting (circuit breaker, DB pool, RabbitMQ queue depth)
- [ ] Custom business metrics registered (job counters, domain-specific gauges)
- [ ] `X-Correlation-ID` propagated on all outbound Feign calls
- [ ] `correlationId` included in all RabbitMQ event envelopes
- [ ] `/actuator/health/liveness` and `/actuator/health/readiness` returning correct states
- [ ] ECS task definition health check configured with `startPeriod: 60`
- [ ] CloudWatch log group created with correct retention policy
- [ ] Alert rules added to Grafana for this service
- [ ] Tracing sampling probability set correctly for environment (1.0 dev/staging, 0.1 prod)
