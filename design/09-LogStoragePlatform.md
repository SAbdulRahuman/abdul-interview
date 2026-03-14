# Design a Log Storage Platform

Examples: Elasticsearch, ELK Stack, Splunk, Loki

---

## 1. Requirements

### Functional
- Ingest structured and unstructured log data from thousands of sources
- Full-text search across logs
- Filter by timestamp, severity, service, host
- Aggregate and visualise log data (dashboards)
- Retention policies (hot/warm/cold tiers)

### Non-Functional
- High ingestion rate (100K+ events/sec)
- Sub-second search latency for recent logs
- Petabyte-scale storage
- High availability and durability
- Multi-tenancy support

---

## 2. High-Level Architecture

```
  ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ App Srv 1 │ │ App Srv 2 │ │ App Srv 3 │
  │ (log agent│ │ (log agent│ │ (log agent│
  │  Filebeat)│ │  Fluentd) │ │  Vector)  │
  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
              ┌────────▼────────┐
              │  Message Queue   │
              │  (Kafka)         │
              │  Buffer + decouple│
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Log Processor   │
              │  (Logstash /     │
              │   Flink)         │
              │  • Parse         │
              │  • Enrich        │
              │  • Transform     │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Index & Store   │
              │  (Elasticsearch) │
              │  ┌────────────┐  │
              │  │  Shards    │  │
              │  │  (inverted │  │
              │  │   index)   │  │
              │  └────────────┘  │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Query / UI      │
              │  (Kibana /       │
              │   Grafana)       │
              └─────────────────┘
```

---

## 3. Ingestion Pipeline

```
  Log Event Lifecycle:

  Source                    Agent              Kafka              Processor
    │                        │                   │                    │
    │──log line──────────▶  │                   │                    │
    │ "2026-03-12 ERROR     │                   │                    │
    │  NullPointer at..."   │                   │                    │
    │                        │──batch + compress─▶│                    │
    │                        │  (every 5s or      │                    │
    │                        │   1000 lines)      │                    │
    │                        │                   │──consume batch────▶│
    │                        │                   │                    │
    │                        │                   │                    │──parse fields
    │                        │                   │                    │  timestamp, level
    │                        │                   │                    │  service, message
    │                        │                   │                    │
    │                        │                   │                    │──enrich
    │                        │                   │                    │  add geo, hostname
    │                        │                   │                    │  trace_id
    │                        │                   │                    │
    │                        │                   │                    │──write to index
    │                        │                   │                    │  (Elasticsearch)

  Backpressure handling:
  • Kafka absorbs spikes — processors consume at their own pace
  • If processors are overloaded → increase Kafka consumer group size
  • Dead letter queue for unparseable events
```

---

## 4. Inverted Index (Search Engine Core)

```
  Log entry: "Connection timeout to database server db-01"

  Inverted Index:
  ┌────────────────┬────────────────────────┐
  │ Term           │ Document IDs           │
  ├────────────────┼────────────────────────┤
  │ connection     │ [doc1, doc5, doc42]    │
  │ timeout        │ [doc1, doc7, doc99]    │
  │ database       │ [doc1, doc3, doc15]    │
  │ server         │ [doc1, doc8, doc33]    │
  │ db-01          │ [doc1, doc12]          │
  └────────────────┴────────────────────────┘

  Search "connection AND timeout":
  → Intersect posting lists: [doc1, doc5, doc42] ∩ [doc1, doc7, doc99]
  → Result: [doc1]

  Additional indexes:
  • Time-based index: partition by day/hour for fast time-range queries
  • Field-specific: separate index for structured fields (service, level)
```

---

## 5. Sharding and Replication

```
  Index: "app-logs-2026.03.12"

  ┌─────────────────────────────────────────────┐
  │  5 Primary Shards + 1 Replica each          │
  │                                             │
  │  Node 1: [P0] [R2]                         │
  │  Node 2: [P1] [R3]                         │
  │  Node 3: [P2] [R4]                         │
  │  Node 4: [P3] [R0]                         │
  │  Node 5: [P4] [R1]                         │
  │                                             │
  │  Write: goes to primary shard               │
  │  Read: can go to primary OR replica          │
  │                                             │
  │  Shard routing: hash(doc_id) % num_shards  │
  └─────────────────────────────────────────────┘

  Time-based indices:
  • New index per day: app-logs-2026.03.12, app-logs-2026.03.13
  • Hot index (today): more shards, SSD, more replicas
  • Old indices: merge shards, move to HDD, reduce replicas
  • Delete: drop entire index (fast, no per-doc deletion needed)
```

---

## 6. Storage Tiering

```
  ┌───────────────────────────────────────────┐
  │  Hot Tier (0–3 days)                      │
  │  • SSD storage                            │
  │  • Full replicas                          │
  │  • All fields indexed                      │
  │  • Sub-second query latency               │
  ├───────────────────────────────────────────┤
  │  Warm Tier (3–30 days)                    │
  │  • HDD storage                            │
  │  • Read-only (force merge shards)         │
  │  • Reduced replicas                        │
  │  • Slower queries OK (< 5s)               │
  ├───────────────────────────────────────────┤
  │  Cold Tier (30–365 days)                  │
  │  • Object store (S3)                      │
  │  • Frozen index (searchable snapshots)    │
  │  • No replicas (rely on object store)     │
  │  • Query on demand (seconds to minutes)   │
  ├───────────────────────────────────────────┤
  │  Delete (> 365 days)                      │
  │  • Drop index entirely                    │
  └───────────────────────────────────────────┘

  ILM (Index Lifecycle Management) automates transitions
```

---

## 7. Query Optimization

```
  Typical log query: "Show all ERROR logs from payment-service in last 1 hour"

  Query plan:
  1. Time filter → narrow to indices for last 1 hour
  2. Field filter → service:payment-service (structured field)
  3. Level filter → level:ERROR
  4. Full-text    → optional keyword search within message

  Optimisations:
  ┌──────────────────────────────────────────────┐
  │  • Time-based index pruning: skip old indices │
  │  • Field-level index: avoid full-text scan    │
  │  • Caching: query result cache, field cache   │
  │  • Scroll API: paginate large result sets     │
  │  • Aggregation push-down: compute on shards   │
  │  • Query routing: route to relevant shards    │
  └──────────────────────────────────────────────┘
```

---

## 8. Fault Tolerance

- **Node failure:** Replicas serve reads; primary re-elected from replicas
- **Cluster split:** Minimum master nodes (quorum) prevents split-brain
- **Ingestion spike:** Kafka buffers; back-pressure propagated to agents
- **Index corruption:** Replicas provide redundancy; snapshots for disaster recovery
- **Data loss prevention:** Translog (WAL) per shard ensures durability before flush

---

---

## 9. Low-Level Design (LLD)

### API Contract
```
  Ingestion:
  POST   /api/v1/logs/ingest
         Body: [{ "timestamp": iso8601, "level": "INFO|WARN|ERROR",
                  "service": str, "message": str, "metadata": {} }]
         Response: { "accepted": int, "rejected": int }

  POST   /api/v1/logs/bulk  (NDJSON format for high throughput)
         Body: newline-delimited JSON log entries

  Query:
  GET    /api/v1/logs/search?q=error+timeout&service=api-gw&from=2026-03-11&to=2026-03-12
         Response: { "hits": int, "results": [{ log entries }],
                     "aggregations": { "level_counts": {} } }

  POST   /api/v1/logs/query  (structured query)
         Body: { "bool": { "must": [{ "match": { "message": "timeout" } }],
                            "filter": [{ "range": { "@timestamp": { "gte": "..." } } }] } }

  Alert Rules:
  POST   /api/v1/alerts/rules
         Body: { "name": str, "query": str, "threshold": int,
                 "window_minutes": 5, "channels": ["slack", "pagerduty"] }
```

### Index Structure (Elasticsearch-like)
```
  Per-Index Shard:
  ┌──────────────────────────────────────────────┐
  │  Inverted Index:                             │
  │  "timeout" → [doc1:pos3, doc5:pos1, doc9:pos7]│
  │  "error"   → [doc1:pos1, doc2:pos1, doc5:pos4]│
  │                                              │
  │  Stored Fields (columnar):                   │
  │  @timestamp: [doc1→ts1, doc2→ts2, ...]       │
  │  level:      [doc1→ERROR, doc2→INFO, ...]    │
  │  service:    [doc1→api-gw, doc2→auth, ...]   │
  │                                              │
  │  Doc Values (sorted, for aggregations):      │
  │  level → {ERROR: [doc1,doc5], INFO: [doc2,...]}│
  │                                              │
  │  Segment (immutable after merge):            │
  │  .tip (term index) .tim (term dict)          │
  │  .doc (postings) .fdx/.fdt (stored fields)   │
  └──────────────────────────────────────────────┘
```

### Pipeline Processing
```
  Log Entry → Parser → Enricher → Router → Writer

  Parser:
  • Structured: JSON → extract fields
  • Unstructured: grok patterns (regex)
  • Multi-line: stack trace assembly

  Enricher:
  • GeoIP lookup for IP addresses
  • Service metadata injection (team, environment)
  • Timestamp normalization to UTC

  Router:
  • By log level: ERROR → high-priority index
  • By retention: audit logs → long-retention index
  • By volume: high-volume service → dedicated index
```

---

## 10. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion Scaling:                                     │
  │  • Kafka partitions: 1 partition ≈ 50K logs/sec         │
  │  • 100 partitions → 5M logs/sec ingestion capacity     │
  │  • Horizontal: add more Kafka brokers + consumers       │
  │                                                         │
  │  Index Scaling:                                         │
  │  • Time-based indices: logs-2026-03-12 (1 per day)      │
  │  • Each index: 3-10 primary shards × 1 replica         │
  │  • Hot nodes: 10 shards (today's data, SSDs)            │
  │  • Warm nodes: yesterday-7d (HDDs, fewer replicas)      │
  │  • Frozen: searchable snapshots from S3                 │
  │                                                         │
  │  Query Scaling:                                         │
  │  • Coordinating nodes: scatter-gather across shards    │
  │  • Circuit breaker: reject queries > memory threshold  │
  │  • Query cache: cache frequent aggregation results     │
  │  • Time-bounded queries: always require time range     │
  │                                                         │
  │  Cluster Sizing (1 TB/day ingestion):                   │
  │  • Hot: 6 nodes (8 vCPU, 64 GB RAM, 2 TB SSD each)    │
  │  • Warm: 4 nodes (4 vCPU, 32 GB RAM, 10 TB HDD each)  │
  │  • Cold: S3 (searchable snapshots, pay per query)      │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  End-to-End At-Least-Once:                              │
  │  1. Agent → Kafka: acks=all (no msg lost in transit)   │
  │  2. Kafka retention: 72h (buffer for consumer outage)  │
  │  3. Consumer → ES: translog (WAL) before ACK to Kafka  │
  │  4. ES → disk: translog fsynced every 5s               │
  │  5. Replica: each shard has 1 replica on different node│
  │                                                         │
  │  Agent-Side Durability:                                 │
  │  • Filebeat: registry file tracks file offset           │
  │  • On agent restart: resume from last committed offset  │
  │  • Disk buffer: queue logs locally if Kafka unreachable │
  │                                                         │
  │  Exactly-Once Dedup (if needed):                        │
  │  • Each log entry has unique ID (hash of content + ts)  │
  │  • ES: dedup by document ID on index                    │
  │  • Kafka idempotent producer prevents retransmit dupes  │
  │                                                         │
  │  Disaster Recovery:                                     │
  │  • Cross-cluster replication (CCR) to DR region         │
  │  • S3 snapshots of indices (daily)                      │
  │  • RPO: < 1 hour (CCR lag)                              │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Latency

```
  Latency Targets:
  ┌──────────────────────┬──────────┬──────────┐
  │ Operation            │ p50      │ p99      │
  ├──────────────────────┼──────────┼──────────┤
  │ Ingest → searchable  │ 1 s      │ 5 s     │
  │ Simple search (1 day)│ 100 ms   │ 500 ms  │
  │ Complex agg (7 days) │ 1 s      │ 5 s     │
  │ Full-text search     │ 200 ms   │ 1 s     │
  │ Tail -f (live tail)  │ 2 s      │ 5 s     │
  └──────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion:                                             │
  │  • Bulk API: batch 500-5000 docs per request           │
  │  • Reduce refresh interval: 30s (vs default 1s) for    │
  │    write-heavy indices                                  │
  │  • Index templates: optimize mappings (keyword vs text) │
  │                                                         │
  │  Query:                                                 │
  │  • Time range filter first (prune shards)               │
  │  • Request cache for repeated aggregations              │
  │  • Shard request cache for time-based indices           │
  │  • Pre-warm: force-merge old indices to 1 segment      │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ ES node crash          │ Replica shards auto-promoted;   │
  │                        │ re-replication to new node      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Kafka broker crash     │ ISR failover; consumers         │
  │                        │ reconnect to new leader         │
  ├────────────────────────┼─────────────────────────────────┤
  │ Agent crash            │ Filebeat resumes from registry; │
  │                        │ at-most few seconds of logs lost│
  ├────────────────────────┼─────────────────────────────────┤
  │ Index corruption       │ Restore from replica; snapshot  │
  │                        │ restore if replica also corrupt │
  ├────────────────────────┼─────────────────────────────────┤
  │ Ingestion spike (10×)  │ Kafka absorbs burst; consumers  │
  │                        │ catch up; circuit breaker on ES │
  └────────────────────────┴─────────────────────────────────┘

  Backpressure:
  • Kafka consumer lag > threshold → alert
  • ES bulk queue full → consumer slows down (bounded queue)
  • Agent disk buffer when downstream is slow
```

---

## 14. Availability

```
  Target: 99.9% for search, 99.99% for ingestion (never lose logs)

  ┌─────────────────────────────────────────────────────────┐
  │  Ingestion Path:                                        │
  │  • Agents: local disk buffer if pipeline down           │
  │  • Kafka: 3 brokers, 3 AZs, replication=3              │
  │  • Even if ES is down → Kafka buffers for 72h          │
  │                                                         │
  │  Search Path:                                           │
  │  • ES cluster: 3+ data nodes, each shard replicated    │
  │  • Master nodes: 3 dedicated (quorum-based election)   │
  │  • Coordinating nodes: stateless, auto-scalable         │
  │                                                         │
  │  Multi-Region:                                          │
  │  • CCR (Cross-Cluster Replication) for DR               │
  │  • Active-passive: query local cluster primarily       │
  │  • Failover: switch DNS to DR region                    │
  │                                                         │
  │  Maintenance:                                           │
  │  • Rolling restarts without search downtime             │
  │  • ILM (Index Lifecycle Management) manages rollover   │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Access Control:                                        │
  │  • RBAC: read-only for devs, write for agents, admin   │
  │  • Index-level permissions (team sees only their logs) │
  │  • Field-level security: mask PII fields               │
  │  • Document-level security: filter by tenant            │
  │                                                         │
  │  Encryption:                                            │
  │  • In-transit: TLS for agent→Kafka→ES and ES→client    │
  │  • At-rest: encrypted indices (AES-256)                 │
  │  • Key management: KMS integration                      │
  │                                                         │
  │  PII Protection:                                        │
  │  • Ingest pipeline: redact credit cards, SSNs, emails  │
  │  • Field masking: show "*****" for sensitive fields     │
  │  • GDPR: delete right → delete-by-query + reindex      │
  │                                                         │
  │  Audit:                                                  │
  │  • Log all search queries (who searched what)           │
  │  • Compliance audit trail for log access                │
  │  • Retention policies aligned with compliance needs     │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Storage is the biggest cost (logs are voluminous):     │
  │                                                         │
  │  ILM Tiering (critical for cost):                       │
  │  • Hot (SSD, 0-2 days):  $0.10/GB/month                │
  │  • Warm (HDD, 2-30 days): $0.03/GB/month               │
  │  • Cold (S3, 30-365 days): $0.004/GB/month             │
  │  • Delete after 365 days                                │
  │                                                         │
  │  Reduction Strategies:                                  │
  │  • Structured logging: smaller messages (no freeform)   │
  │  • Sample verbose logs: keep 10% of DEBUG level         │
  │  • Force-merge old indices: 1 segment = better compress│
  │  • Compression: best_compression codec (zstd)           │
  │  • Drop unneeded fields before indexing                 │
  │                                                         │
  │  Cost Estimate (1 TB/day ingestion, 90-day retention):  │
  │  • Hot nodes: 6 × i3.2xl = ~$6K/month                  │
  │  • Warm nodes: 4 × d3.2xl = ~$3K/month                 │
  │  • Cold (S3): 60 TB = ~$240/month                       │
  │  • Kafka: 3 brokers = ~$2K/month                        │
  │  • Total: ~$11K/month                                   │
  │  • Without tiering: ~$25K/month (55% more expensive)   │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Why Kafka before Elasticsearch?** — Decouples producers from consumers; absorbs spikes; enables replay if indexing fails
2. **How to handle high cardinality fields?** — Avoid indexing `trace_id` as keyword if billions of unique values; use doc_values
3. **How to reduce cost at scale?** — Time-based indices + ILM tiering + force-merge old indices + searchable snapshots
4. **ELK vs Loki?** — ELK: full inverted index, rich queries, high cost. Loki: only indexes labels (not log content), cheaper, relies on grep-like search
5. **How to ensure logs are not lost?** — At-least-once pipeline: agent → Kafka (acks=all) → processor → ES (translog)
