# Design a Cloud Logging Platform

Examples: ELK Stack, AWS CloudWatch Logs, Google Cloud Logging, Datadog Logs

---

## 1. Requirements

### Functional
- Collect logs from applications, infrastructure, and cloud services
- Structured and unstructured log ingestion
- Full-text search and field-based filtering
- Dashboards and alerting on log patterns
- Log-based metrics extraction

### Non-Functional
- Ingest 1M+ log events/sec
- Sub-second search for recent logs
- Cost-efficient long-term retention (months to years)
- Multi-tenant isolation
- Compliance (audit logs cannot be deleted)

---

## 2. High-Level Architecture

```
  ┌──────────────────────────────────────────────────────┐
  │                  Log Sources                          │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐ │
  │  │ K8s  │  │ VMs  │  │ Lambda│  │ DB   │  │ LB   │ │
  │  │ Pods │  │      │  │      │  │ Audit│  │ Access│ │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘ │
  └─────┼─────────┼────────┼─────────┼──────────┼──────┘
        │         │        │         │          │
  ┌─────▼─────────▼────────▼─────────▼──────────▼──────┐
  │              Collection Agents                       │
  │  Fluentd / Fluent Bit / Vector / Filebeat           │
  │  • Tail log files                                   │
  │  • Parse + enrich                                   │
  │  • Buffer + batch + compress                        │
  └─────────────────────────┬────────────────────────────┘
                            │
  ┌─────────────────────────▼────────────────────────────┐
  │              Ingestion Pipeline                       │
  │  ┌──────────────┐    ┌──────────────┐                │
  │  │  Kafka        │───▶│  Stream       │                │
  │  │  (buffer)     │    │  Processor    │                │
  │  │               │    │  • Parse      │                │
  │  │               │    │  • Enrich     │                │
  │  │               │    │  • Route      │                │
  │  └──────────────┘    └──────┬───────┘                │
  └─────────────────────────────┼────────────────────────┘
                                │
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
  ┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
  │   Hot Storage     │ │ Warm Storage │ │ Cold Storage │
  │  (Elasticsearch)  │ │  (reduced    │ │ (S3 / GCS)  │
  │  0-7 days         │ │   replicas)  │ │  90+ days   │
  │  Full index       │ │  7-90 days   │ │  Compressed │
  └──────────────────┘ └──────────────┘ └──────────────┘
               │                │                │
               └────────────────┼────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │     Query Engine       │
                    │  • Full-text search    │
                    │  • Aggregations        │
                    │  • Alerting rules      │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Dashboard (Kibana /  │
                    │   Grafana)             │
                    └───────────────────────┘
```

---

## 3. Log Processing Pipeline

```
  Raw log line:
  "2026-03-12T10:15:30.123Z ERROR [payment-svc] Failed to charge card: timeout"

  After processing:
  ┌──────────────────────────────────────────────┐
  │  {                                           │
  │    "timestamp": "2026-03-12T10:15:30.123Z",  │
  │    "level": "ERROR",                         │
  │    "service": "payment-svc",                 │
  │    "message": "Failed to charge card",       │
  │    "error_type": "timeout",                  │
  │    "host": "web-03.us-east-1",               │
  │    "trace_id": "abc123def456",               │
  │    "tenant_id": "acme-corp",                 │
  │    "environment": "production",              │
  │    "kubernetes": {                           │
  │      "namespace": "payments",                │
  │      "pod": "payment-svc-7b9f4d-xk2lm",    │
  │      "container": "main"                     │
  │    }                                         │
  │  }                                           │
  └──────────────────────────────────────────────┘

  Processing steps:
  1. Parse: extract structured fields from log line
  2. Enrich: add metadata (hostname, K8s labels, geo)
  3. Redact: mask PII (credit card numbers, emails)
  4. Route: tenant_id → tenant-specific index
  5. Sample: drop DEBUG logs or sample at 10% for noisy services
```

---

## 4. Storage Strategy

```
  Index Lifecycle Management:

  Day 0-7:   HOT TIER
  ┌──────────────────────────────────────┐
  │  • SSD storage                       │
  │  • 2 replicas per shard              │
  │  • All fields indexed                │
  │  • Fast queries (< 1s)               │
  │  • Cost: $$$                         │
  └──────────────────────────────────────┘
             │ (auto-transition after 7 days)
             ▼
  Day 7-90:  WARM TIER
  ┌──────────────────────────────────────┐
  │  • HDD storage                       │
  │  • 1 replica, force-merged shards    │
  │  • Read-only indexes                 │
  │  • Acceptable queries (< 5s)         │
  │  • Cost: $$                          │
  └──────────────────────────────────────┘
             │ (auto-transition after 90 days)
             ▼
  Day 90+:   COLD TIER
  ┌──────────────────────────────────────┐
  │  • Object store (S3)                 │
  │  • Searchable snapshots              │
  │  • No local replicas                 │
  │  • Slow queries (seconds-minutes)    │
  │  • Cost: $                           │
  └──────────────────────────────────────┘
             │ (delete after 365 days)
             ▼
           DELETED
```

---

## 5. Multi-Tenancy

```
  Tenant isolation approaches:

  Option A: Separate indices per tenant
  ┌──────────────────────────────────────┐
  │  tenant-acme-logs-2026.03.12         │
  │  tenant-globex-logs-2026.03.12       │
  │  tenant-initech-logs-2026.03.12      │
  │                                      │
  │  Pro: Strong isolation, easy cleanup │
  │  Con: Many indices → cluster overhead│
  └──────────────────────────────────────┘

  Option B: Shared indices with tenant field
  ┌──────────────────────────────────────┐
  │  logs-2026.03.12 (all tenants)       │
  │  Filter: tenant_id = "acme"          │
  │                                      │
  │  Pro: Fewer indices, simpler mgmt    │
  │  Con: Noisy neighbor, less isolation │
  └──────────────────────────────────────┘

  Recommended: Hybrid
  Large tenants → dedicated indices
  Small tenants → shared indices with routing
```

---

## 6. Alerting on Logs

```
  Alert rule: "More than 100 ERROR logs from payment-svc in 5 minutes"

  ┌──────────────────────────────────────────────┐
  │  Evaluation engine:                          │
  │  Every 1 minute:                             │
  │    count = query("level:ERROR AND            │
  │                   service:payment-svc         │
  │                   AND @timestamp > now-5m")   │
  │    if count > 100:                           │
  │      fire alert → PagerDuty, Slack           │
  │                                              │
  │  Log-based metrics:                          │
  │  Extract: error_count{service="payment"} = N │
  │  Push to Prometheus → graph in Grafana       │
  └──────────────────────────────────────────────┘
```

---

## 7. Fault Tolerance

- **Agent failure:** Buffer to disk; resume on restart
- **Kafka buffer:** Absorbs spikes; replay if processor crashes
- **Elasticsearch node failure:** Replicas serve reads; auto-rebalance
- **Data loss prevention:** At-least-once delivery from agent → Kafka → ES
- **Backpressure:** If ES is slow, Kafka retains; processors slow down

---

---

## 8. Low-Level Design (LLD)

### API Contract
```
  Ingestion:
  POST   /api/v1/logs/ingest
         Header: X-Tenant-ID: tenant_abc
         Body (JSON lines):
           {"timestamp":"2024-03-13T10:00:00Z","severity":"ERROR",
            "service":"payment","message":"timeout","trace_id":"abc123",
            "labels":{"env":"prod","region":"us-east-1"}}

  POST   /api/v1/logs/ingest/otlp    (OpenTelemetry Protocol)
         Body: protobuf ExportLogsServiceRequest

  Query:
  POST   /api/v1/logs/query
         { "query": "service=payment AND severity=ERROR",
           "start": "2024-03-13T09:00:00Z", "end": "2024-03-13T10:00:00Z",
           "limit": 1000, "sort": "desc" }
         Response: { "logs": [...], "next_cursor": "..." }

  GET    /api/v1/logs/tail?query=service=payment
         Response: SSE stream of matching log lines

  Alerting:
  POST   /api/v1/alerts/rules
         { "name": "high_error_rate", "query": "severity=ERROR",
           "window": "5m", "threshold": 100, "channel": "pagerduty" }

  Admin:
  GET    /api/v1/tenants/{id}/usage   → bytes ingested, stored, queried
  PUT    /api/v1/tenants/{id}/quota   → set ingestion rate limit
```

### Log Processing Pipeline Internals
```
  ┌────────────────────────────────────────────────────────┐
  │  Agent (Fluent Bit / Vector):                          │
  │    File tail → parse (regex/JSON) → buffer (disk)     │
  │    → enrich (K8s metadata: pod, ns, container)        │
  │    → filter (drop DEBUG, sample /healthz)              │
  │    → compress (gzip) → batch (5MB or 5s)              │
  │    → send to gateway (HTTP/gRPC with retry)           │
  │                                                        │
  │  Gateway (Distributor):                                │
  │    Receive batch → validate tenant + rate limit        │
  │    → hash by labels → route to ingester shard          │
  │                                                        │
  │  Ingester:                                             │
  │    Receive chunks → build in-memory chunk              │
  │    → WAL for durability → when chunk full (1MB/5min)  │
  │    → compress (snappy/lz4) → flush to object store    │
  │    → update index (label → chunk pointer)              │
  │                                                        │
  │  Index:                                                │
  │    Loki: label-based index only (DynamoDB/Cassandra)   │
  │    ELK: full inverted index (Elasticsearch)             │
  │    ┌───────────────────────────────────┐                │
  │    │ tenant=abc, service=payment,     │                │
  │    │ 2024-03-13 10:00-10:05           │                │
  │    │ → chunks: [s3://bucket/chunk_42] │                │
  │    └───────────────────────────────────┘                │
  └────────────────────────────────────────────────────────┘
```

### Storage Layer
```
  Chunk Object Format (Loki):
  ┌────────────────────────────────────────────────────────┐
  │  Header:                                               │
  │    Magic bytes + version                               │
  │    Label set hash                                      │
  │    Time range: [from_ts, through_ts]                   │
  │    Compression: snappy                                 │
  │    Entries count: 5000                                 │
  │                                                        │
  │  Body (compressed):                                    │
  │    [timestamp_delta, log_line_length, log_line_bytes]  │
  │    ... repeated for each entry                         │
  │                                                        │
  │  Footer:                                               │
  │    CRC32 checksum                                      │
  │    Metadata (structured labels extracted)               │
  └────────────────────────────────────────────────────────┘
```

---

## 9. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion Scaling:                                     │
  │  • Agent per node (DaemonSet): scales with cluster     │
  │  • Distributor: stateless, scale horizontally           │
  │  • Ingester: shard by label hash (consistent hashing)  │
  │  • 10 ingesters → ~100 GB/day each → 1 TB/day total   │
  │                                                         │
  │  Query Scaling:                                         │
  │  • Querier: stateless, scale horizontally               │
  │  • Query-frontend: split time ranges, parallel exec    │
  │  • Query result cache: Memcached                        │
  │  • Bloom filters: skip chunks that can't match query   │
  │                                                         │
  │  Storage Scaling:                                       │
  │  • Object store: infinite (S3/GCS)                      │
  │  • Tiering: hot (30d, fast query) → cold (archive)     │
  │  • Compaction: merge small chunks into larger           │
  │  • Index: shard by tenant + time period                 │
  │                                                         │
  │  Multi-Cluster:                                         │
  │  • Per-cluster agents → central logging platform        │
  │  • Cross-cluster query federation                       │
  └─────────────────────────────────────────────────────────┘
```

---

## 10. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Agent Level:                                          │
  │  • File position tracking: resume from last offset     │
  │  • Disk buffer (100MB default): survive downstream     │
  │    outage; queue logs until ingester is back            │
  │  • At-least-once delivery: retry on failure            │
  │                                                         │
  │  Ingester Level:                                       │
  │  • WAL: all received logs written to WAL before ACK    │
  │  • Replication factor 3: write to 3 ingesters          │
  │  • Crash recovery: replay WAL, re-flush chunks         │
  │                                                         │
  │  Storage Level:                                        │
  │  • Object store: 11 nines durability (S3)              │
  │  • Chunk checksums: detect corruption on read          │
  │  • Index backup: periodic snapshot + continuous WAL     │
  │                                                         │
  │  End-to-End Guarantee:                                 │
  │  • Log delivery is at-least-once (not exactly-once)    │
  │  • Dedup at query time by timestamp + content hash     │
  │  • Missing log detection: compare log count per pod    │
  │    against container runtime log count                  │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Log → searchable             │ 3 s      │ 10 s    │
  │ Simple query (1h, 1 service) │ 200 ms   │ 2 s     │
  │ Full-text search (1h)        │ 500 ms   │ 5 s     │
  │ Aggregation (24h)            │ 2 s      │ 10 s    │
  │ Live tail                    │ 1 s      │ 3 s     │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Label-based index: narrow chunks before scanning    │
  │  • Bloom filters on chunks: skip non-matching chunks   │
  │  • Parallel chunk scanning across queriers              │
  │  • Query result cache: identical queries return cached │
  │  • Structured metadata: avoid full-text on structured  │
  │  • Pre-built log views: materialized for common queries│
  │  • Chunk size tuning: 1MB chunks → optimal scan speed │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Agent crash            │ Systemd auto-restart; file pos  │
  │                        │ tracking resumes from offset    │
  ├────────────────────────┼─────────────────────────────────┤
  │ Ingester crash         │ WAL replay on restart; replicas │
  │                        │ serve queries for lost chunks   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Log storm (10× spike)  │ Per-tenant rate limiting at     │
  │                        │ distributor; drop with warning  │
  │                        │ metric; backpressure to agent   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Object store outage    │ Ingesters buffer in memory +    │
  │                        │ WAL; retry flush; hours of      │
  │                        │ buffer capacity                 │
  ├────────────────────────┼─────────────────────────────────┤
  │ Query of death         │ Query timeout (60s); max bytes  │
  │                        │ scanned per query; circuit      │
  │                        │ breaker on expensive patterns   │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 13. Availability

```
  Target: 99.99% for ingestion; 99.9% for query

  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion Path (higher SLA):                           │
  │  • Agent disk buffer: tolerate ~1h ingester outage      │
  │  • Distributor: stateless, N+2 replicas                 │
  │  • Ingester: 3× replication, zone-aware placement       │
  │  • Write succeeds if majority replicas ACK              │
  │                                                         │
  │  Query Path:                                            │
  │  • Querier: stateless, scale horizontally               │
  │  • Store-gateway: replicated for object store access    │
  │  • Query-frontend: queue + retry on querier failure     │
  │  • Graceful degradation: return partial results if      │
  │    one querier times out                                │
  │                                                         │
  │  Multi-AZ:                                              │
  │  • All components spread across 3 AZs                   │
  │  • Zone-aware replication for ingesters                 │
  │  • AZ failure: no data loss, slight latency increase   │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Multi-Tenancy:                                         │
  │  • Tenant ID on every request (header or API key)       │
  │  • Complete data isolation: separate index + chunks     │
  │  • Cross-tenant query impossible by design              │
  │  • Per-tenant rate limits and quotas                    │
  │                                                         │
  │  PII Protection:                                        │
  │  • Pipeline stage: regex redact SSN, email, CC numbers │
  │  • Hash sensitive fields at ingestion                   │
  │  • RBAC: restrict who can query which services/labels  │
  │  • Field-level access: hide sensitive log fields        │
  │                                                         │
  │  Encryption:                                            │
  │  • In-transit: TLS for agent → gateway → ingester      │
  │  • At-rest: SSE-S3/KMS for object store                │
  │  • Index: encrypted filesystem or managed DB encryption │
  │                                                         │
  │  Compliance:                                            │
  │  • Audit logs: immutable, WORM storage                  │
  │  • Retention enforcement: auto-delete after policy      │
  │  • Right to be forgotten: delete by user ID (tombstone) │
  │  • SOC2 / HIPAA: access logging on all queries          │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Loki Architecture (label-only index):                  │
  │  • Ingest 1 TB/day, 30-day retention                   │
  │  • Storage: 30 TB × $0.023/GB (S3) = $700/month       │
  │  • Ingesters (3 × m5.2xl): ~$1.5K/month               │
  │  • Queriers (3 × c5.2xl): ~$1.5K/month                │
  │  • Index (DynamoDB): ~$500/month                       │
  │  • Total: ~$4K/month                                   │
  │                                                         │
  │  ELK Architecture (full-text index):                    │
  │  • Same 1 TB/day, 30-day retention                     │
  │  • Elasticsearch: 30 data nodes (i3.2xl) = $15K/month  │
  │  • Kibana + coordinating: ~$1K/month                   │
  │  • Total: ~$16K/month (4× Loki)                       │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Drop DEBUG/TRACE in production (50% volume reduction)│
  │  • Sample high-volume logs (keep 1 in 10 for healthz)  │
  │  • Aggressive compression: snappy → zstd (30% smaller) │
  │  • Shorter retention: 7d hot, 30d warm, 90d cold archive│
  │  • Alert on per-service ingestion spikes               │
  │                                                         │
  │  Managed (Datadog Logs):                               │
  │  • $0.10/GB ingested + $2.55/M logs retained > 15d     │
  │  • 1 TB/day → ~$100K/month (25× self-hosted Loki)     │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **ELK vs Loki architecture?** — ELK: full inverted index (rich queries, high cost). Loki: index only labels, store raw logs in object store (cheaper, simpler)
2. **How to handle log storms?** — Rate limit at agent, sample/drop verbose logs, per-service quotas
3. **How to correlate logs with traces?** — Include trace_id in every log line; query by trace_id to see all logs for a request
4. **Cost optimization?** — Aggressive tiering, drop DEBUG in production, compress, deduplicate, sample low-value logs
5. **Compliance logging?** — Separate immutable index for audit logs; WORM storage; no delete allowed
