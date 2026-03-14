# Design a Time Series Database

Examples: Prometheus, InfluxDB, TimescaleDB, QuestDB

---

## 1. Requirements

### Functional
- Write time-stamped data points: `(metric_name, {labels}, timestamp, value)`
- Query by metric name, label filters, and time range
- Aggregation functions (avg, sum, rate, percentile) over time windows
- Downsampling (rollups: 1s → 1m → 1h → 1d)
- Retention policies per metric/tenant

### Non-Functional
- Extremely high write throughput (millions of data points/sec)
- Low-latency range queries (< 100ms for recent data)
- Efficient compression (10–15× for typical metrics)
- Horizontal scalability
- High availability

---

## 2. High-Level Architecture

```
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │ Exporters  │  │  Agents    │  │  App SDK   │
  │ (node, db) │  │ (telegraf) │  │ (custom)   │
  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
        │               │               │
        └───────────────┬┘───────────────┘
                        │ push or pull
               ┌────────▼────────┐
               │  Ingestion      │
               │  Router         │
               │  (distribute    │
               │   by metric     │
               │   hash)         │
               └────────┬────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │  TSDB      │ │  TSDB      │ │  TSDB      │
   │  Shard 1   │ │  Shard 2   │ │  Shard 3   │
   │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │
   │ │ WAL    │ │ │ │ WAL    │ │ │ │ WAL    │ │
   │ │ Head   │ │ │ │ Head   │ │ │ │ Head   │ │
   │ │ Blocks │ │ │ │ Blocks │ │ │ │ Blocks │ │
   │ └────────┘ │ │ └────────┘ │ │ └────────┘ │
   └────────────┘ └────────────┘ └────────────┘
          │             │             │
          └─────────────┼─────────────┘
                        │
               ┌────────▼────────┐
               │  Query Engine    │
               │  (fan-out to    │
               │   shards,       │
               │   merge results)│
               └────────┬────────┘
                        │
               ┌────────▼────────┐
               │  Dashboard      │
               │  (Grafana)      │
               └─────────────────┘
```

---

## 3. Data Model

```
  A time series is uniquely identified by its metric name + labels:

  cpu_usage{host="web-01", region="us-east", core="0"}

  ┌─────────────────────────────────────────────┐
  │  Series: cpu_usage{host="web-01", core="0"} │
  │                                             │
  │  Timestamp        │ Value                   │
  │  ─────────────────┼──────                   │
  │  1710288000       │ 0.72                    │
  │  1710288015       │ 0.68                    │
  │  1710288030       │ 0.81                    │
  │  1710288045       │ 0.76                    │
  │  ...              │ ...                     │
  └─────────────────────────────────────────────┘

  Cardinality:
  • 1000 hosts × 8 cores × 10 metrics = 80,000 time series
  • High cardinality labels (user_id, request_id) are dangerous
  • 10M+ active series is challenging for a single instance
```

---

## 4. Storage Engine — Write Path

```
  ┌─────────────────────────────────────────────────────┐
  │  Prometheus TSDB Storage Architecture               │
  │                                                     │
  │  1. Write arrives: (metric, labels, ts, value)      │
  │                                                     │
  │  2. Append to WAL (durability)                      │
  │     ┌──────────────────────────┐                    │
  │     │ WAL segment (128 MB)     │                    │
  │     │ [sample1][sample2][...]  │                    │
  │     └──────────────────────────┘                    │
  │                                                     │
  │  3. Write to in-memory "Head" block                 │
  │     ┌──────────────────────────┐                    │
  │     │ Head Block (last 2 hours)│                    │
  │     │ Per-series MemSeries:    │                    │
  │     │   series_1: [ts,val]...  │                    │
  │     │   series_2: [ts,val]...  │                    │
  │     └──────────────────────────┘                    │
  │                                                     │
  │  4. Every 2 hours: cut Head → immutable Block       │
  │     ┌──────────────────────────┐                    │
  │     │ Block (2h chunk on disk) │                    │
  │     │ ├── meta.json            │                    │
  │     │ ├── index (label → series│                    │
  │     │ │    posting lists)      │                    │
  │     │ ├── chunks/              │                    │
  │     │ │    (compressed ts+val) │                    │
  │     │ └── tombstones           │                    │
  │     └──────────────────────────┘                    │
  │                                                     │
  │  5. Compaction: merge multiple blocks               │
  │     [2h][2h] → [4h] → [8h] → [24h]                │
  └─────────────────────────────────────────────────────┘
```

---

## 5. Compression — Gorilla Algorithm

```
  Time-series data is highly compressible because:
  • Timestamps are usually regular intervals (15s, 1m)
  • Values change slowly (CPU: 0.72, 0.73, 0.71, 0.74)

  Delta-of-Delta Timestamp Compression:
  ┌──────────────────────────────────────────────┐
  │  Raw timestamps:  1000, 1015, 1030, 1045     │
  │  Deltas:          -, 15, 15, 15              │
  │  Delta-of-delta:  -, -, 0, 0                  │
  │                                              │
  │  Encode 0 as single bit (1 bit vs 64 bits!)  │
  │  Regular intervals → nearly free to store    │
  └──────────────────────────────────────────────┘

  XOR Float Value Compression:
  ┌──────────────────────────────────────────────┐
  │  Values:          0.72, 0.73, 0.71, 0.74    │
  │  XOR with prev:   -, 0x...small, 0x...small │
  │                                              │
  │  Similar floats XOR to small values          │
  │  → fewer bits to encode the difference       │
  │                                              │
  │  Result: ~1.37 bytes per sample              │
  │  (vs 16 bytes uncompressed = 12× compression)│
  └──────────────────────────────────────────────┘
```

---

## 6. Query Path

```
  Query: rate(http_requests_total{service="api"}[5m])

  ┌──────────────────────────────────────────────────┐
  │  1. Parse query (PromQL / InfluxQL / SQL)        │
  │                                                  │
  │  2. Label index lookup:                          │
  │     __name__="http_requests_total"               │
  │     ∩ service="api"                              │
  │     → matching series IDs: [s1, s5, s12]         │
  │                                                  │
  │  3. Time range filter: [now-5m, now]             │
  │     → select relevant blocks/chunks              │
  │                                                  │
  │  4. Decompress chunks for matching series        │
  │                                                  │
  │  5. Apply function: rate()                       │
  │     rate = (last_value - first_value) / duration │
  │                                                  │
  │  6. Return result per series                     │
  └──────────────────────────────────────────────────┘

  Fan-out query (distributed):
  Query Router → Shard 1, Shard 2, Shard 3
  Each shard returns partial results
  Router merges and applies final aggregation
```

---

## 7. Downsampling

```
  Raw data (15s interval):
  ┌──────────────────────────────────────────────────┐
  │ 10:00:00  72% │ 10:00:15  68% │ 10:00:30  81% │
  │ 10:00:45  76% │ 10:01:00  69% │ 10:01:15  74% │
  │ ...                                             │
  └──────────────────────────────────────────────────┘

  1-minute rollup (avg, min, max, count):
  ┌──────────────────────────────────────────────────┐
  │ 10:00  avg=74.25  min=68  max=81  count=4      │
  │ 10:01  avg=71.50  min=69  max=74  count=2      │
  └──────────────────────────────────────────────────┘

  Retention tiers:
  • Raw (15s):    keep 15 days    → 5,760 points/day
  • 1-min rollup: keep 90 days   → 1,440 points/day
  • 1-hour rollup: keep 1 year   → 24 points/day
  • 1-day rollup:  keep 5 years  → 1 point/day

  Storage savings: ~100× reduction at 1-hour tier
```

---

## 8. Scaling Strategy

```
  ┌─────────────────────────────────────────────────┐
  │  Horizontal Sharding Options:                   │
  │                                                 │
  │  By metric name:                                │
  │    cpu_* → Shard 1                              │
  │    http_* → Shard 2                             │
  │    disk_* → Shard 3                             │
  │                                                 │
  │  By label hash:                                 │
  │    hash(series_labels) % num_shards             │
  │                                                 │
  │  By tenant (multi-tenant):                      │
  │    tenant_A → Shard 1                           │
  │    tenant_B → Shard 2                           │
  │                                                 │
  │  Thanos / Cortex architecture:                  │
  │  • Local Prometheus per cluster (short-term)    │
  │  • Upload blocks to object store (long-term)    │
  │  • Global query layer federates across all      │
  └─────────────────────────────────────────────────┘
```

---

## 9. Fault Tolerance

- **WAL replay:** On restart, replay WAL to recover in-memory head block
- **Replication:** Write to 2+ TSDB instances; query deduplicates
- **Object store backup:** Upload compacted blocks to S3 (Thanos/Cortex pattern)
- **Graceful degradation:** If ingestion overwhelmed, drop lowest-priority metrics

---

---

## 10. Low-Level Design (LLD)

### API Contract
```
  Write (Push):
  POST   /api/v1/write
         Body (Prometheus exposition format):
           http_requests_total{method="GET",status="200"} 1234 1710288000
         Or: remote_write protobuf (compressed, batched)

  Write (Pull/Scrape):
  GET    /metrics  (on target application)
         Response: text/plain Prometheus format

  Query:
  GET    /api/v1/query?query=rate(http_requests_total[5m])&time=1710288000
         Response: { "resultType": "vector", "result": [{ "metric": {...}, "value": [ts,val] }] }

  GET    /api/v1/query_range?query=...&start=T1&end=T2&step=15s
         Response: { "resultType": "matrix", "result": [{ "metric": {...}, "values": [[ts,val],...] }] }

  Metadata:
  GET    /api/v1/series?match[]=up{job="api"}
  GET    /api/v1/labels
  GET    /api/v1/label/{name}/values
```

### TSDB Block Structure
```
  On-Disk Layout:
  /data/
    /01ABCDEF.../           (block: 2h of data)
      meta.json              (time range, stats)
      index                  (label → series → chunks)
      chunks/
        000001               (compressed time-value pairs)
      tombstones             (deletion markers)
    /01ABCDEG.../           (next 2h block)
    wal/
      00000001              (WAL segment, 128 MB)
      00000002

  Block Compaction:
  ┌──────────────────────────────────────────────┐
  │  Head block (in-memory, WAL-backed):         │
  │    Last 2 hours of data                      │
  │    Append-only, mmap WAL for crash recovery  │
  │                                              │
  │  Compaction:                                 │
  │    2h block × 3 → merge into 6h block        │
  │    6h blocks → merge into larger blocks      │
  │    Removes tombstones, re-sorts series       │
  └──────────────────────────────────────────────┘

  Series Index (label inverted index):
  ┌──────────────────────────────────────────────┐
  │  Posting list:                               │
  │  job="api" → [series_1, series_5, series_9]  │
  │  method="GET" → [series_1, series_2, ...]    │
  │                                              │
  │  Query: job="api" AND method="GET"            │
  │  → intersect posting lists → matching series │
  └──────────────────────────────────────────────┘
```

### Gorilla Compression (detailed)
```
  Timestamp compression (delta-of-delta):
  ┌──────────────────────────────────────────────┐
  │  t0 = 1710288000                             │
  │  t1 = 1710288015  → delta = 15              │
  │  t2 = 1710288030  → delta = 15, DoD = 0    │
  │  t3 = 1710288045  → delta = 15, DoD = 0    │
  │  t4 = 1710288061  → delta = 16, DoD = 1    │
  │                                              │
  │  DoD = 0: encode as single '0' bit          │
  │  DoD small: 2-bit header + value            │
  │  Most samples: 1 bit each!                   │
  └──────────────────────────────────────────────┘

  Value compression (XOR):
  ┌──────────────────────────────────────────────┐
  │  v0 = 72.5  (IEEE 754 float64)              │
  │  v1 = 72.5  → XOR = 0 → single '0' bit     │
  │  v2 = 72.8  → XOR = small → encode only     │
  │              changed bits (leading/trailing) │
  │  Result: ~1.37 bytes per sample              │
  │  vs 16 bytes uncompressed (12× compression) │
  └──────────────────────────────────────────────┘
```

---

## 11. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Vertical (single Prometheus):                          │
  │  • Handle ~1M active series per instance                │
  │  • Ingest rate: ~200K samples/sec                       │
  │  • Storage: ~1.5 bytes/sample with compression          │
  │                                                         │
  │  Horizontal (Cortex / Mimir / Thanos):                  │
  │  • Shard by metric hash across ingesters                │
  │  • Each ingester: ~1M series                            │
  │  • 100 ingesters → 100M active series                  │
  │                                                         │
  │  Storage Scaling:                                       │
  │  • Short-term: local SSD (head block + recent blocks)   │
  │  • Long-term: upload compacted blocks to S3              │
  │  • Infinite retention via object store                   │
  │  • Store-gateway: lazy-load blocks from S3 for queries  │
  │                                                         │
  │  Query Scaling:                                         │
  │  • Query-frontend: split large queries into sub-queries │
  │  • Parallel execution across querier instances          │
  │  • Result cache: cache query results for static ranges  │
  │  • Query sharding: split by series                      │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Write Path Durability:                                 │
  │  • WAL: every sample written to WAL before in-memory    │
  │  • WAL segment: fsync at configurable interval          │
  │  • Crash recovery: replay WAL to rebuild head block     │
  │  • WAL truncation: only after block persisted to disk   │
  │                                                         │
  │  Replication (distributed mode):                        │
  │  • Write to 3 ingesters (quorum = 2)                    │
  │  • Each ingester maintains own WAL                      │
  │  • If ingester crashes: WAL replayed by replacement     │
  │                                                         │
  │  Long-Term Storage:                                     │
  │  • Upload compacted blocks to object store              │
  │  • Verify upload with checksum before deleting local    │
  │  • Object store: 11 nines durability (S3)               │
  │  • Cross-region replication for object store             │
  │                                                         │
  │  Gap Detection:                                         │
  │  • Monitor scrape success rate per target               │
  │  • Alert on missing data (absent_over_time())           │
  │  • Staleness markers for disappeared series             │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Latency

```
  Latency Targets:
  ┌──────────────────────────┬──────────┬──────────┐
  │ Operation                │ p50      │ p99      │
  ├──────────────────────────┼──────────┼──────────┤
  │ Scrape (per target)      │ 100 ms   │ 500 ms  │
  │ Remote write batch       │ 50 ms    │ 200 ms  │
  │ Instant query (recent)   │ 10 ms    │ 100 ms  │
  │ Range query (1h, 100 ser)│ 50 ms    │ 500 ms  │
  │ Range query (7d, 1K ser) │ 1 s      │ 5 s     │
  │ Dashboard load (10 panels)│ 500 ms  │ 2 s     │
  └──────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Head block in memory: instant access to recent data │
  │  • Posting list intersection: bitmap for fast filtering│
  │  • Chunk pool: reuse allocated memory for decode       │
  │  • Recording rules: pre-compute expensive aggregations │
  │  • Query result cache: don't recompute static ranges   │
  │  • Downsampled data for long-range queries             │
  │  • Limit series cardinality: reject at ingestion       │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Prometheus crash       │ WAL replay recovers head block; │
  │                        │ ~30s data loss (WAL segment)    │
  ├────────────────────────┼─────────────────────────────────┤
  │ Scrape target down     │ up{} metric = 0; alert on       │
  │                        │ absent data; staleness markers  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Storage full           │ Block retention auto-deletes;   │
  │                        │ alert at 70% capacity           │
  ├────────────────────────┼─────────────────────────────────┤
  │ High cardinality       │ Reject series > limit per metric│
  │ explosion              │ Pre-aggregation rules           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Query of death         │ Query timeout (60s); max series │
  │                        │ per query; circuit breaker      │
  └────────────────────────┴─────────────────────────────────┘

  Meta-Monitoring:
  • Prometheus monitors Prometheus (yes, recursive)
  • Dead man's switch: always-firing alert → check pipeline
  • Watchdog: external blackbox monitoring of alerting path
```

---

## 15. Availability

```
  Target: 99.99% for ingestion (never miss metrics)

  ┌─────────────────────────────────────────────────────────┐
  │  Prometheus HA (Thanos/Mimir Pattern):                  │
  │  • 2+ Prometheus replicas scraping same targets         │
  │  • Thanos Query: deduplicates across replicas           │
  │  • Querier routes to available Prometheus instance      │
  │                                                         │
  │  Ingester HA (Cortex/Mimir):                            │
  │  • 3 ingester replicas per series (hash ring)           │
  │  • Quorum writes: succeed if 2/3 ingesters accept      │
  │  • Zone-aware replication across AZs                    │
  │                                                         │
  │  Query Availability:                                    │
  │  • Multiple querier instances (stateless)               │
  │  • Query-frontend: queueing + retry on failure          │
  │  • Store-gateway: replicated for object store access    │
  │                                                         │
  │  Alerting Availability:                                 │
  │  • Alertmanager cluster (3+ instances, gossip)          │
  │  • Dedup across instances prevents duplicate alerts     │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication:                                        │
  │  • Scrape targets: bearer token or basic auth           │
  │  • Remote write/read: mTLS or OAuth2                    │
  │  • API access: reverse proxy with auth (nginx + OIDC)  │
  │                                                         │
  │  Multi-Tenancy:                                         │
  │  • Cortex/Mimir: tenant ID header on every request     │
  │  • Tenant isolation: separate TSDB per tenant           │
  │  • Rate limiting per tenant (ingestion + query)         │
  │  • Cost attribution per tenant                          │
  │                                                         │
  │  Encryption:                                            │
  │  • TLS for all scrape connections                       │
  │  • TLS for remote write / federate                      │
  │  • At-rest: encrypted object store (SSE-S3/SSE-KMS)    │
  │  • Local blocks: filesystem encryption                   │
  │                                                         │
  │  Data Protection:                                       │
  │  • Metric relabeling: drop sensitive label values       │
  │  • Audit: query logging for compliance                  │
  │  • Retention policies enforced per tenant               │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Storage (primary cost at scale):                       │
  │  • Gorilla compression: ~1.5 bytes/sample               │
  │  • 10M series × 15s interval × 30d retention           │
  │    = 1.7T samples × 1.5B = ~2.5 TB on disk             │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Downsampling: 5m and 1h resolution for old data     │
  │    → 90% storage reduction for data > 48h              │
  │  • Object store for long-term: $0.023/GB vs $0.10/GB   │
  │  • Recording rules: pre-compute heavy queries           │
  │  • Label discipline: avoid high-cardinality labels     │
  │    (each unique label set = new time series)            │
  │  • Drop unused metrics at scrape time (metric_relabel) │
  │                                                         │
  │  Cost Estimate (10M active series, 1-year retention):   │
  │  • Ingesters (3 × c5.4xl): ~$3K/month                  │
  │  • Queriers (3 × c5.2xl): ~$1.5K/month                 │
  │  • Object store (30 TB): ~$700/month                    │
  │  • Total: ~$5K/month                                    │
  │                                                         │
  │  Managed (Datadog/New Relic):                            │
  │  • $0.10 per custom metric per host                     │
  │  • At scale: 10-50× more expensive than self-hosted    │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Why not use a regular database?** — TSDBs are optimised for append-heavy writes, time-range scans, and columnar compression. A regular DB would be 10–100× slower
2. **How to handle high cardinality?** — Cap label cardinality; reject series with > threshold unique label values; pre-aggregate at ingestion
3. **Push vs pull model?** — Pull (Prometheus): simpler, scrape targets on schedule. Push (InfluxDB, Datadog): better for ephemeral workloads, serverless
4. **How does Gorilla compression work?** — Delta-of-delta for timestamps, XOR for values → 12× compression for regular metrics
5. **How to query across long time ranges efficiently?** — Downsampled data for old ranges; only use raw data for recent queries
