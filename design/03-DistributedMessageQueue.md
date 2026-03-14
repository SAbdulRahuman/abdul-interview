# Design a Distributed Message Queue

Examples: Apache Kafka, RabbitMQ, Amazon SQS, Pulsar

---

## 1. Requirements

### Functional
- Producers publish messages to named topics
- Consumers subscribe to topics and read messages
- Support consumer groups (parallel consumption)
- Message ordering within a partition
- Message retention (time-based and size-based)
- Replay / seek to a specific offset

### Non-Functional
- High throughput (millions of messages/sec)
- Low latency (< 10ms end-to-end for real-time)
- Durability (no message loss after acknowledgment)
- Horizontal scalability
- At-least-once or exactly-once delivery guarantees

---

## 2. Scale Estimation

```
Messages per second:      1 million
Average message size:     1 KB
Throughput:               1 GB/s ingestion
Retention:                7 days
Storage:                  1 GB/s × 86400 × 7 = ~600 TB
Replication factor:       3 → ~1.8 PB
Partitions per topic:     100 (for parallelism)
```

---

## 3. High-Level Architecture

```
  Producers                    Broker Cluster                    Consumers
  ┌────────┐              ┌─────────────────────┐              ┌────────────┐
  │ Prod 1 │──┐           │  ┌──────────────┐   │         ┌──▶│ Consumer 1 │
  ├────────┤  │           │  │  Topic: orders│   │         │   ├────────────┤
  │ Prod 2 │──┼──────────▶│  │  Partition 0 │───┼─────────┤   │ Consumer 2 │
  ├────────┤  │           │  │  Partition 1 │───┼─────────┤   ├────────────┤
  │ Prod 3 │──┘           │  │  Partition 2 │───┼─────────┘   │ Consumer 3 │
  └────────┘              │  └──────────────┘   │              └────────────┘
                          │                     │
                          │  ┌──────────────┐   │              Consumer Group
                          │  │  Broker 1    │   │              ┌────────────┐
                          │  │  Broker 2    │   │              │ Each partition│
                          │  │  Broker 3    │   │              │ assigned to  │
                          │  └──────────────┘   │              │ exactly ONE  │
                          │                     │              │ consumer in  │
                          │  ┌──────────────┐   │              │ the group    │
                          │  │  ZooKeeper / │   │              └────────────┘
                          │  │  Raft (meta) │   │
                          │  └──────────────┘   │
                          └─────────────────────┘
```

---

## 4. Core Concepts

### Topic & Partition Model

```
  Topic: "orders"
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Partition 0: [msg0][msg1][msg2][msg3][msg4]  ──▶       │
  │                                    ▲                    │
  │                                  offset=4               │
  │                                                         │
  │  Partition 1: [msg0][msg1][msg2]  ──▶                   │
  │                              ▲                          │
  │                            offset=2                     │
  │                                                         │
  │  Partition 2: [msg0][msg1][msg2][msg3]  ──▶             │
  │                                    ▲                    │
  │                                  offset=3               │
  └─────────────────────────────────────────────────────────┘

  • Messages within a partition are strictly ordered
  • Messages across partitions have NO ordering guarantee
  • Partition key determines which partition a message goes to:
    hash(order_id) mod num_partitions → partition
```

### Append-Only Log

```
  On-Disk Segment Structure (per partition):

  ┌──────────────────────────────────────────────┐
  │  Segment File: 00000000000000000000.log      │
  │  ┌────────┬────────┬────────┬────────┐       │
  │  │ offset │ offset │ offset │ offset │ ...   │
  │  │  0     │  1     │  2     │  3     │       │
  │  │ [key,  │ [key,  │ [key,  │ [key,  │       │
  │  │  val,  │  val,  │  val,  │  val,  │       │
  │  │  ts]   │  ts]   │  ts]   │  ts]   │       │
  │  └────────┴────────┴────────┴────────┘       │
  │                                              │
  │  Index File: 00000000000000000000.index       │
  │  offset → file position (sparse index)        │
  │                                              │
  │  Segments are immutable once full (1 GB)     │
  │  Old segments deleted based on retention      │
  └──────────────────────────────────────────────┘
```

---

## 5. Write Path (Producer)

```
  Producer                    Broker (Partition Leader)
     │                              │
     │──1. Send message ───────────▶│
     │   (key, value, topic)        │
     │                              │──2. Append to log segment
     │                              │
     │                              │──3. Replicate to followers
     │                              │     ┌──────────┐
     │                              │────▶│ Follower 1│──ACK
     │                              │     └──────────┘
     │                              │     ┌──────────┐
     │                              │────▶│ Follower 2│──ACK
     │                              │     └──────────┘
     │                              │
     │◀─4. ACK (when ISR ack'd)────│
     │                              │

  Durability settings:
  • acks=0:  Fire and forget (fastest, may lose data)
  • acks=1:  Leader wrote to its log (fast, leader failure = data loss)
  • acks=all: All in-sync replicas (ISR) acknowledged (safest, slower)
```

---

## 6. Read Path (Consumer)

```
  Consumer                    Broker (Partition Leader)
     │                              │
     │──1. Fetch(topic, partition,  │
     │       offset=42, maxBytes)  ─▶│
     │                              │
     │                              │──2. Read from log at offset 42
     │                              │     (page cache / disk)
     │                              │
     │◀─3. Return batch of msgs ────│
     │   [msg42, msg43, msg44, ...] │
     │                              │
     │──4. Process messages         │
     │                              │
     │──5. Commit offset=45 ───────▶│
     │   (store in __consumer_      │
     │    offsets topic)             │
     │                              │

  Consumer group rebalancing:
  • When a consumer joins/leaves, partitions are redistributed
  • Strategies: Range, RoundRobin, Sticky (minimise movement)
  • Cooperative rebalancing: only reassign affected partitions
```

---

## 7. Replication

```
  Partition 0 — Replica Assignment:
  ┌─────────────┬──────────────┬──────────────┐
  │  Broker 1   │  Broker 2    │  Broker 3    │
  │  (Leader)   │  (Follower)  │  (Follower)  │
  │  [0,1,2,3]  │  [0,1,2,3]  │  [0,1,2,3]  │
  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
         │                │                │
         │   ISR (In-Sync Replicas)        │
         │◀──────────────▶│◀──────────────▶│
         │                │                │

  ISR (In-Sync Replica Set):
  • Followers that are caught up with the leader
  • If a follower falls behind → removed from ISR
  • Writes only acknowledged when all ISR replicas have the data
  • Leader failure → new leader elected from ISR

  min.insync.replicas = 2:
  • At least 2 replicas (including leader) must ACK
  • If ISR drops below 2, producer gets error (prevents data loss)
```

---

## 8. Delivery Guarantees

```
  ┌─────────────────────────────────────────────────────────┐
  │  At-Most-Once:                                          │
  │  • Consumer commits offset BEFORE processing            │
  │  • If consumer crashes → message skipped                │
  │  • Use case: metrics, logs (some loss acceptable)       │
  ├─────────────────────────────────────────────────────────┤
  │  At-Least-Once:                                         │
  │  • Consumer commits offset AFTER processing             │
  │  • If consumer crashes → message re-delivered           │
  │  • Consumer must be idempotent                          │
  │  • Most common in production                            │
  ├─────────────────────────────────────────────────────────┤
  │  Exactly-Once:                                          │
  │  • Idempotent producer (dedup by sequence number)       │
  │  • Transactional writes (atomic multi-partition writes) │
  │  • Consumer: read-process-write in same transaction     │
  │  • Kafka achieves this with transactional API           │
  └─────────────────────────────────────────────────────────┘
```

---

## 9. Scaling Strategy

- **More partitions:** Increases parallelism (more consumers can read in parallel)
- **More brokers:** Distributes storage and network load
- **Partition reassignment:** Move partitions between brokers during scaling
- **Tiered storage:** Keep recent data on local SSD, older data on object store (S3)
- **Compression:** LZ4, Snappy, zstd to reduce storage and network I/O
- **Batching:** Producers batch messages for fewer network round trips

---

## 10. Fault Tolerance

```
  Failure Scenario          │ What Happens
  ──────────────────────────┼───────────────────────────────────
  Broker crash (non-leader) │ No impact; consumer reads from leader
  Leader broker crash       │ New leader elected from ISR (< 1s)
  Consumer crash            │ Consumer group rebalances partitions
  Producer crash            │ In-flight msgs lost (unless acks=all + retries)
  Disk failure              │ Replicas on other brokers serve data
  Network partition         │ ISR shrinks; writes may be rejected if
                            │ min.insync.replicas not met
```

---

---

## 11. Low-Level Design (LLD)

### API Contract
```
  Producer API:
  POST   /api/v1/topics/{topic}/messages
         Body: { "key": bytes, "value": bytes, "headers": {}, "partition": int|null }
         Response: { "topic": str, "partition": int, "offset": long, "timestamp": long }

  Consumer API:
  POST   /api/v1/topics/{topic}/subscribe
         Body: { "group_id": str, "auto_offset_reset": "earliest|latest" }

  GET    /api/v1/topics/{topic}/poll?max_records=500&timeout_ms=1000
         Response: { "records": [{ "offset": long, "key": bytes, "value": bytes, "ts": long }] }

  POST   /api/v1/topics/{topic}/commit
         Body: { "offsets": { "0": 42, "1": 18 } }

  Admin API:
  POST   /api/v1/topics   (create topic: name, partitions, replication factor)
  PUT    /api/v1/topics/{topic}/partitions  (increase partition count)
```

### On-Disk Storage Format
```
  Per Partition Directory: /data/topic-orders/partition-0/
  ┌──────────────────────────────────────────────┐
  │  00000000000000000000.log     (segment file)  │
  │  00000000000000000000.index   (offset→pos)    │
  │  00000000000000000000.timeindex (ts→offset)   │
  │  00000000000001048576.log     (next segment)  │
  │                                              │
  │  Message format on disk (per record):        │
  │  ┌────────┬───────┬──────┬──────┬──────────┐ │
  │  │CRC (4B)│Len(4B)│Key   │Value │Headers   │ │
  │  │        │       │(var) │(var) │(var)     │ │
  │  └────────┴───────┴──────┴──────┴──────────┘ │
  │                                              │
  │  Record batch: group multiple messages        │
  │  Batch header: base offset, count, codec     │
  │  Compression applied per batch (not per msg)  │
  └──────────────────────────────────────────────┘

  Zero-Copy Transfer:
  ┌─────────────────────────────────────────────────┐
  │  sendfile() syscall:                            │
  │  Disk → Kernel page cache → NIC (skip userspace)│
  │  Reduces CPU usage by 50%+ for consumer reads   │
  └─────────────────────────────────────────────────┘
```

### Consumer Group Coordinator
```
  Rebalance Protocol:
  1. Consumer sends JoinGroup → coordinator
  2. Coordinator selects one consumer as "leader"
  3. Leader computes partition assignments
  4. Leader sends SyncGroup with assignments
  5. Coordinator distributes assignments to all consumers
  6. Each consumer fetches from assigned partitions

  Assignment Strategies:
  • Range: partition ranges per consumer (contiguous)
  • RoundRobin: cycle through partitions & consumers
  • Sticky: minimize partition movement during rebalance
  • Cooperative Sticky: incremental rebalance (no stop-the-world)
```

---

## 12. Scalability

```
  ┌────────────────────────────────────────────────────────┐
  │  Throughput Scaling:                                   │
  │  • More partitions → more consumer parallelism        │
  │  • 1 partition ≈ 10-50 MB/s write throughput          │
  │  • 100 partitions × 10 MB/s = 1 GB/s per topic       │
  │                                                        │
  │  Broker Scaling:                                       │
  │  • Add brokers → partition leaders redistributed       │
  │  • Each broker: ~100K msg/s (depends on msg size)     │
  │  • 30 brokers → 3M msg/s cluster throughput           │
  │                                                        │
  │  Storage Scaling:                                      │
  │  • Tiered storage: hot segments on local SSD           │
  │    cold segments in S3 (Kafka 3.6+ KIP-405)           │
  │  • Infinite retention without local disk limits        │
  │  • Segment size: 1 GB default, configurable           │
  │                                                        │
  │  Consumer Scaling:                                     │
  │  • Max consumers in a group = number of partitions    │
  │  • Scale consumers to match partition count            │
  │  • Multiple consumer groups can read independently     │
  └────────────────────────────────────────────────────────┘
```

---

## 13. No Data Loss

```
  End-to-End Durability Guarantees:
  ┌─────────────────────────────────────────────────────────┐
  │  Producer Side:                                         │
  │  • acks=all → leader + all ISR replicas acknowledge    │
  │  • enable.idempotence=true → dedup retries (seq nums)  │
  │  • max.in.flight.requests=5 (with idempotent producer) │
  │  • retries=MAX_INT with delivery.timeout.ms            │
  │                                                         │
  │  Broker Side:                                           │
  │  • min.insync.replicas=2 → require 2 replicas for ACK │
  │  • unclean.leader.election=false → never elect out-of- │
  │    sync replica as leader (prevents data loss)          │
  │  • log.flush.interval.messages=1 (optional: sync every │
  │    message — usually OS page cache is sufficient)       │
  │                                                         │
  │  Consumer Side:                                         │
  │  • enable.auto.commit=false → manual offset commit     │
  │  • Process message → then commit offset                │
  │  • Idempotent processing for at-least-once semantics   │
  │                                                         │
  │  Exactly-Once (Transactions):                           │
  │  • Producer: beginTransaction → send → commitTransaction│
  │  • Consumer: read_committed isolation level             │
  │  • Offset commit within transaction boundary            │
  └─────────────────────────────────────────────────────────┘

  Disaster Recovery:
  • Cross-datacenter replication (MirrorMaker 2)
  • Async replication: RPO = replication lag (~seconds)
  • Geo-active: two clusters with bidirectional mirroring
```

---

## 14. Latency

```
  Latency Targets:
  ┌──────────────────┬──────────┬──────────┐
  │ Operation        │ p50      │ p99      │
  ├──────────────────┼──────────┼──────────┤
  │ Produce (acks=1) │ 2 ms     │ 10 ms   │
  │ Produce (acks=all)│ 5 ms    │ 25 ms   │
  │ Consume (fetch)  │ 1 ms     │ 5 ms    │
  │ End-to-end       │ 10 ms    │ 50 ms   │
  └──────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  Producer:                                              │
  │  • Batching: linger.ms=5 → batch msgs for 5ms          │
  │  • Compression: LZ4 (fast) or Zstd (better ratio)      │
  │  • Partition-level sticky partitioning for better batch │
  │                                                         │
  │  Broker:                                                │
  │  • OS page cache for reads (kernel manages caching)     │
  │  • Zero-copy: sendfile() for consumer fetches           │
  │  • Sequential I/O: append-only writes → disk-friendly   │
  │  • SSD for log segments reduces seek latency            │
  │                                                         │
  │  Consumer:                                              │
  │  • Fetch.min.bytes & fetch.max.wait.ms tuning           │
  │  • Pre-fetch: consumer buffers messages ahead            │
  │  • Parallel processing within partition (if ordered)     │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Reliability

```
  Failure Modes & Recovery:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Recovery                        │
  ├────────────────────────┼─────────────────────────────────┤
  │ Broker crash (leader)  │ ISR elects new leader (<1s)     │
  │                        │ Producers retry to new leader   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Broker crash (follower)│ Removed from ISR; data safe     │
  │                        │ on leader; follower catches     │
  │                        │ up on restart                   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Consumer crash         │ Rebalance assigns partitions to │
  │                        │ remaining consumers (~10-30s)   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Network partition      │ ISR shrinks; writes rejected if │
  │                        │ < min.insync.replicas           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Disk failure           │ Replicas serve data; replace    │
  │                        │ disk and re-replicate           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Poison message         │ Dead letter topic; skip & log;  │
  │                        │ manual investigation            │
  └────────────────────────┴─────────────────────────────────┘

  Health Monitoring:
  • Under-replicated partitions (ISR < replication factor)
  • Consumer lag per partition (offset diff)
  • Broker request handler idle ratio
```

---

## 16. Availability

```
  Target: 99.99% (52.6 min downtime/year)

  Cluster Topology:
  ┌────────────────────────────────────────────────────────┐
  │  3+ brokers across 3 AZs (rack-aware)                 │
  │  Replication factor = 3, min.insync.replicas = 2       │
  │                                                        │
  │  AZ-1        AZ-2        AZ-3                          │
  │  ┌────────┐ ┌────────┐ ┌────────┐                     │
  │  │Broker 1│ │Broker 2│ │Broker 3│                     │
  │  │P0(L)   │ │P0(F)   │ │P0(F)   │                     │
  │  │P1(F)   │ │P1(L)   │ │P1(F)   │                     │
  │  │P2(F)   │ │P2(F)   │ │P2(L)   │                     │
  │  └────────┘ └────────┘ └────────┘                     │
  │                                                        │
  │  Any single AZ outage → leaders fail over to other AZs│
  │  Writes continue if ISR ≥ min.insync.replicas          │
  │  Zero downtime rolling upgrades (one broker at a time) │
  └────────────────────────────────────────────────────────┘

  Controller Availability:
  • KRaft mode: 3 controller nodes with Raft consensus
  • Controller quorum tolerates 1 controller failure
  • Metadata stored in internal Raft log (no ZooKeeper)
```

---

## 17. Security

```
  ┌────────────────────────────────────────────────────────┐
  │  Authentication:                                       │
  │  • SASL/SCRAM: username/password per client            │
  │  • mTLS: certificate-based client authentication       │
  │  • OAuth 2.0 bearer tokens (Kafka 3.0+)               │
  │                                                        │
  │  Authorization (ACLs):                                 │
  │  • Per-topic: producer/consumer/admin permissions      │
  │  • Per-consumer-group: who can join which group        │
  │  • Per-cluster: who can create/delete topics           │
  │  • Example: user=payments → write to "orders" topic    │
  │                                                        │
  │  Encryption:                                           │
  │  • TLS 1.3 for client ↔ broker and broker ↔ broker     │
  │  • At-rest encryption via filesystem encryption        │
  │    (dm-crypt/LUKS) or cloud-managed encryption         │
  │                                                        │
  │  Data Governance:                                      │
  │  • Schema Registry: enforce message schema (Avro/Proto)│
  │  • PII detection and field-level encryption            │
  │  • Audit logging for topic access and config changes   │
  │  • Retention policies for compliance (GDPR: right to   │
  │    delete → use compacted topics with null tombstones) │
  └────────────────────────────────────────────────────────┘
```

---

## 18. Cost Constraints

```
  ┌────────────────────────────────────────────────────────┐
  │  Storage (largest cost driver):                        │
  │  • Compression: LZ4 (2-3×), Zstd (3-5× ratio)        │
  │  • Right-size retention: 7d instead of infinite        │
  │  • Tiered storage: local SSD (hot) + S3 (cold)        │
  │    → 10× cost reduction for cold data                 │
  │  • Compact topics: keep only latest per key            │
  │                                                        │
  │  Compute:                                              │
  │  • Brokers are I/O-bound, not CPU-bound                │
  │  • Use storage-optimized instances (d3/i3en)           │
  │  • Right-size: 1 broker per 100-200 partitions         │
  │                                                        │
  │  Network:                                              │
  │  • Cross-AZ replication: ~$0.01/GB both directions    │
  │  • Minimize replication factor for non-critical topics  │
  │  • Consumer reads from closest replica (KIP-392)       │
  │                                                        │
  │  Cost Estimate (1 GB/s ingest, 7d retention):          │
  │  • Storage: 600 TB × $0.10/GB = ~$60K/month           │
  │  • Brokers: 30 × i3en.xlarge = ~$20K/month            │
  │  • Network (cross-AZ): ~$10K/month                    │
  │  • Total: ~$90K/month                                  │
  │  • With tiered storage: ~$35K/month (60% savings)     │
  └────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Kafka vs RabbitMQ?** — Kafka: log-based, high throughput, replay; RabbitMQ: traditional queue, push-based, per-message routing, lower throughput
2. **How do you guarantee ordering?** — All related messages use the same partition key (e.g., user_id)
3. **How do you handle slow consumers?** — Consumer lag monitoring, auto-scaling consumer group, dead letter queue for poison messages
4. **How do you handle message replay?** — Consumer can seek to any offset (messages retained for retention period)
5. **How do you achieve exactly-once end-to-end?** — Idempotent producers + transactional API + consumer offset commit in same transaction
