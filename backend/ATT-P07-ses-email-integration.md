# ATT-P07 — Amazon SES Email Integration (notification-service)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P07 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, ses, email, notification-service, aws |
| **Perspective** | Infrastructure / Backend |

---

## Story

As the notification-service, I want to send transactional emails via Amazon SES so that OMS notifications (booking confirmations, request approvals, alerts) are delivered reliably through a managed AWS email service with bounce and complaint handling — replacing the generic "email provider" placeholder from the original design.

---

## Background

`notification-service` is a FastAPI Python 3.12 service that consumes events from SQS queues and dispatches email notifications. The original design referenced a generic "email provider" — in v4.0 this is specifically Amazon SES.

**Why SES over third-party providers (SendGrid, Mailgun):**
- No API key to rotate — SES uses IAM role on the ECS task (`ses:SendEmail` permission)
- Native AWS service — consistent with the rest of the OMS AWS-first stack
- Cost: ~$0.10/1,000 emails (vs SendGrid free tier that requires rotation)
- SES configuration sets provide automatic bounce and complaint tracking via SNS

**Authentication model:** No `EMAIL_API_KEY` secret. The ECS task IAM role grants `ses:SendEmail`. `aioboto3` picks up credentials from the task execution role automatically — no explicit credential configuration in code.

**Notification types that trigger emails:**

| Trigger Event | Email Recipient | Template |
|---|---|---|
| `seat.booking.confirmed` | Employee | Booking confirmation |
| `remote.request.approved` | Employee | Remote day approved |
| `remote.request.rejected` | Employee | Remote day rejected |
| `ooo.request.approved` | Employee | OOO approved |
| `ooo.request.rejected` | Employee | OOO rejected |
| `supply.request.pending_manager` | Manager | Approval required |
| `supply.request.fulfilled` | Employee | Request fulfilled |

---

## Acceptance Criteria

### SES Domain Setup (AWS Console / Terraform)
- [ ] SES domain identity `oms.yourorg.com` created and verified.
- [ ] DKIM enabled — three CNAME records added to the Route 53 hosted zone for `oms.yourorg.com`.
- [ ] `noreply@oms.yourorg.com` address verified (or domain verification covers all addresses at the domain).
- [ ] SES configuration set `oms-email-events` created:
  - Bounce notifications → SNS topic `oms-ses-bounce` → SQS queue `oms-ses-bounce-queue`
  - Complaint notifications → SNS topic `oms-ses-complaint` → SQS queue `oms-ses-complaint-queue`
- [ ] SES sandbox exit requested for the production account. Dev/staging uses SES sandbox with a verified recipient allowlist.
- [ ] SES sending quota reviewed: default 200/day → request increase to 10,000/day before production.

### ECS Task Role
- [ ] The `notification-service` ECS task IAM role includes an inline policy:
  ```json
  {
    "Effect": "Allow",
    "Action": ["ses:SendEmail", "ses:SendRawEmail"],
    "Resource": "arn:aws:ses:eu-west-1:ACCOUNT_ID:identity/oms.yourorg.com"
  }
  ```
- [ ] No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` env vars — IAM role only.

### notification-service Code Changes
- [ ] `aioboto3` added to `requirements.txt`.
- [ ] `EmailService` class created in `app/email/service.py`:
  - Single public method: `async def send(to: str, subject: str, html: str, text: str) -> None`
  - Creates an `aioboto3.Session()` and opens an async SES client per call (or uses a shared client injected via FastAPI lifespan)
  - Uses `ses.send_email()` with `ConfigurationSetName="oms-email-events"` on every call
  - On `ClientError` with code `MessageRejected`, `MailFromDomainNotVerified`, or `AccountSendingPaused`: logs `ERROR` with `user_id`, `template_name`, `ses_error_code`; does not raise — the notification is a non-critical side effect
  - On `ClientError` with code `Throttling`: logs `WARN`; raises exception so the SQS message stays in-flight and retries (up to DLQ max receive count)
- [ ] `EmailService` injected via FastAPI `Depends` or app state (not constructed inline in handlers).
- [ ] `settings.EMAIL_FROM_ADDRESS` env var (`noreply@oms.yourorg.com`) used as `Source` on every call. Not a secret.
- [ ] `settings.AWS_REGION` env var used when constructing the SES client (`region_name=settings.AWS_REGION`).

```python
# app/email/service.py
import aioboto3
from botocore.exceptions import ClientError
import structlog
from app.config import settings

logger = structlog.get_logger()

class EmailService:
    async def send(
        self,
        to: str,
        subject: str,
        html_body: str,
        text_body: str,
        user_id: str = "",
        template_name: str = "",
    ) -> None:
        session = aioboto3.Session()
        async with session.client("ses", region_name=settings.AWS_REGION) as ses:
            try:
                await ses.send_email(
                    Source=settings.EMAIL_FROM_ADDRESS,
                    Destination={"ToAddresses": [to]},
                    Message={
                        "Subject": {"Data": subject, "Charset": "UTF-8"},
                        "Body": {
                            "Html": {"Data": html_body, "Charset": "UTF-8"},
                            "Text": {"Data": text_body, "Charset": "UTF-8"},
                        },
                    },
                    ConfigurationSetName="oms-email-events",
                )
            except ClientError as e:
                code = e.response["Error"]["Code"]
                if code == "Throttling":
                    # Raise to keep SQS message in-flight — will retry
                    logger.warning("ses_throttled", user_id=user_id, template=template_name)
                    raise
                # Non-retriable errors: log and swallow
                logger.error(
                    "ses_send_failed",
                    user_id=user_id,
                    template=template_name,
                    ses_error_code=code,
                )
```

### Email Templates
- [ ] HTML + plain-text templates implemented for all 7 notification types listed in the Background section.
- [ ] Templates stored in `app/email/templates/` as Python string constants or Jinja2 templates.
- [ ] No user PII (name, email, location name) logged — only `userId` (UUID) logged for tracing.
- [ ] All templates include a plain-text fallback (required by RFC 2045 and SES best practices).
- [ ] Unsubscribe header included: `List-Unsubscribe: <mailto:unsubscribe@oms.yourorg.com>`.

### SQS Consumer Integration
- [ ] `notification-service` SQS consumer correctly maps each incoming event type to the appropriate `EmailService.send()` call with the correct template.
- [ ] Email recipient resolved from event payload `userId` → call `auth-service GET /api/v1/users/{userId}` to retrieve email address (or event payload includes email if available).
- [ ] If user email lookup fails (404 from auth-service), log `WARN` and skip sending — do not crash the consumer.

### Tests
- [ ] Unit test: `EmailService.send()` — mock `aioboto3` SES client; assert `send_email` called with correct `Source`, `Destination`, `ConfigurationSetName`.
- [ ] Unit test: `ClientError(Throttling)` → exception re-raised (retry behaviour).
- [ ] Unit test: `ClientError(MessageRejected)` → exception swallowed, `ERROR` logged.
- [ ] Integration test: `LocalStack` SES stub — send event to SQS, assert `notification-service` consumer picks it up and calls SES `send_email` with correct template content.
- [ ] Integration test: invalid `userId` in event → `WARN` logged, consumer does not crash, SQS message deleted.

---

## Environment Variables (notification-service ECS Task Definition)

```bash
EMAIL_FROM_ADDRESS=noreply@oms.yourorg.com
AWS_REGION=eu-west-1
# No EMAIL_API_KEY — IAM role grants ses:SendEmail
```

---

## Terraform Resource Outline

```hcl
# SES domain identity
resource "aws_ses_domain_identity" "oms" {
  domain = "oms.yourorg.com"
}

# DKIM
resource "aws_ses_domain_dkim" "oms" {
  domain = aws_ses_domain_identity.oms.domain
}

# DKIM CNAME records in Route 53
resource "aws_route53_record" "ses_dkim" {
  count   = 3
  zone_id = var.route53_zone_id
  name    = "${aws_ses_domain_dkim.oms.dkim_tokens[count.index]}._domainkey.oms.yourorg.com"
  type    = "CNAME"
  ttl     = 600
  records = ["${aws_ses_domain_dkim.oms.dkim_tokens[count.index]}.dkim.amazonses.com"]
}

# SES Configuration Set
resource "aws_ses_configuration_set" "oms" {
  name = "oms-email-events"
}

# SNS topics for bounce/complaint
resource "aws_sns_topic" "ses_bounce" {
  name = "oms-ses-bounce"
}
resource "aws_sns_topic" "ses_complaint" {
  name = "oms-ses-complaint"
}

# SQS queues for bounce/complaint events
resource "aws_sqs_queue" "ses_bounce" {
  name = "oms-ses-bounce-queue"
}
resource "aws_sqs_queue" "ses_complaint" {
  name = "oms-ses-complaint-queue"
}

# SNS → SQS subscriptions
resource "aws_sns_topic_subscription" "ses_bounce" {
  topic_arn = aws_sns_topic.ses_bounce.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.ses_bounce.arn
}

# notification-service ECS task role inline policy
resource "aws_iam_role_policy" "notification_ses" {
  name = "oms-notification-ses-policy"
  role = aws_iam_role.notification_task.id
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["ses:SendEmail", "ses:SendRawEmail"]
      Resource = "arn:aws:ses:${var.aws_region}:${data.aws_caller_identity.current.account_id}:identity/oms.yourorg.com"
    }]
  })
}
```

---

## Definition of Done

- [ ] Terraform code reviewed and merged
- [ ] SES domain identity verified; DKIM CNAME records live in Route 53
- [ ] `notification-service` ECS task role updated with `ses:SendEmail` permission
- [ ] `EmailService` implemented with correct throttling vs non-retriable error handling
- [ ] All 7 email templates implemented with HTML + plain-text fallback
- [ ] No API key or AWS credentials in code, config, or env vars
- [ ] Bounce and complaint events routed to SQS queues
- [ ] Unit tests passing; integration test confirms SES `send_email` called from SQS consumer
- [ ] SES sandbox allowlist configured for dev/staging recipient addresses
- [ ] SES production sending quota increase requested before go-live
- [ ] CloudWatch Alarm created: SES bounce rate > 5% → SNS alert
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P05 / ATT-P08** (Cloud Map) — `notification-service` resolves `auth-service.oms.local` to look up user email addresses.
- Route 53 hosted zone for `oms.yourorg.com` must exist for DKIM CNAME records.
- SQS queues for `notification-service` consumers must be provisioned (M1 Terraform).
- `notification-service` SQS consumer loop (the service itself) must be implemented — this ticket only adds the SES email dispatch step within that consumer.
