# Design a Metrics Collection and Alerting System (Datadog)

Examples: Datadog, Prometheus + Alertmanager, Grafana Cloud, New Relic

---

## 1. Requirements

### Functional
- Collect metrics from applications, infrastructure, containers, and cloud services
- Support metric types: counter, gauge, histogram, summary
- Query metrics with a flexible language (PromQL-style)
- Define alert rules with threshold, rate-of-change, and anomaly detection
- Route alerts to PagerDuty, Slack, email, webhooks
- Dashboard visualization with time-range selection
- Multi-tenant support with per-tenant quotas

### Non-Functional
- Ingest 10M+ active time series
- Sub-second query latency for recent data
- Alert evaluation latency < 30 seconds from metric arrival to notification
- 99.99% availability for alerting path
- Horizontal scalability for ingestion and query

---

## 2. Scale Estimation

```
Agents/exporters:   100,000 hosts
Metrics per host:   100 metrics
Scrape interval:    15 seconds
Active time series: 100K × 100 = 10M
Samples per second: 10M / 15 ≈ 667K samples/sec
Sample size:        16 bytes (8-byte timestamp + 8-byte float)
Raw storage/day:    667K × 16 × 86400 ≈ 900 GB/day
With compression:   900 GB / 12 (Gorilla) ≈ 75 GB/day
Annual storage:     75 GB × 365 ≈ 27 TB/year (before downsampling)
Alert rules:        50,000 rules evaluated every 30s
```

---

## 3. High-Level Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      Metric Sources                         │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐   │
  │  │Node  │  │App   │  │K8s   │  │Cloud │  │Custom    │   │
  │  │Export │  │SDK   │  │cAdv  │  │Watch │  │Exporter  │   │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └────┬─────┘   │
  └─────┼─────────┼────────┼─────────┼────────────┼───────────┘
        │         │        │         │            │
        │ PULL    │ PUSH   │ PULL    │ PUSH       │ PULL
        │         │        │         │            │
  ┌─────▼─────────▼────────▼─────────▼────────────▼───────────┐
  │                  Ingestion Gateway                          │
  │  • Authentication (API key / mTLS)                         │
  │  • Validation (label cardinality check)                    │
  │  • Rate limiting (per-tenant quotas)                       │
  │  • Tenant routing (hash tenant_id → shard)                 │
  └──────────────────────────┬────────────────────────────────┘
                             │
                ┌────────────┼────────────┐
                ▼            ▼            ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Ingester 0  │  │  Ingester 1  │  │  Ingester 2  │
  │  WAL + Head  │  │  WAL + Head  │  │  WAL + Head  │
  │  (in-memory) │  │  (in-memory) │  │  (in-memory) │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │
                           ▼
  ┌──────────────────────────────────────────────────────────┐
  │              Long-Term Storage                            │
  │  ┌────────────────┐  ┌─────────────┐  ┌───────────────┐ │
  │  │  Hot (SSD)     │  │  Warm (HDD) │  │  Cold (S3)    │ │
  │  │  Last 48h      │  │  30 days    │  │  1 year+      │ │
  │  │  Raw samples   │  │  1m rollup  │  │  5m/1h rollup │ │
  │  └────────────────┘  └─────────────┘  └───────────────┘ │
  └──────────────────────────┬───────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Query       │  │  Alert Rule  │  │  Dashboard   │
  │  Engine      │  │  Evaluator   │  │  API         │
  │  (PromQL)    │  │  (every 30s) │  │  (Grafana)   │
  └──────────────┘  └──────┬───────┘  └──────────────┘
                           │
                    ┌──────▼───────┐
                    │  Alert       │
                    │  Manager     │
                    │  • Group     │
                    │  • Dedup     │
                    │  • Silence   │
                    │  • Route     │
                    │  • Escalate  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         PagerDuty      Slack       Email/Webhook
```

---

## 4. Ingestion Pipeline

### Push vs Pull

```
  PULL MODEL (Prometheus-style):
  ┌──────────────────────────────────────────────┐
  │  Scraper ──HTTP GET──▶ /metrics endpoint     │
  │           every 15s                          │
  │                                              │
  │  Pro: Scraper controls rate; easy to detect  │
  │       when target is down (scrape failure)   │
  │  Con: Hard with ephemeral containers;        │
  │       scraper must know all targets           │
  └──────────────────────────────────────────────┘

  PUSH MODEL (Datadog/StatsD-style):
  ┌──────────────────────────────────────────────┐
  │  Agent ──POST──▶ Ingestion Gateway           │
  │         every 10-15s (batched)               │
  │                                              │
  │  Pro: Works with serverless/ephemeral;       │
  │       no service discovery needed            │
  │  Con: Agent failure = silent data loss;      │
  │       harder to rate-limit                   │
  └──────────────────────────────────────────────┘

  HYBRID (recommended):
  Pull for long-running infrastructure (nodes, databases)
  Push for ephemeral workloads (containers, lambdas, batch jobs)
```

### Ingestion Data Flow

```
  Agent batches 1000 samples:
  ┌────────────────────────────────────────────┐
  │  POST /api/v1/push                         │
  │  {                                         │
  │    "tenant": "acme",                       │
  │    "timeseries": [                         │
  │      {                                     │
  │        "labels": {                         │
  │          "__name__": "http_requests_total", │
  │          "method": "GET",                  │
  │          "status": "200",                  │
  │          "service": "api-gateway"          │
  │        },                                  │
  │        "samples": [                        │
  │          [1710000000, 42.0],               │
  │          [1710000015, 43.0],               │
  │          [1710000030, 45.0]                │
  │        ]                                   │
  │      },                                    │
  │      ...                                   │
  │    ]                                       │
  │  }                                         │
  └────────────────────────────────────────────┘
        │
        ▼
  ┌────────────────────────────────────────────┐
  │  Ingestion Gateway:                        │
  │  1. Authenticate (API key)                 │
  │  2. Validate labels (no high-cardinality)  │
  │  3. Check tenant quota (series limit)      │
  │  4. Hash(tenant_id) → ingester shard       │
  │  5. Forward to ingester (with replication) │
  └────────────────────────────────────────────┘
        │
        ▼
  ┌────────────────────────────────────────────┐
  │  Ingester:                                 │
  │  1. Write to WAL (crash recovery)          │
  │  2. Append to in-memory Head block         │
  │  3. When Head is full (2h of data):        │
  │     → Compact to on-disk Block             │
  │     → Upload Block to object store         │
  │  4. Replicate to peer ingester (RF=2)      │
  └────────────────────────────────────────────┘
```

---

## 5. Time-Series Storage Engine

```
  Gorilla Compression (Facebook, 2015):
  ┌──────────────────────────────────────────────────┐
  │  Timestamps: Delta-of-delta encoding             │
  │                                                  │
  │  t0 = 1710000000                                 │
  │  t1 = 1710000015  →  delta = 15                  │
  │  t2 = 1710000030  →  delta = 15, DoD = 0  (1bit)│
  │  t3 = 1710000045  →  delta = 15, DoD = 0  (1bit)│
  │  t4 = 1710000075  →  delta = 30, DoD = 15 (few) │
  │                                                  │
  │  Values: XOR encoding                            │
  │  v0 = 42.0                                       │
  │  v1 = 43.0  →  XOR(v0,v1) = small diff (few bits)│
  │  v2 = 43.0  →  XOR(v1,v2) = 0 (1 bit!)          │
  │                                                  │
  │  Result: 16 bytes/sample → ~1.37 bytes/sample    │
  │          (~12× compression)                      │
  └──────────────────────────────────────────────────┘

  On-disk block structure (TSDB):
  ┌──────────────────────────────────────┐
  │  Block: 2-hour time range            │
  │  ┌──────────────────────────────┐    │
  │  │  meta.json                   │    │
  │  │  • Block ID, time range      │    │
  │  │  • Number of series/samples  │    │
  │  ├──────────────────────────────┤    │
  │  │  index                       │    │
  │  │  • Label → series ID mapping │    │
  │  │  • Posting lists for queries │    │
  │  ├──────────────────────────────┤    │
  │  │  chunks/                     │    │
  │  │  • Compressed time-value     │    │
  │  │    pairs per series          │    │
  │  ├──────────────────────────────┤    │
  │  │  tombstones                  │    │
  │  │  • Deletion markers          │    │
  │  └──────────────────────────────┘    │
  └──────────────────────────────────────┘
```

### Downsampling Pipeline

```
  Raw (15s resolution) ──kept 48h──▶ DELETE
        │
        ▼
  1-minute rollup   ──kept 30d──▶ DELETE
  (min, max, sum, count per minute)
        │
        ▼
  5-minute rollup   ──kept 90d──▶ DELETE
        │
        ▼
  1-hour rollup     ──kept 1y──▶ DELETE
        │
        ▼
  1-day rollup      ──kept 5y──▶ DELETE

  Storage savings:
  Raw 15s for 1 year:   27 TB
  With downsampling:    ~2 TB  (13.5× reduction)
```

---

## 6. Query Engine

```
  Query: rate(http_requests_total{service="api", status="500"}[5m])

  Execution plan:
  ┌─────────────────────────────────────────┐
  │  1. Parse PromQL expression             │
  │                                         │
  │  2. Label matchers → find matching      │
  │     series in index:                    │
  │     service="api" ∩ status="500"        │
  │     → posting list intersection         │
  │     → 147 matching series               │
  │                                         │
  │  3. Fetch samples for [now-5m, now]     │
  │     from Head block (in-memory)         │
  │                                         │
  │  4. Apply rate() function:              │
  │     For each series:                    │
  │       rate = (last - first) /           │
  │              (last_t - first_t)         │
  │                                         │
  │  5. Return 147 instant vectors          │
  └─────────────────────────────────────────┘

  Distributed query (Thanos/Cortex pattern):
  ┌──────────────┐
  │ Query Router │
  └──────┬───────┘
         │ fan-out
    ┌────┼────┐
    ▼    ▼    ▼
  ┌───┐┌───┐┌───┐
  │S1 ││S2 ││S3 │  (storage shards)
  └─┬─┘└─┬─┘└─┬─┘
    │    │    │
    └────┼────┘
         │ merge + dedup
  ┌──────▼───────┐
  │ Final Result │
  └──────────────┘
```

---

## 7. Alert Evaluation Engine

```
  Alert rule example:
  ┌──────────────────────────────────────────────┐
  │  - alert: HighErrorRate                      │
  │    expr: rate(http_errors_total[5m]) > 0.05  │
  │    for: 5m                                   │
  │    labels:                                   │
  │      severity: critical                      │
  │      team: payments                          │
  │    annotations:                              │
  │      summary: "Error rate > 5%"              │
  └──────────────────────────────────────────────┘

  Alert state machine:
  ┌──────────┐    expr true     ┌──────────┐   for duration   ┌──────────┐
  │ Inactive │ ──────────────▶  │ Pending  │ ──────────────▶  │ Firing   │
  └──────────┘                  └──────────┘                  └──────────┘
       ▲                             │                             │
       │         expr false          │        expr false           │
       └─────────────────────────────┘                             │
       ▲                                                           │
       │                        expr false for resolve_timeout     │
       └───────────────────────────────────────────────────────────┘

  Evaluation scheduling:
  ┌──────────────────────────────────────────────────┐
  │  50,000 alert rules                              │
  │  Evaluation interval: 30s                        │
  │  → 50K / 30s ≈ 1,667 evaluations/sec            │
  │                                                  │
  │  Shard by: hash(rule_id) % num_evaluators        │
  │  10 evaluator instances → 5,000 rules each       │
  │  Each evaluator: 167 evaluations/sec             │
  │                                                  │
  │  Evaluator state:                                │
  │  • Track per-rule: last state, pending_since     │
  │  • Checkpoint to etcd every 30s                  │
  │  • On failure: peer takes over, loads checkpoint │
  └──────────────────────────────────────────────────┘
```

---

## 8. Alert Manager

```
  Alert flow:
  Evaluator fires alert
        │
        ▼
  ┌─────────────┐
  │  Grouping    │  Group by: {service, severity}
  │              │  "3 alerts from payment-svc" → 1 notification
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Dedup       │  Same alert firing again? Suppress within
  │              │  repeat_interval (e.g., 4 hours)
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Inhibition  │  If "cluster_down" is firing,
  │              │  suppress "pod_unhealthy" alerts
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Silencing   │  Maintenance window active?
  │              │  Match labels → suppress
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Routing     │  Route by labels:
  │              │  severity=critical → PagerDuty
  │              │  severity=warning  → Slack
  │              │  team=payments     → #payments-alerts
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Escalation  │  Not ack'd in 15 min?
  │              │  → escalate to manager on-call
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────────────────────┐
  │  Notification Channels              │
  │  PagerDuty │ Slack │ Email │ Webhook│
  └─────────────────────────────────────┘
```

---

## 9. High-Cardinality Management

```
  PROBLEM:
  metric{user_id="abc123"} → millions of unique series
  
  10M users × 10 metrics = 100M series → storage explosion

  SOLUTIONS:
  ┌──────────────────────────────────────────────┐
  │  1. Ingestion-time validation                │
  │     Reject labels with cardinality > 10,000  │
  │     Return HTTP 400 with clear error message │
  │                                              │
  │  2. Pre-aggregation at agent                 │
  │     Instead of per-user: aggregate to        │
  │     per-service, per-endpoint buckets        │
  │                                              │
  │  3. Per-tenant series limits                 │
  │     Free tier: 100K series                   │
  │     Pro tier: 1M series                      │
  │     Enterprise: 10M series                   │
  │                                              │
  │  4. Adaptive sampling                        │
  │     High-cardinality dims sampled at 10%     │
  │     with count estimation                    │
  │                                              │
  │  5. Streaming top-K aggregation              │
  │     Keep only top 100 values per label       │
  │     Aggregate rest into "other" bucket       │
  └──────────────────────────────────────────────┘
```

---

## 10. Scaling Strategy

```
  Multi-tenant architecture (Cortex/Mimir pattern):

  ┌─────────────────────────────────────────────┐
  │              Global View                     │
  │                                             │
  │  Region: US-East          Region: EU-West   │
  │  ┌──────────────┐       ┌──────────────┐   │
  │  │ Ingesters    │       │ Ingesters    │   │
  │  │ Compactors   │       │ Compactors   │   │
  │  │ Queriers     │       │ Queriers     │   │
  │  │ Store-GW     │       │ Store-GW     │   │
  │  │     │        │       │     │        │   │
  │  │     ▼        │       │     ▼        │   │
  │  │  S3 bucket   │       │  S3 bucket   │   │
  │  └──────────────┘       └──────────────┘   │
  │         │                      │            │
  │         └──────────┬───────────┘            │
  │                    ▼                        │
  │           Query Federation                  │
  │           (cross-region queries)            │
  └─────────────────────────────────────────────┘

  Scaling each component independently:
  • Ingesters: scale by active series count
  • Queriers: scale by query QPS and complexity
  • Compactors: scale by block count
  • Store-gateway: scale by block count in object store
  • Alert evaluators: scale by rule count
```

---

## 11. Fault Tolerance

| Failure | Mitigation |
|---|---|
| Agent crash | Buffer to disk; resume sending on restart |
| Ingester crash | WAL replay on restart; peer holds replica |
| Storage node failure | Replicated blocks in object store; store-gateway serves from any replica |
| Alert evaluator crash | Peer takes over rules via consistent hashing; checkpoint in etcd |
| Alert manager crash | Clustered alert managers (Gossip protocol); any node can send notifications |
| Network partition | Local ingestion continues; catch up when reconnected |
| Total region failure | Failover to secondary region; multi-region alert federation |

---

## 12. Meta-Monitoring (Monitoring the Monitor)

```
  Critical: if monitoring is down, you're blind

  ┌──────────────────────────────────────────────┐
  │  Dead Man's Switch:                          │
  │  An alert that ALWAYS fires.                 │
  │  If you stop receiving it → system is down.  │
  │                                              │
  │  alert: DeadMansSwitch                       │
  │  expr: vector(1)        # always true        │
  │  → sends heartbeat to external watchdog      │
  │    (separate system like Cronitor/OpsGenie)  │
  │                                              │
  │  Self-monitoring metrics:                    │
  │  • cortex_ingester_ingested_samples_total    │
  │  • cortex_query_duration_seconds (p99)       │
  │  • cortex_alertmanager_notifications_failed  │
  │  • cortex_ingester_wal_replay_duration       │
  └──────────────────────────────────────────────┘
```

---

---

## 13. Low-Level Design (LLD)

### API Contract
```
  Metric Ingestion:
  POST   /api/v1/write           (Prometheus remote_write, protobuf)
  POST   /api/v1/import          (Datadog JSON)
         { "series": [{ "metric": "cpu.usage", "points": [[ts, val]],
           "tags": ["host:web01","env:prod"], "type": "gauge" }] }

  StatsD UDP (low-overhead):
  cpu.usage:72.5|g|#host:web01,env:prod

  OpenTelemetry (OTLP):
  POST   /v1/metrics             (protobuf ExportMetricsServiceRequest)

  Query:
  GET    /api/v1/query?query=rate(http_requests_total[5m])&time=T
  GET    /api/v1/query_range?query=...&start=T1&end=T2&step=15s
  POST   /api/v1/query           (POST for long queries)

  Alert Management:
  POST   /api/v1/alerts/rules
         { "name": "high_latency", "expr": "p99_latency > 500",
           "for": "5m", "labels": {"severity":"critical"},
           "annotations": {"summary":"P99 latency > 500ms"} }

  GET    /api/v1/alerts/active   → list currently firing alerts
  POST   /api/v1/alerts/silence  → suppress alert by matchers

  Dashboard:
  POST   /api/v1/dashboards      → create dashboard
  GET    /api/v1/dashboards/{id} → get dashboard panels
```

### Alert Evaluation Internals
```
  ┌────────────────────────────────────────────────────────┐
  │  Evaluation Loop (every eval_interval, default 15s):   │
  │                                                        │
  │  for each rule in rule_groups:                         │
  │    1. Execute PromQL query against TSDB                 │
  │    2. Compare result against threshold                  │
  │    3. If condition met:                                 │
  │       - Start "for" timer (e.g., 5m)                   │
  │       - State: PENDING                                  │
  │    4. If "for" duration elapsed with sustained breach:  │
  │       - State: FIRING                                   │
  │       - Send to Alertmanager                            │
  │    5. If condition clears:                              │
  │       - State: RESOLVED                                 │
  │       - Send resolve notification                       │
  │                                                        │
  │  Alert State Machine:                                   │
  │  INACTIVE ─(threshold crossed)─→ PENDING               │
  │  PENDING  ─(for duration met)──→ FIRING                │
  │  PENDING  ─(threshold clears)──→ INACTIVE              │
  │  FIRING   ─(threshold clears)──→ RESOLVED → INACTIVE  │
  │                                                        │
  │  Multi-Window Burn Rate (SLO alerts):                  │
  │    Fast window (1h): burn rate > 14× → page immediate  │
  │    Slow window (6h): confirm sustained → reduce noise  │
  └────────────────────────────────────────────────────────┘
```

---

## 14. Scalability (Enhanced)

```
  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion: (additional to Section 10)                  │
  │  • Write sharding by metric hash across ingesters      │
  │  • Pre-aggregation at agent (StatsD flush interval)     │
  │  • Drop high-cardinality at ingestion gateway           │
  │                                                         │
  │  Query:                                                 │
  │  • Query-frontend: split by time + shard by series      │
  │  • Parallel sub-queries across querier fleet             │
  │  • Result cache: Memcached, cache aligned time ranges   │
  │  • Recording rules: pre-compute dashboard queries       │
  │                                                         │
  │  Alert Evaluation:                                      │
  │  • Shard rules across evaluator instances               │
  │  • 100K rules across 10 evaluators                      │
  │  • Ruler stateless: re-shards on scale events            │
  │                                                         │
  │  Capacity Planning:                                     │
  │  • 10M active series: 3 ingesters, 3 queriers           │
  │  • 100M active series: 30 ingesters, 10 queriers        │
  │  • Storage: ~1.5 bytes/sample (Gorilla compression)     │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion Durability:                                  │
  │  • Agent-side buffer: disk-backed queue (30-60 min)     │
  │  • Ingester WAL: every sample WAL'd before in-memory    │
  │  • Replication factor 3: write to 3 ingesters           │
  │  • WAL replay on crash: rebuild head block              │
  │                                                         │
  │  Storage Durability:                                    │
  │  • Compacted blocks uploaded to S3 (11 nines)            │
  │  • Checksum verification before deleting local copy     │
  │  • Cross-region replication for DR                       │
  │                                                         │
  │  Alert Durability:                                      │
  │  • Alert state persisted in ruler's WAL                 │
  │  • Alertmanager: nflog (notification log) persisted     │
  │  • Silence state replicated across AM cluster            │
  │  • Alert history: write all state transitions to Kafka  │
  │                                                         │
  │  Gap Detection:                                         │
  │  • Monitor scrape success rate per target                │
  │  • Stale series markers for disappeared targets          │
  │  • Dead man's switch: always-firing alert validates      │
  │    entire pipeline end-to-end                            │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Metric ingestion (remote_write)│ 20 ms  │ 100 ms  │
  │ Instant query (recent data)  │ 10 ms    │ 100 ms  │
  │ Range query (1h, 100 series) │ 50 ms    │ 500 ms  │
  │ Dashboard load (20 panels)   │ 500 ms   │ 3 s     │
  │ Alert evaluation cycle       │ 1 s      │ 5 s     │
  │ Alert → notification         │ 15 s     │ 60 s    │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Recording rules: pre-compute expensive aggregations │
  │  • Exemplar linking: jump from metric to trace          │
  │  • Query result caching (aligned time boundaries)       │
  │  • Downsampled data for ranges > 48h                    │
  │  • Chunk pool reuse: avoid GC pressure                  │
  │  • Alert evaluation parallelism within rule group       │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Scrape target down     │ up{} = 0; alert on absent data; │
  │                        │ staleness handling               │
  ├────────────────────────┼─────────────────────────────────┤
  │ Ingester crash         │ WAL replay; replicas serve data;│
  │                        │ hash ring rebalances             │
  ├────────────────────────┼─────────────────────────────────┤
  │ Alert rule misconfigured│ Dry-run evaluation; unit tests │
  │                        │ for alert rules; review process │
  ├────────────────────────┼─────────────────────────────────┤
  │ Alert storm (cascade)  │ Grouping, inhibition, rate      │
  │                        │ limiting; de-dup in AM cluster  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Query of death         │ Query timeout; max series limit;│
  │                        │ query audit log                  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Cardinality explosion  │ Ingestion-time cardinality cap; │
  │                        │ reject series > threshold        │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 18. Availability

```
  Target: 99.99% for ingestion; 99.99% for alerting (never miss a page)

  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion HA:                                          │
  │  • 2+ Prometheus replicas scraping same targets         │
  │  • Dedup at query time (Thanos / Mimir)                 │
  │  • Agent buffer survives scraper outage (~1 hour)       │
  │                                                         │
  │  Alerting HA:                                           │
  │  • Alertmanager: 3-node gossip cluster                  │
  │  • Dedup: only one instance sends notification          │
  │  • If all AMs down: ruler buffers alerts                │
  │                                                         │
  │  Query HA:                                              │
  │  • Stateless queriers behind load balancer              │
  │  • Store-gateway: replicated for object store access    │
  │  • Partial response mode: return data even if 1 store   │
  │    is unavailable                                       │
  │                                                         │
  │  Dead Man's Switch:                                     │
  │  • Always-firing "watchdog" alert                       │
  │  • If watchdog stops → pipeline is broken               │
  │  • External monitoring checks watchdog presence          │
  └─────────────────────────────────────────────────────────┘
```

---

## 19. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Multi-Tenancy:                                         │
  │  • Tenant ID on every request                           │
  │  • Complete isolation: separate TSDB per tenant          │
  │  • Per-tenant rate limits + cardinality limits           │
  │  • Cost attribution per tenant                           │
  │                                                         │
  │  Authentication:                                        │
  │  • Scrape: bearer token / mTLS to targets               │
  │  • API: OAuth2 / API key via reverse proxy              │
  │  • Grafana: SSO integration (OIDC/SAML)                 │
  │                                                         │
  │  Data Protection:                                       │
  │  • Metric relabeling: drop sensitive label values       │
  │  • TLS for all ingestion and query traffic              │
  │  • At-rest encryption: SSE-S3/KMS for object store     │
  │  • Audit logging: query log for compliance              │
  │                                                         │
  │  Alert Security:                                        │
  │  • Alert content: avoid PII in alert messages           │
  │  • Notification channels: encrypted (TLS webhooks)      │
  │  • Silence/inhibit changes: require approval workflow  │
  └─────────────────────────────────────────────────────────┘
```

---

## 20. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Self-Hosted (Mimir/Thanos, 50M active series):         │
  │  • Ingesters (10 × c5.4xl): ~$10K/month                │
  │  • Queriers (5 × c5.2xl): ~$3K/month                   │
  │  • Object store (100 TB): ~$2.3K/month                 │
  │  • Alert evaluation (3 × m5.xl): ~$500/month           │
  │  • Total: ~$16K/month                                   │
  │                                                         │
  │  Managed (Datadog, 50M custom metrics):                 │
  │  • $0.05/metric/host → easily $100K+/month             │
  │  • 10-50× more expensive than self-hosted               │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Drop unused metrics (metric_relabel_configs)         │
  │  • Reduce scrape interval (30s vs 15s = 50% savings)   │
  │  • Recording rules: pre-aggregate, drop raw            │
  │  • Downsampling: 5m/1h resolution for old data         │
  │  • Cardinality analysis: find expensive label combos   │
  │  • Tiered retention: 7d raw → 90d 5m → 1y 1h           │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Push vs pull trade-offs?** — Pull gives server control of rate, easier target health detection. Push handles ephemeral workloads. Hybrid recommended.
2. **How to handle 10M+ series?** — Horizontal sharding by tenant/metric, Gorilla compression, tiered storage with downsampling
3. **Alert storm prevention?** — Grouping (batch related alerts), inhibition (suppress lower when higher fires), rate limiting notifications
4. **Clock skew handling?** — Accept samples within ±5 min window; reject outliers; NTP sync agents
5. **Anomaly detection without false positives?** — Baseline per time-of-week; use "for" duration to require sustained anomaly; seasonal decomposition
6. **SLO monitoring?** — Error budget = 1 - SLO; burn rate alert fires when error budget consumed too fast (multi-window: 1h fast + 6h slow)
7. **Multi-region alerting?** — Each region evaluates independently; global dedup in Alert Manager cluster; prevent duplicate pages
