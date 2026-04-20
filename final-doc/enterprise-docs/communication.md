# Service Communication
## Office Management System (OMS)

**Version:** 4.0 | Sync: WebClient (Spring Boot) / httpx.AsyncClient (FastAPI) | Async: Amazon SNS + SQS

---

## 1. Communication Principles

| Rule | Enforcement |
|---|---|
| Use REST for real-time user-facing queries | AWS API Gateway routes all external traffic |
| Use SNS + SQS for state changes and cross-service workflows | Audit and notification services consume events only |
| Never call notification or audit services directly via REST | No REST client to these services exists in any codebase |
| Every consumer is idempotent | `processed_events` deduplication table in every consumer |
| Consumers delete SQS messages only after successful processing | Explicit `delete_message` call after handler completes |
| All events carry a correlationId | Propagated via SQS message body and HTTP headers |

---

## 2. Synchronous Communication (WebClient / httpx)

Used for: user-facing queries requiring an immediate response.

### Spring Boot — WebClient Pattern

Service URLs are resolved via AWS Cloud Map DNS. Each service receives its upstream URLs as environment variables (e.g., `AUTH_SERVICE_URL=http://auth-service.oms.local:8081`). There are no Eureka registrations or `lb://` URIs.

```java
// seating-service calling auth-service (Spring Boot)
@Service
public class AuthServiceClient {
    private final WebClient webClient;

    public AuthServiceClient(@Value("${AUTH_SERVICE_URL}") String baseUrl) {
        this.webClient = WebClient.builder().baseUrl(baseUrl).build();
    }

    public Mono<UserResponse> getUser(UUID id) {
        return webClient.get()
            .uri("/api/v1/users/{id}", id)
            .retrieve()
            .bodyToMono(UserResponse.class);
    }

    public Mono<LocationConfigResponse> getLocationConfig(UUID id) {
        return webClient.get()
            .uri("/api/v1/locations/{id}/config", id)
            .retrieve()
            .bodyToMono(LocationConfigResponse.class);
    }
}
```

### FastAPI — httpx.AsyncClient Pattern

```python
# attendance-service → auth-service (FastAPI)
async def get_user(user_id: UUID, client: httpx.AsyncClient) -> UserResponse:
    resp = await client.get(f"/api/v1/users/{user_id}")
    resp.raise_for_status()
    return UserResponse(**resp.json()["data"])

# base_url injected from settings: AUTH_SERVICE_URL=http://auth-service.oms.local:8081
```

The `httpx.AsyncClient` is initialised once at application startup with `base_url` set from `settings.AUTH_SERVICE_URL` and shared across requests via dependency injection.

### Resilience4j Configuration (Spring Boot services)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      auth-service:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 2s
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      auth-service:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - org.springframework.web.reactive.function.client.WebClientRequestException
          - java.net.ConnectException
  timelimiter:
    instances:
      auth-service:
        timeout-duration: 3s
        cancel-running-future: true
```

### Inter-Service REST Call Matrix

```
                auth  attend  seating  remote  notif  audit
auth-service      -     -        -        -      -      -
attendance      REST    -        -     [SNS/SQS] -      -
seating         REST [SNS/SQS]   -        -      -      -
remote          REST    -        -        -      -      -
notification      -     -        -        -      -      -
audit             -     -        -        -      -      -

REST        = synchronous WebClient / httpx call
[SNS/SQS]  = consumes SNS event via SQS queue from that service
```

---

## 3. Asynchronous Communication (Amazon SNS + SQS)

OMS uses one **SNS topic** per domain. Each consumer service has its own **SQS queue** subscribed to the relevant topic with an SNS filter policy on the `eventType` message attribute. Dead-letter queues (DLQs) are provisioned as separate SQS queues via Terraform `RedrivePolicy` (maxReceiveCount=3).

### Architecture Overview

```
remote-service
    │
    │  SNS Publish (MessageAttribute: eventType=request.approved)
    ▼
oms-remote (SNS Topic)
    ├── oms-remote-attendance-service   (SQS) ← filter: request.approved
    ├── oms-remote-notification-service (SQS) ← filter: request.approved, request.rejected
    └── oms-remote-audit-service        (SQS) ← no filter (receives all)

Dead-letter queues (Terraform):
    oms-remote-attendance-service-dlq
    oms-remote-notification-service-dlq
    oms-remote-audit-service-dlq
```

### SNS Topic Registry

| SNS Topic | Event Types (`eventType` MessageAttribute) | Producer | Consumer Queues |
|---|---|---|---|
| `oms-user` | `user.created`, `user.updated`, `user.deactivated` | auth-service | `oms-user-notification-service`, `oms-user-audit-service` |
| `oms-attendance` | `record.resolved`, `restamp.requested` | attendance-service | `oms-attendance-seating-service`, `oms-attendance-audit-service`, `oms-attendance-restamp` |
| `oms-seating` | `booking.created`, `booking.cancelled`, `booking.released` | seating-service | `oms-seating-notification-service`, `oms-seating-audit-service`, `oms-seating-attendance-service` |
| `oms-remote` | `request.submitted`, `request.approved`, `request.rejected`, `ooo.request.approved` | remote-service | `oms-remote-attendance-service`, `oms-remote-notification-service`, `oms-remote-audit-service` |
| `oms-inventory` | `request.fulfilled`, `asset.assigned`, `stock.low` | inventory-service | `oms-inventory-notification-service`, `oms-inventory-audit-service` |
| `oms-audit` | all event types | all services | `oms-audit-events` |

### SNS Filter Policy

Subscriptions that should not receive every event type use an SNS filter policy. Example — `oms-remote-attendance-service` receives only approved requests:

```json
{ "eventType": ["request.approved"] }
```

Filter policies are defined in Terraform alongside the SQS subscription resource.

---

### SNS Publish Pattern (Spring Boot — AWS SDK v2)

```java
@Service
public class RemoteEventPublisher {
    private final SnsClient snsClient;
    private final ObjectMapper objectMapper;

    @Value("${SNS_TOPIC_ARN_REMOTE}") private String topicArn;

    public RemoteEventPublisher(SnsClient snsClient, ObjectMapper objectMapper) {
        this.snsClient = snsClient;
        this.objectMapper = objectMapper;
    }

    public void publishRequestApproved(RemoteRequest request) {
        var event = Map.of(
            "eventId",       UUID.randomUUID().toString(),
            "eventType",     "request.approved",
            "version",       "1",
            "correlationId", MDC.get("correlationId"),
            "occurredAt",    Instant.now().toString(),
            "locationId",    request.getLocationId().toString(),
            "payload",       Map.of(
                "requestId", request.getId().toString(),
                "userId",    request.getUserId().toString(),
                "dates",     request.getDates(),
                "approvedBy", request.getApprovedBy().toString()
            )
        );

        snsClient.publish(PublishRequest.builder()
            .topicArn(topicArn)
            .message(objectMapper.writeValueAsString(event))
            .messageAttributes(Map.of(
                "eventType", MessageAttributeValue.builder()
                    .dataType("String")
                    .stringValue("request.approved")
                    .build()
            ))
            .build());

        log.info("Published request.approved for requestId={}", request.getId());
    }
}
```

---

### SQS Consumer Pattern (Spring Boot — Spring Cloud AWS)

```java
@SqsListener("${SQS_QUEUE_URL_REMOTE}")
public void onRemoteRequestApproved(String rawMessage) throws Exception {
    var event = objectMapper.readValue(rawMessage, EventEnvelope.class);

    // Idempotency check
    if (processedEventRepo.existsById(event.getEventId())) {
        log.info("Duplicate event skipped: eventId={}", event.getEventId());
        return;
    }

    MDC.put("correlationId", event.getCorrelationId());
    try {
        overlayService.overlayRemoteStatus(event.getPayload());
        processedEventRepo.save(new ProcessedEvent(event.getEventId(), Instant.now()));
        log.info("Processed request.approved: eventId={}", event.getEventId());
    } finally {
        MDC.clear();
    }
    // Message is deleted automatically by Spring Cloud AWS after the listener returns.
    // On exception, message is not deleted; it returns to the queue and routes to DLQ
    // after maxReceiveCount=3 (configured in Terraform RedrivePolicy).
}
```

---

### SQS Consumer Pattern (FastAPI — aioboto3)

```python
async def poll_sqs(queue_url: str, handler, sqs_client):
    while True:
        response = await sqs_client.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,
            AttributeNames=["All"],
        )
        for msg in response.get("Messages", []):
            try:
                body = json.loads(msg["Body"])
                # SNS wraps the original message in a "Message" field
                payload = json.loads(body.get("Message", msg["Body"]))
                await handler(payload)
                await sqs_client.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=msg["ReceiptHandle"],
                )
            except Exception as e:
                logger.error("Failed to process SQS message", error=str(e))
                # Message is not deleted; it returns to the queue.
                # Routes to DLQ after maxReceiveCount=3 (Terraform RedrivePolicy).
```

---

### Dead-Letter Queue Configuration

DLQs are provisioned entirely in Terraform. There is no `auto-bind-dlq` or framework-managed DLQ creation.

```hcl
resource "aws_sqs_queue" "remote_attendance_dlq" {
  name                      = "oms-remote-attendance-service-dlq"
  message_retention_seconds = 604800  # 7 days
}

resource "aws_sqs_queue" "remote_attendance" {
  name = "oms-remote-attendance-service"
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.remote_attendance_dlq.arn
    maxReceiveCount     = 3
  })
}
```

---

### Universal Event Envelope

All events share this structure regardless of topic or service language.

**Java:**

```java
@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class EventEnvelope {
    private String eventId;        // UUID v4 — idempotency key
    private String eventType;      // e.g., "request.approved"
    private String version;        // "1" — increment on breaking schema change
    private String correlationId;  // propagated from X-Correlation-ID header
    private String occurredAt;     // ISO 8601 timestamp
    private String locationId;     // location context
    private Map<String, Object> payload; // event-specific data
}
```

**Python:**

```python
@dataclass
class EventEnvelope:
    event_id: str        # UUID v4 — idempotency key
    event_type: str      # e.g., "request.approved"
    version: str         # "1"
    correlation_id: str  # from X-Correlation-ID header
    occurred_at: str     # ISO 8601
    location_id: str
    payload: dict
```

**JSON example:**

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "request.approved",
  "version": "1",
  "correlationId": "660e8400-e29b-41d4-a716-446655440001",
  "occurredAt": "2026-04-17T09:15:00Z",
  "locationId": "770e8400-e29b-41d4-a716-446655440002",
  "payload": {
    "requestId": "880e8400-e29b-41d4-a716-446655440003",
    "userId": "990e8400-e29b-41d4-a716-446655440004",
    "dates": ["2026-04-18", "2026-04-19"],
    "approvedBy": "aa0e8400-e29b-41d4-a716-446655440005"
  }
}
```

---

## 4. AWS API Gateway + Lambda Authorizer — Auth and Routing

Spring Cloud Gateway and its `SessionValidationFilter` have been replaced by AWS API Gateway with a Lambda Authorizer for authentication and AWS Cloud Map for service discovery.

### Lambda Authorizer

- **Runtime:** Python 3.12
- **Trigger:** Every inbound API Gateway request (except explicitly open paths)
- **Cache TTL:** 300 seconds — reduces Lambda invocations by approximately 90% for active sessions
- **Function behaviour:**
  1. Extracts the HTTP-only `SESSION` cookie from the `Cookie` header.
  2. Calls `auth-service /api/v1/auth/validate` with the session token.
  3. On success, returns an IAM `Allow` policy and injects context into the request.
  4. On failure or missing cookie, returns an IAM `Deny` policy (HTTP 401 to the client).

**Context injected into every authorised request:**

| Key | Description |
|---|---|
| `userId` | Authenticated user's UUID |
| `userRoles` | Comma-separated role list |
| `locationIds` | Comma-separated location UUIDs the user may access |
| `correlationId` | Generated by the Lambda Authorizer for this request (UUID v4) |
| `internalJwt` | Short-lived JWT forwarded to downstream services as `Authorization` header |

Downstream services read these values from the `$context.authorizer.*` API Gateway context, which API Gateway maps to HTTP headers before forwarding to the target service.

### API Gateway Route Table

API Gateway forwards authenticated requests to services via a VPC Link. Service addresses are resolved through AWS Cloud Map DNS (`http://<service-name>.oms.local:<PORT>`).

| API Gateway Path Pattern | Target Service | Cloud Map DNS | Port |
|---|---|---|---|
| `/api/v1/auth/**`, `/api/v1/users/**`, `/api/v1/locations/**` | auth-service | `auth-service.oms.local` | 8081 |
| `/api/v1/attendance/**` | attendance-service | `attendance-service.oms.local` | 8082 |
| `/api/v1/seat-bookings/**`, `/api/v1/seats/**`, `/api/v1/locations/**/floor-plan` | seating-service | `seating-service.oms.local` | 8083 |
| `/api/v1/remote-requests/**`, `/api/v1/ooo-requests/**`, `/api/v1/teams/**/schedule`, `/api/v1/approval-delegates/**`, `/api/v1/remote-day-policies/**` | remote-service | `remote-service.oms.local` | 8084 |
| `/api/v1/notifications/**` | notification-service | `notification-service.oms.local` | 8085 |
| `/api/v1/audit-logs/**` | audit-service | `audit-service.oms.local` | 8086 |
| `/api/v1/supplies/**`, `/api/v1/supply-requests/**`, `/api/v1/assets/**`, `/api/v1/asset-requests/**`, `/api/v1/fault-reports/**`, `/api/v1/maintenance-records/**` | inventory-service | `inventory-service.oms.local` | 8087 |
| `/api/v1/visitors/**`, `/api/v1/visits/**`, `/api/v1/events/**` | workplace-service | `workplace-service.oms.local` | 8088 |

Open paths (Lambda Authorizer skipped): `/api/v1/auth/login`, `/api/v1/auth/callback`, `/actuator/health`.

---

## 5. Correlation ID Propagation

Every request receives a `correlationId` generated by the Lambda Authorizer. It is propagated to all downstream services and included in the body of every SNS/SQS message.

```
Browser request
    → API Gateway Lambda Authorizer: generates correlationId: abc-123
    → Forwards to seating-service: X-Correlation-ID: abc-123
    → seating-service publishes SNS event: correlationId: abc-123 (in message body)
    → notification-service SQS consumer: MDC.put("correlationId", "abc-123")
    → audit-service SQS consumer: MDC.put("correlationId", "abc-123")

All log lines from all services: { "correlationId": "abc-123", ... }
Trace in observability tooling: search by correlationId → full request path visible
```

### Spring Boot — MDC Filter

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String correlationId = req.getHeader("X-Correlation-ID");
        if (correlationId != null) {
            MDC.put("correlationId", correlationId);
        }
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("correlationId");
        }
    }
}
```

### FastAPI — Middleware

```python
@app.middleware("http")
async def correlation_id_middleware(request: Request, call_next):
    correlation_id = request.headers.get("X-Correlation-ID")
    if correlation_id:
        structlog.contextvars.bind_contextvars(correlation_id=correlation_id)
    try:
        return await call_next(request)
    finally:
        structlog.contextvars.clear_contextvars()
```

### Idempotency

Every SQS consumer checks the `eventId` against a `processed_events` table before handling an event. If a record already exists, the event is acknowledged (message deleted) and processing is skipped. This guards against duplicate deliveries caused by SQS at-least-once semantics and Lambda Authorizer retries.

```sql
CREATE TABLE processed_events (
    event_id   UUID        PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL
);
```
