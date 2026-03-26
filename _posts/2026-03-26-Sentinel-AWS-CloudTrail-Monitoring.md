---
title: "Sentinel: Building a Production-Grade AWS CloudTrail Monitoring System"
date: 2026-03-26
categories:
  - devops
tags:
  - aws
  - cloudtrail
  - security
  - python
  - fastapi
  - postgres
---

Security visibility in AWS is not optional. Every API call — a user logging in, an IAM policy changing, an S3 bucket going public — generates a CloudTrail event. The problem is that CloudTrail produces enormous volumes of raw JSON that are nearly impossible to act on without tooling. I built **Sentinel** to solve this: a self-hosted, production-ready CloudTrail monitoring platform with real-time alerting, a searchable event browser, analytics dashboards, and Slack integration.

The full source is at [github.com/Becram/aws-cloudtrail-monitoring](https://github.com/Becram/aws-cloudtrail-monitoring).

---

## Why Build This?

Managed CloudTrail alerting solutions like AWS Security Hub, GuardDuty, and CloudWatch Alarms are powerful but come with trade-offs: they charge per event ingested, they abstract away the raw data, and they make it hard to write custom rules against your specific usage patterns.

Sentinel is fully self-hosted. You own the data, you define the rules, and you pay only for the compute running the stack (which fits on a single `t3.small`).

---

## Architecture

Sentinel is a three-service system that separates ingestion, storage, and presentation cleanly.

```
 AWS Account
┌─────────────────────────────────────┐
│                                     │
│  API Calls → CloudTrail → S3 Bucket │
│                 │                   │
│                 └──→ SQS Queue      │
│                         │           │
└─────────────────────────┼───────────┘
                          │
                          ▼
┌─────────────────────────────────────┐
│  Sentinel Stack (Docker Compose)    │
│                                     │
│  ┌─────────────┐                    │
│  │ SQS Worker  │  polls SQS         │
│  │ (Python)    │  downloads S3 logs │
│  │             │  batch inserts     │
│  └──────┬──────┘                    │
│         │                           │
│         ▼                           │
│  ┌─────────────┐                    │
│  │ PostgreSQL  │  event store       │
│  │ (postgres   │  alert rules       │
│  │  15-alpine) │  config            │
│  └──────┬──────┘                    │
│         │                           │
│         ▼                           │
│  ┌─────────────┐                    │
│  │ FastAPI App │  REST API          │
│  │ (Python     │  Web UI            │
│  │  3.11)      │  Alert Engine      │
│  │             │  Slack Integration │
│  └─────────────┘                    │
└─────────────────────────────────────┘
          │
          ▼
    ┌──────────┐
    │  Slack   │  real-time alerts
    └──────────┘
```

### Data flow

1. Every AWS API call in your account is logged to CloudTrail
2. CloudTrail delivers log files to an S3 bucket and publishes a notification to an SQS queue
3. The **SQS Worker** polls the queue, downloads each log file from S3, parses the gzipped JSON, and batch-inserts events into PostgreSQL
4. The **FastAPI application** serves the web UI and REST API over the stored events
5. The **Alert Engine** runs as a background thread inside the FastAPI process, checking new events every 30 seconds against configured rules and firing Slack notifications on matches

---

## Component Deep Dive

### 1. The SQS Worker

The worker is an independent microservice. It has no imports from the main application and runs in its own container, so it can be scaled, restarted, or versioned separately.

```python
# worker/worker.py (simplified)

class CloudTrailWorker:
    def __init__(self, config: WorkerConfig):
        self.config = config
        self.sqs = boto3.client("sqs", region_name=config.aws_region)
        self.s3  = boto3.client("s3",  region_name=config.aws_region)
        self.db  = DatabaseManager(config)
        self.running = False

    def process_s3_notification(self, message: dict) -> int:
        """Download a log file from S3, parse it, insert events."""
        s3_bucket = message["s3Bucket"]
        s3_key    = message["s3ObjectKey"][0]

        response = self.s3.get_object(Bucket=s3_bucket, Key=s3_key)
        compressed = response["Body"].read()

        with gzip.open(io.BytesIO(compressed)) as f:
            log_data = json.load(f)

        events = log_data.get("Records", [])
        if events:
            self.db.batch_insert_events(events, batch_size=self.config.batch_size)

        return len(events)

    def poll_once(self):
        """One round of SQS polling."""
        messages = self.sqs.receive_message(
            QueueUrl            = self.config.sqs_queue_url,
            MaxNumberOfMessages = self.config.sqs_max_messages,
            WaitTimeSeconds     = self.config.sqs_wait_time,  # long polling
        ).get("Messages", [])

        for msg in messages:
            try:
                body = json.loads(msg["Body"])
                notification = json.loads(body.get("Message", "{}"))
                if "s3ObjectKey" in notification:
                    count = self.process_s3_notification(notification)
                    log.info("events_inserted", count=count)
                self.sqs.delete_message(
                    QueueUrl      = self.config.sqs_queue_url,
                    ReceiptHandle = msg["ReceiptHandle"],
                )
            except Exception as e:
                log.error("message_processing_failed", error=str(e))
                # message returns to queue after visibility timeout

    def run(self):
        self.running = True
        while self.running:
            self.poll_once()
            time.sleep(self.config.poll_interval)
```

Key design decisions:
- **Long polling** (20s default) reduces SQS API calls and cost
- **Batch inserts** amortise the database round-trip across 100 events by default
- **Per-message error isolation** — a bad message does not crash the loop; it returns to the queue after the visibility timeout
- **Graceful shutdown** on `SIGTERM`/`SIGINT` so Kubernetes/ECS draining works correctly

### 2. PostgreSQL Schema

Every CloudTrail event is stored in a single wide table with JSONB columns for the fields that vary by event type.

```sql
CREATE TABLE cloudtrail_events (
    id                  SERIAL PRIMARY KEY,
    event_id            VARCHAR(64) UNIQUE NOT NULL,
    event_time          TIMESTAMP NOT NULL,
    event_date          DATE NOT NULL,       -- pre-computed for date-range queries
    event_name          VARCHAR(255),
    event_source        VARCHAR(255),
    event_version       VARCHAR(32),
    aws_region          VARCHAR(64),
    source_ip_address   VARCHAR(64),
    user_agent          TEXT,
    read_only           BOOLEAN,
    management_event    BOOLEAN,

    -- identity: who made the call
    user_identity       JSONB,

    -- what was sent and what came back
    request_parameters  JSONB,
    response_elements   JSONB,

    -- affected resources
    resources           JSONB,

    created_at          TIMESTAMP DEFAULT NOW()
);

-- Indexes that matter for the query patterns Sentinel uses
CREATE INDEX idx_event_time       ON cloudtrail_events(event_time DESC);
CREATE INDEX idx_event_date       ON cloudtrail_events(event_date);
CREATE INDEX idx_event_name       ON cloudtrail_events(event_name);
CREATE INDEX idx_event_source     ON cloudtrail_events(event_source);
CREATE INDEX idx_aws_region       ON cloudtrail_events(aws_region);
CREATE INDEX idx_source_ip        ON cloudtrail_events(source_ip_address);
CREATE INDEX idx_read_only        ON cloudtrail_events(read_only);
CREATE INDEX idx_user_identity    ON cloudtrail_events USING GIN(user_identity);
CREATE INDEX idx_resources        ON cloudtrail_events USING GIN(resources);
```

The GIN indexes on the JSONB columns allow Sentinel to filter on `user_identity->>'type'` or `resources @> '[{"type":"AWS::S3::Bucket"}]'` efficiently without scanning the full table.

The connection pool is configured conservatively for a single-host deployment but can be widened:

```python
engine = create_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_recycle=3600,
    pool_pre_ping=True,   # test connections before use
)
```

### 3. The FastAPI Application

The web application is a single `webapp.py` with 20+ REST endpoints and 5 Jinja2-rendered pages. On startup it initialises the database schema and starts the alert engine background thread:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    init_database()
    alert_engine.start()
    cleanup_scheduler.start()
    yield
    # shutdown
    alert_engine.stop()
    cleanup_scheduler.stop()

app = FastAPI(title="Sentinel", lifespan=lifespan)
```

The event browser endpoint shows how filters are composed dynamically:

```python
@app.get("/api/events")
async def get_events(
    limit:        int = 100,
    offset:       int = 0,
    event_name:   str | None = None,
    event_source: str | None = None,
    aws_region:   str | None = None,
    user_type:    str | None = None,
    read_only:    bool | None = None,
    start_date:   str | None = None,
    end_date:     str | None = None,
    search:       str | None = None,
):
    filters = EventFilters(
        event_name=event_name,
        event_source=event_source,
        aws_region=aws_region,
        user_type=user_type,
        read_only=read_only,
        start_date=start_date,
        end_date=end_date,
        search=search,
    )
    events, total = get_events_with_filters(filters, limit=limit, offset=offset)
    return {"events": events, "total": total, "limit": limit, "offset": offset}
```

Pydantic validates every input before it reaches the ORM, so there is no risk of user-supplied values reaching raw SQL.

### 4. The Alert Engine

The alert engine runs as a daemon thread and checks for new events every 30 seconds. It maintains an in-memory history of the last 1000 alerts to enforce rate limiting — so a flapping condition does not flood Slack.

```python
class AlertEngine:
    def __init__(self):
        self.running   = False
        self.thread    = None
        self.history   = deque(maxlen=1000)
        self.config    = self._load_config()

    def _check_events(self):
        """Evaluate all enabled rules against recent events."""
        rules  = get_enabled_alert_rules()
        events = get_recent_events(minutes=self.config.check_interval_minutes)

        for event in events:
            for rule in rules:
                if self._matches(event, rule) and not self._rate_limited(rule):
                    self._fire(event, rule)

    def _matches(self, event: dict, rule: AlertRule) -> bool:
        field_value = self._extract_field(event, rule.field)
        match rule.operator:
            case "equals":
                return str(field_value) == rule.value
            case "contains":
                return rule.value.lower() in str(field_value).lower()
            case "regex":
                return bool(re.search(rule.value, str(field_value)))
            case "not_equals":
                return str(field_value) != rule.value
        return False

    def _fire(self, event: dict, rule: AlertRule):
        self.history.append({
            "rule_id":   rule.id,
            "event_id":  event["event_id"],
            "fired_at":  datetime.utcnow(),
        })
        if self.config.slack_enabled:
            slack_alerts.send_alert(event, rule)
```

### 5. Alert Rules

Rules are stored in PostgreSQL so they persist across restarts and can be managed through the UI or API.

```python
@dataclass
class AlertRule:
    id:          int
    name:        str
    description: str
    field:       str        # e.g. "event_name", "user_identity.type"
    operator:    str        # equals | contains | regex | not_equals
    value:       str        # the value to match against
    severity:    str        # critical | high | medium | low | info
    enabled:     bool
    rate_limit:  int        # minimum seconds between fires for this rule
```

Sentinel ships with six default rules that catch the most common security-relevant events:

| Rule | Field | Operator | Value | Severity |
|------|-------|----------|-------|----------|
| Unauthorized API calls | error_code | equals | AccessDenied | high |
| IAM policy modification | event_name | contains | PutUserPolicy | critical |
| S3 bucket policy change | event_name | contains | PutBucketPolicy | high |
| Security group change | event_name | contains | AuthorizeSecurityGroup | high |
| Root account activity | user_identity.type | equals | Root | critical |
| Console login failure | event_name + error | contains | ConsoleLogin | medium |

You can add rules via the API without restarting the engine:

```bash
curl -X POST http://localhost:8080/api/alerts/rules \
  -H "Content-Type: application/json" \
  -d '{
    "name": "KMS key deletion",
    "field": "event_name",
    "operator": "equals",
    "value": "ScheduleKeyDeletion",
    "severity": "critical",
    "rate_limit": 300
  }'
```

### 6. Slack Notifications

The Slack integration uses the `slack-sdk` and sends a structured Block Kit message with the full event context:

```python
def send_alert(event: dict, rule: AlertRule):
    client = WebClient(token=SLACK_BOT_TOKEN)
    client.chat_postMessage(
        channel=SLACK_DEFAULT_CHANNEL,
        blocks=[
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": f"{severity_emoji(rule.severity)} [{rule.severity.upper()}] {rule.name}",
                }
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": f"*Event:*\n{event['event_name']}"},
                    {"type": "mrkdwn", "text": f"*Region:*\n{event['aws_region']}"},
                    {"type": "mrkdwn", "text": f"*Source IP:*\n{event['source_ip_address']}"},
                    {"type": "mrkdwn", "text": f"*Time:*\n{event['event_time']}"},
                    {"type": "mrkdwn", "text": f"*User:*\n{extract_user(event)}"},
                    {"type": "mrkdwn", "text": f"*Service:*\n{event['event_source']}"},
                ],
            },
            {
                "type": "context",
                "elements": [{"type": "mrkdwn", "text": f"Rule: {rule.description}"}]
            },
        ],
    )
```

### 7. Automated Cleanup

CloudTrail produces hundreds of thousands of events per day in an active account. Without cleanup, the PostgreSQL volume grows unbounded. Sentinel includes a configurable retention scheduler:

```python
class CleanupPolicy:
    retention_days: int        # delete events older than this
    event_filter:   str        # all | read_only | write_only
    enabled:        bool
    last_run:       datetime
    deleted_count:  int

class CleanupScheduler:
    def _run_policy(self, policy: CleanupPolicy):
        cutoff = datetime.utcnow() - timedelta(days=policy.retention_days)

        query = delete(CloudTrailEvent).where(
            CloudTrailEvent.event_time < cutoff
        )
        if policy.event_filter == "read_only":
            query = query.where(CloudTrailEvent.read_only == True)
        elif policy.event_filter == "write_only":
            query = query.where(CloudTrailEvent.read_only == False)

        result = session.execute(query)
        return result.rowcount
```

A typical production policy: keep write events for 90 days, keep read-only events for 7 days. This dramatically reduces storage without losing the security-relevant trail.

---

## AWS Setup

### CloudTrail → S3 → SQS

```hcl
# terraform/cloudtrail.tf

resource "aws_cloudtrail" "sentinel" {
  name                          = "sentinel"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
}

resource "aws_s3_bucket_notification" "cloudtrail_to_sqs" {
  bucket = aws_s3_bucket.cloudtrail.id

  queue {
    queue_arn     = aws_sqs_queue.sentinel.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "AWSLogs/"
  }
}

resource "aws_sqs_queue" "sentinel" {
  name                       = "sentinel-cloudtrail"
  visibility_timeout_seconds = 300   # must be > worker processing time
  message_retention_seconds  = 86400
}

resource "aws_sqs_queue_policy" "sentinel" {
  queue_url = aws_sqs_queue.sentinel.id
  policy    = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.sentinel.arn
      Condition = {
        ArnLike = {
          "aws:SourceArn" = aws_s3_bucket.cloudtrail.arn
        }
      }
    }]
  })
}
```

### IAM for the worker

The worker needs read access to SQS and S3 only — no write access to any AWS resource:

```hcl
resource "aws_iam_policy" "sentinel_worker" {
  name = "sentinel-worker"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["sqs:ReceiveMessage", "sqs:DeleteMessage", "sqs:GetQueueAttributes"]
        Resource = aws_sqs_queue.sentinel.arn
      },
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject"]
        Resource = "${aws_s3_bucket.cloudtrail.arn}/AWSLogs/*"
      }
    ]
  })
}
```

---

## Deployment

### Local / development

```bash
git clone https://github.com/Becram/aws-cloudtrail-monitoring
cd aws-cloudtrail-monitoring

cat > .env <<EOF
POSTGRES_USER=sentinel
POSTGRES_PASSWORD=changeme
POSTGRES_DB=sentinel
AWS_REGION=us-east-1
SQS_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/123456789012/sentinel-cloudtrail
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
SLACK_BOT_TOKEN=xoxb-...
SLACK_DEFAULT_CHANNEL=#security-alerts
EOF

./docker-manage.sh start
# Web UI available at http://localhost:8080
```

### Production on ECS Fargate

```hcl
module "sentinel_webapp" {
  source           = "./modules/ecs-service"
  service_name     = "sentinel-webapp"
  cluster_name     = module.ecs_cluster.cluster_name
  container_image  = "${aws_ecr_repository.sentinel.repository_url}:latest"
  cpu              = 512
  memory           = 1024
  desired_count    = 2
  target_group_arn = module.alb.sentinel_target_group_arn
  subnets          = module.vpc.private_subnet_ids
  security_groups  = [module.vpc.app_security_group_id]

  secrets = {
    POSTGRES_PASSWORD   = aws_ssm_parameter.db_password.arn
    SLACK_BOT_TOKEN     = aws_ssm_parameter.slack_token.arn
  }

  environment_variables = {
    POSTGRES_HOST = module.rds.endpoint
    POSTGRES_USER = "sentinel"
    POSTGRES_DB   = "sentinel"
    LOG_LEVEL     = "INFO"
  }
}

module "sentinel_worker" {
  source           = "./modules/ecs-service"
  service_name     = "sentinel-worker"
  cluster_name     = module.ecs_cluster.cluster_name
  container_image  = "${aws_ecr_repository.sentinel_worker.repository_url}:latest"
  cpu              = 256
  memory           = 512
  desired_count    = 1
  # no load balancer — worker has no inbound traffic

  secrets = {
    POSTGRES_PASSWORD    = aws_ssm_parameter.db_password.arn
    AWS_ACCESS_KEY_ID    = aws_ssm_parameter.worker_key_id.arn
    AWS_SECRET_ACCESS_KEY = aws_ssm_parameter.worker_key_secret.arn
  }

  environment_variables = {
    POSTGRES_HOST   = module.rds.endpoint
    POSTGRES_USER   = "sentinel"
    POSTGRES_DB     = "sentinel"
    AWS_REGION      = "us-east-1"
    SQS_QUEUE_URL   = aws_sqs_queue.sentinel.url
    BATCH_SIZE      = "100"
    POLL_INTERVAL   = "5"
  }
}
```

### GitHub Actions CI/CD

The repo ships with two separate workflows — one for the web app and one for the worker — so they can be built and deployed independently:

```yaml
# .github/workflows/build-and-push-app.yml
on:
  push:
    branches: [main]
    paths:
      - 'webapp.py'
      - 'services/**'
      - 'models/**'
      - 'templates/**'
      - 'Dockerfile'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE }}
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
      - run: |
          docker build -t sentinel-webapp .
          docker tag sentinel-webapp $ECR_REGISTRY/sentinel-webapp:$GITHUB_SHA
          docker push $ECR_REGISTRY/sentinel-webapp:$GITHUB_SHA
      - run: |
          aws ecs update-service \
            --cluster production \
            --service sentinel-webapp \
            --force-new-deployment
```

---

## Web UI

Sentinel provides four pages built with Bootstrap 5 and server-side Jinja2 rendering.

**Dashboard** — summary cards showing total events, events today, active alerts, and recent activity. A sparkline chart renders events per hour for the last 24 hours.

**Event Browser** — paginated, filterable table of all CloudTrail events. Filters: event name, service (ec2.amazonaws.com, s3.amazonaws.com, etc.), region, user type, read/write, date range, and full-text search across event name and user identity.

**Analytics** — aggregated views: top event names, top IAM users, top source IPs, events by region, events over time. Built from SQL aggregations served as JSON to Chart.js.

**Alerts** — list of recent alert fires with severity, rule name, matched event, and timestamp. Start/stop the alert engine, enable/disable individual rules, configure Slack.

**Cleanup** — preview how many events a retention policy would delete before executing it. Run manually or let the scheduler handle it.

---

## Performance Characteristics

With proper indexing, Sentinel handles:

| Operation | Typical latency |
|-----------|----------------|
| Event list (paginated 100) | < 50ms |
| Filtered search | < 200ms |
| Analytics aggregation (7 days) | < 500ms |
| Batch insert (100 events) | < 100ms |
| Alert engine cycle (30s) | < 1s |

A single `t3.small` running all three containers (web app, worker, PostgreSQL) handles ~50 API calls per second and ingests several hundred CloudTrail events per minute without issues. For larger accounts generating thousands of events per minute, scale the worker horizontally (multiple worker containers reading from the same SQS queue) and move PostgreSQL to RDS.

---

## Production Checklist

- [ ] CloudTrail enabled for all regions (`is_multi_region_trail = true`)
- [ ] Log file validation enabled (`enable_log_file_validation = true`)
- [ ] S3 bucket blocked from public access
- [ ] SQS visibility timeout set to 5× the worker's expected processing time
- [ ] Worker IAM policy scoped to read-only on S3 and SQS only
- [ ] PostgreSQL volume backed up (RDS automated backups or EBS snapshots)
- [ ] Cleanup policy configured — 90 days write events, 7 days read events
- [ ] Slack alert channel monitored by on-call rotation
- [ ] All default alert rules enabled, especially Root activity and IAM modifications

---

## Conclusion

Sentinel turns the firehose of raw CloudTrail events into something actionable. The architecture is intentionally simple — three containers, one database, no message brokers beyond the SQS queue that AWS already manages for you. The alert rules are stored in the database so you can tune them at runtime without a deployment. The cleanup scheduler keeps storage costs bounded. And everything runs on infrastructure you control.

Source code, documentation, and a five-minute quick-start guide are all at [github.com/Becram/aws-cloudtrail-monitoring](https://github.com/Becram/aws-cloudtrail-monitoring).
