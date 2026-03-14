# Design a Notification System

Examples: Apple Push Notification Service (APNs), Firebase Cloud Messaging (FCM), email/SMS platforms

---

## 1. Requirements

### Functional
- Send notifications via push, SMS, email, and in-app channels
- User preference management (opt-in/out per channel)
- Template-based notification rendering
- Scheduled and triggered notifications
- Delivery tracking and analytics

### Non-Functional
- 1B notifications/day
- Push delivery < 1 second for real-time events
- At-least-once delivery guarantee
- Handle provider failures (APNs, FCM, Twilio, SendGrid)
- Rate limiting to prevent spamming users

---

## 2. High-Level Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                   Event Sources                          │
  │  ┌───────────┐  ┌───────────┐  ┌───────────┐           │
  │  │ App Events│  │ Scheduled │  │ Marketing │           │
  │  │ (order    │  │ (cron job)│  │ Campaign  │           │
  │  │  shipped) │  │           │  │ Service   │           │
  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘           │
  └────────┼──────────────┼──────────────┼──────────────────┘
           │              │              │
           └──────────────┼──────────────┘
                          │
               ┌──────────▼──────────┐
               │  Notification       │
               │  Service            │
               │  • Validate         │
               │  • Check prefs      │
               │  • Render template  │
               │  • Rate limit       │
               └──────────┬──────────┘
                          │
               ┌──────────▼──────────┐
               │  Message Queue      │
               │  (Kafka)            │
               │  Partitioned by     │
               │  channel type       │
               └──────────┬──────────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │  Push Worker │ │  SMS Worker  │ │  Email Worker│
  │              │ │              │ │              │
  │  APNs / FCM │ │  Twilio /    │ │  SendGrid /  │
  │              │ │  SNS         │ │  SES         │
  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
               ┌──────────▼──────────┐
               │  Delivery Tracker   │
               │  (sent, delivered,  │
               │   opened, failed)   │
               └─────────────────────┘
```

---

## 3. Notification Processing Pipeline

```
  Event: "Order #12345 shipped"

  Step 1: Notification Service
  ┌──────────────────────────────────────────────┐
  │  1. Receive event                            │
  │  2. Look up user preferences:                │
  │     user_id: 42                              │
  │     push: enabled                            │
  │     email: enabled                           │
  │     sms: disabled                            │
  │  3. Check rate limit:                        │
  │     user:42 → 15 notifs in last hour (< 20) │
  │     → ALLOW                                  │
  │  4. Render template:                         │
  │     "Hi {{name}}, order #{{id}} shipped!"    │
  │     → "Hi Abdul, order #12345 shipped!"      │
  │  5. Enqueue messages to Kafka:               │
  │     → push topic (for push worker)           │
  │     → email topic (for email worker)         │
  └──────────────────────────────────────────────┘

  Step 2: Channel Workers
  ┌──────────────────────────────────────────────┐
  │  Push Worker:                                │
  │  1. Dequeue from push topic                  │
  │  2. Look up device tokens for user_id:42     │
  │     → [ios_token_1, android_token_2]         │
  │  3. Send to APNs and FCM                     │
  │  4. Handle response:                         │
  │     - Success → mark delivered               │
  │     - Token invalid → remove token           │
  │     - Rate limited → retry with backoff      │
  │     - Provider down → retry from queue       │
  └──────────────────────────────────────────────┘
```

---

## 4. Priority & Rate Limiting

```
  Priority levels:
  ┌──────────┬──────────────────────────────────┐
  │ Priority │ Examples                          │
  ├──────────┼──────────────────────────────────┤
  │ P0 (now) │ 2FA code, security alert         │
  │ P1 (fast)│ Order shipped, payment received  │
  │ P2 (norm)│ New follower, weekly digest       │
  │ P3 (low) │ Marketing, recommendations       │
  └──────────┴──────────────────────────────────┘

  Separate Kafka topics per priority:
  P0 → dedicated consumer group (always processed first)
  P1-P3 → weighted fair queuing

  Rate limiting:
  ┌──────────────────────────────────────────────┐
  │  Per-user limits:                            │
  │    Push: max 20/hour, 100/day                │
  │    Email: max 5/day                          │
  │    SMS: max 3/day                            │
  │                                              │
  │  Implementation: Redis sliding window        │
  │    INCR user:{id}:push:hourly                │
  │    EXPIRE 3600                               │
  │                                              │
  │  Over limit → drop P3, buffer P2, send P0/P1│
  └──────────────────────────────────────────────┘
```

---

## 5. Retry & Failure Handling

```
  Retry strategy (exponential backoff):

  Attempt 1: immediate
  Attempt 2: wait 1s
  Attempt 3: wait 4s
  Attempt 4: wait 16s
  Attempt 5: wait 64s
  Give up → move to dead letter queue (DLQ)

  Provider failover:
  ┌──────────────────────────────────────────────┐
  │  Primary: APNs                               │
  │  If APNs fails 3× → try fallback:           │
  │    iOS: Batch via APNs HTTP/2 (retry)        │
  │    Android: FCM → fallback to direct socket  │
  │  Email: SendGrid → fallback to SES           │
  │  SMS: Twilio → fallback to Vonage            │
  │                                              │
  │  Circuit breaker per provider:               │
  │  If >50% failures in 1 min → OPEN circuit   │
  │  → route to fallback provider                │
  │  After 30s → HALF-OPEN → test with 1 request│
  └──────────────────────────────────────────────┘
```

---

## 6. Delivery Tracking

```
  Notification lifecycle:
  ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────┐
  │ Created  │──▶│  Sent    │──▶│ Delivered │──▶│  Opened  │
  └──────────┘   └──────────┘   └───────────┘   └──────────┘
       │              │               │               │
       │              │               │               └── tracking pixel
       │              │               └── APNs/FCM callback
       │              └── worker confirmed send
       └── enqueued in Kafka

  Analytics stored in ClickHouse:
  ┌──────────────────────────────────────┐
  │  notification_id │ UUID             │
  │  user_id         │ BIGINT           │
  │  channel         │ push/email/sms   │
  │  status          │ sent/delivered/  │
  │                  │ opened/failed    │
  │  sent_at         │ TIMESTAMP        │
  │  delivered_at    │ TIMESTAMP        │
  │  opened_at       │ TIMESTAMP        │
  │  campaign_id     │ UUID             │
  └──────────────────────────────────────┘
```

---

## 7. Fault Tolerance

- **Kafka ensures durability:** Messages persisted to disk; consume with at-least-once semantics
- **Idempotency:** Deduplicate by notification_id before sending to provider
- **Worker crash:** Kafka consumer rebalances; another worker picks up partition
- **Provider outage:** Circuit breaker + failover to alternate provider
- **User device offline:** APNs/FCM handle queueing; deliver when device reconnects

---

---

## 8. Low-Level Design (LLD)

### API Contract
```
  Send Notification:
  POST   /api/v1/notifications/send
         { "recipient_id": "user_123",
           "template_id": "order_shipped",
           "channels": ["push", "email", "sms"],
           "data": { "order_id": "o789", "tracking_url": "..." },
           "priority": "high",
           "idempotency_key": "uuid-abc",
           "schedule_at": null }     (null = immediate)

  Batch / Campaign:
  POST   /api/v1/notifications/campaign
         { "segment": "active_users_30d",
           "template_id": "promo_summer",
           "channels": ["push", "email"],
           "throttle_rps": 10000,
           "schedule_at": "2024-03-15T09:00:00Z" }

  User Preferences:
  GET    /api/v1/users/{id}/notification-preferences
  PUT    /api/v1/users/{id}/notification-preferences
         { "push": true, "email": true, "sms": false,
           "quiet_hours": { "start": "22:00", "end": "08:00", "tz": "US/Pacific" },
           "categories": { "marketing": false, "transactional": true } }

  History:
  GET    /api/v1/users/{id}/notifications?limit=50&cursor=...
         Response: { "notifications": [{ "id": "n1", "title": "...",
           "read": false, "channel": "push", "created_at": "..." }] }
  PUT    /api/v1/notifications/{id}/read
```

### Notification Processing Pipeline
```
  ┌────────────────────────────────────────────────────────┐
  │  1. Ingest: API receives notification request          │
  │     - Validate template + recipient                    │
  │     - Dedup by idempotency_key (Redis SETNX, 24h TTL) │
  │     - Write to Kafka (topic: notifications.ingest)     │
  │                                                        │
  │  2. Router:                                            │
  │     - Read user preferences (which channels enabled)   │
  │     - Apply quiet hours (defer if in quiet period)     │
  │     - Rate limit check (max 10 notifs/hour per user)  │
  │     - Fan out to per-channel Kafka topics:             │
  │       notifications.push, notifications.email,         │
  │       notifications.sms                                │
  │                                                        │
  │  3. Channel Workers:                                   │
  │     Push: build payload → FCM/APNs batch API           │
  │     Email: render template (MJML) → SES/SendGrid      │
  │     SMS: format message → Twilio/SNS                   │
  │     In-app: write to notification store + WebSocket    │
  │                                                        │
  │  4. Delivery Tracking:                                 │
  │     - FCM/APNs callback → update delivery status      │
  │     - Email: webhook (delivered/bounced/clicked)        │
  │     - SMS: delivery receipt via webhook                 │
  │     - Write status to notifications.status Kafka topic │
  │     - Persist in ClickHouse for analytics               │
  └────────────────────────────────────────────────────────┘
```

### Database Schema
```
  Notifications (Cassandra):
  ┌────────────────────────────────────────────────────────┐
  │  PK: (user_id, day_bucket)                             │
  │  CK: (created_at DESC, notification_id)               │
  │                                                        │
  │  notification_id  UUID                                 │
  │  template_id      TEXT                                 │
  │  title            TEXT                                 │
  │  body             TEXT                                 │
  │  channel          TEXT      (push/email/sms/in_app)    │
  │  status           TEXT      (sent/delivered/read/failed)│
  │  data             MAP<TEXT,TEXT>                        │
  │  created_at       TIMESTAMP                            │
  │  read_at          TIMESTAMP  (nullable)                │
  └────────────────────────────────────────────────────────┘

  Device Tokens (Redis + PostgreSQL):
  ┌────────────────────────────────────────────────────────┐
  │  Redis: device_tokens:{user_id} → SET of tokens       │
  │  PostgreSQL (source of truth):                         │
  │    user_id, device_token, platform, app_version,       │
  │    last_active, created_at                             │
  └────────────────────────────────────────────────────────┘
```

---

## 9. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Normal Operations (~10K notifs/sec):                   │
  │  • 5 router workers, 10 channel workers per channel    │
  │  • Kafka: 50 partitions per topic                       │
  │                                                         │
  │  Campaign Burst (~1M notifs in minutes):                │
  │  • Throttle at ingest: configurable RPS                 │
  │  • Auto-scale workers based on Kafka consumer lag       │
  │  • Provider batch APIs: FCM supports 500 tokens/request│
  │  • Backpressure: if provider rate-limited, slow ingest │
  │                                                         │
  │  Global:                                                │
  │  • Per-region notification pipeline                     │
  │  • User → region affinity (send from closest region)   │
  │  • Template and preference cache in each region         │
  │                                                         │
  │  Storage:                                               │
  │  • Notification history: Cassandra (partition per user) │
  │  • Analytics: ClickHouse (time-series queries)          │
  │  • 90-day retention for history, 1-year for analytics  │
  └─────────────────────────────────────────────────────────┘
```

---

## 10. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Message Durability:                                    │
  │  • Kafka: acks=all, RF=3 for all notification topics   │
  │  • Consumer commits offset after successful delivery   │
  │  • If worker crashes: unacked messages re-delivered     │
  │                                                         │
  │  Idempotency:                                          │
  │  • idempotency_key: Redis SETNX with 24h TTL           │
  │  • Prevents duplicate sends on API retry               │
  │  • Channel-level dedup: FCM collapse_key               │
  │                                                         │
  │  At-Least-Once Delivery:                               │
  │  • Guarantee: every notification processed at least once│
  │  • Client dedup: notification_id in local storage      │
  │  • Exactly-once NOT guaranteed (acceptable trade-off)   │
  │                                                         │
  │  Dead Letter Queue:                                    │
  │  • Failed notifications → DLQ after 3 retries          │
  │  • Manual review + re-drive from DLQ                   │
  │  • Alert if DLQ depth > threshold                       │
  │                                                         │
  │  Provider Tokens:                                       │
  │  • Device token changes: APNs feedback → update DB     │
  │  • Stale tokens: remove after 3 consecutive failures   │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ API → Kafka (ingest)         │ 10 ms    │ 50 ms   │
  │ Kafka → delivery (real-time) │ 500 ms   │ 3 s     │
  │ Push notification (FCM/APNs) │ 200 ms   │ 2 s     │
  │ Email (SES)                  │ 1 s      │ 5 s     │
  │ SMS (Twilio)                 │ 2 s      │ 10 s    │
  │ In-app (WebSocket)           │ 100 ms   │ 500 ms  │
  │ Notification history load    │ 50 ms    │ 200 ms  │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • FCM batch API: send to 500 tokens per request       │
  │  • Template pre-rendering: cache rendered templates    │
  │  • Preference cache (Redis): avoid DB lookup per notif │
  │  • Priority queues: high-priority bypass campaign queue│
  │  • Connection pooling to FCM/SES/Twilio                │
  │  • Async delivery tracking (non-blocking)              │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ FCM/APNs outage        │ Buffer in Kafka; retry with     │
  │                        │ exponential backoff; fallback   │
  │                        │ to alternative channel (email)  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Email provider down    │ Multi-provider: failover from   │
  │ (SES)                  │ SES to SendGrid automatically  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Consumer crash mid-    │ Kafka offset not committed;     │
  │ delivery               │ message reprocessed (at-least-  │
  │                        │ once)                           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Campaign creating      │ Throttle at source; per-channel │
  │ back-pressure          │ Kafka topics; auto-scale workers│
  ├────────────────────────┼─────────────────────────────────┤
  │ Invalid device token   │ Remove after 3 consecutive      │
  │                        │ failures; periodic cleanup job  │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 13. Availability

```
  Target: 99.99% for transactional; 99.9% for marketing

  ┌─────────────────────────────────────────────────────────┐
  │  Transactional (OTP, password reset, payment confirm):  │
  │  • Separate high-priority Kafka topic                   │
  │  • Dedicated worker pool (not shared with campaigns)    │
  │  • Multi-provider: instant failover                     │
  │  • In-memory queue as backup if Kafka down              │
  │                                                         │
  │  Marketing / Campaign:                                  │
  │  • Lower SLA: delay acceptable                          │
  │  • Throttled delivery to smooth load                    │
  │  • Scheduled: can retry if initial window missed        │
  │                                                         │
  │  Infrastructure:                                        │
  │  • API: multi-AZ, auto-scaling                          │
  │  • Kafka: 3 AZs, auto-recovery                         │
  │  • Workers: Kubernetes pods, auto-restart               │
  │  • Preference store (Redis): cluster with failover     │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Data Protection:                                       │
  │  • PII in notifications: encrypt at rest                │
  │  • Device tokens: encrypted in DB                       │
  │  • Email content: TLS to provider (SES enforces TLS)   │
  │  • Notification content: redacted in logs               │
  │                                                         │
  │  Authentication:                                        │
  │  • API: OAuth2 bearer token per service                 │
  │  • Webhook callbacks: HMAC signature verification       │
  │  • Provider credentials: stored in Vault                │
  │                                                         │
  │  Anti-Abuse:                                            │
  │  • Per-user rate limits (10 notifs/hour default)         │
  │  • Campaign approval workflow for marketing sends       │
  │  • Unsubscribe: one-click in every email (CAN-SPAM)    │
  │  • SMS opt-out: respond STOP to unsubscribe             │
  │                                                         │
  │  Compliance:                                            │
  │  • CAN-SPAM: unsubscribe link + physical address        │
  │  • GDPR: consent tracking, right to be forgotten        │
  │  • TCPA (SMS): explicit opt-in required                 │
  │  • Audit trail: all sends logged with recipient consent │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Provider Costs (largest expense):                      │
  │  • Push (FCM/APNs): FREE                                │
  │  • Email (SES): $0.10 per 1K emails                     │
  │  • SMS (Twilio): $0.0075 per SMS (US)                   │
  │  • SMS international: $0.01-0.10 per SMS                │
  │                                                         │
  │  At Scale (1Bn notifications/month):                    │
  │  • 800M push: $0                                        │
  │  • 150M email: $15K/month                               │
  │  • 50M SMS: $375K/month (SMS dominates cost!)           │
  │                                                         │
  │  Infrastructure:                                        │
  │  • Kafka (6 brokers): ~$5K/month                        │
  │  • Workers (30 instances): ~$4K/month                   │
  │  • Cassandra (notification store): ~$3K/month           │
  │  • Redis (preferences + dedup): ~$1K/month             │
  │  • Total infra: ~$13K/month                             │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Prefer push over SMS (free vs $0.0075/msg)           │
  │  • Smart channel selection: push first, SMS as fallback│
  │  • Batch SMSes for non-urgent notifications             │
  │  • Compress email HTML (smaller payload → lower cost)  │
  │  • Deduplicate: don't send same notification twice      │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How to prevent notification spam?** — Per-user rate limits, user preferences, priority-based dropping
2. **How to handle millions of notifications for a campaign?** — Batch process: read user segments, fan out to Kafka, workers send in parallel
3. **Push vs pull for in-app notifications?** — Push for real-time; pull (polling/pagination) for notification history
4. **How to track email opens?** — Tracking pixel (1×1 transparent image with unique URL); not 100% reliable due to image blocking
5. **Exactly-once delivery possible?** — No, at-least-once is practical. Client deduplicates by notification_id.
