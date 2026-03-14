# Design a Distributed Key-Value Store

Examples: Redis, DynamoDB, etcd

---

## 1. Requirements

### Functional
- `put(key, value)` — store a key-value pair
- `get(key)` — retrieve value by key
- `delete(key)` — remove a key
- Support TTL (time-to-live) per key
- Support versioning / conflict resolution

### Non-Functional
- High availability (99.99%)
- Low latency (< 10ms p99)
- Horizontal scalability (petabytes of data)
- Tunable consistency (strong or eventual)
- Partition tolerance

---

## 2. Scale Estimation

```
Total keys:              1 billion
Average key size:        50 bytes
Average value size:      1 KB
Total storage:           1B × 1 KB = ~1 TB
Replication factor:      3 → 3 TB
Read QPS:                500K
Write QPS:               100K
```

---

## 3. High-Level Architecture

```
                         ┌─────────────────┐
                         │   Client SDK     │
                         │  (hash key →     │
                         │   pick node)     │
                         └────────┬────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │  Node A   │ │  Node B   │ │  Node C   │
              │ (Shard 1) │ │ (Shard 2) │ │ (Shard 3) │
              └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
                    │             │             │
              ┌─────▼─────┐ ┌────▼──────┐ ┌────▼──────┐
              │ Replica A' │ │ Replica B'│ │ Replica C'│
              │ Replica A''│ │ Replica B''│ │ Replica C''│
              └───────────┘ └───────────┘ └───────────┘
```

**How it works:**
1. Client hashes the key to determine which shard (partition) owns it
2. The request is routed to the primary node for that shard
3. The primary replicates to follower replicas before (or after) acknowledging
4. Reads can go to primary (strong) or any replica (eventual)

---

## 4. Consistent Hashing

```
               Token Ring (0 to 2^128)

                    0
                    │
            N4 ─────┼───── N1
                    │
                    │
            N3 ─────┼───── N2
                    │
                  2^64

   Key "user:123" → hash → 0x3FA... → lands between N1 and N2
   → Assigned to N2

   Virtual Nodes: each physical node owns 100-200 tokens
   → Smoother distribution, easier rebalancing
```

**Why consistent hashing?**
- Adding/removing a node only moves ~1/N of the keys
- Virtual nodes prevent hotspots from uneven hash distribution
- Rack-aware / zone-aware placement ensures replicas are in different failure domains

---

## 5. Data Model & Storage Engine

```
  Write Path:
  ┌────────┐    ┌─────────┐    ┌───────────┐    ┌──────────┐
  │ Client │───▶│  WAL    │───▶│ MemTable  │───▶│ SSTable  │
  │        │    │ (append)│    │ (sorted)  │    │ (on disk)│
  └────────┘    └─────────┘    └───────────┘    └──────────┘

  Read Path:
  ┌────────┐    ┌───────────┐    ┌──────────────┐    ┌──────────┐
  │ Client │───▶│ MemTable  │───▶│ Bloom Filter │───▶│ SSTables │
  │        │    │ (check)   │    │ (skip files) │    │ (search) │
  └────────┘    └───────────┘    └──────────────┘    └──────────┘
```

**Storage engine: LSM Tree (Log-Structured Merge Tree)**
- **Write path:** Append to WAL (durability), insert into MemTable (sorted in-memory structure). When MemTable is full, flush to immutable SSTable on disk.
- **Read path:** Check MemTable first, then use Bloom filters to skip irrelevant SSTables, then binary search within SSTables.
- **Compaction:** Background merge of SSTables to remove tombstones, reduce read amplification (leveled or size-tiered compaction).

---

## 6. Replication & Consistency

```
  Quorum Protocol (N=3, W=2, R=2):

  Client ──PUT──▶ Node A (primary)
                   │
                   ├──replicate──▶ Node B  ── ACK ──┐
                   │                                  │
                   ├──replicate──▶ Node C  ── ACK ──┤
                   │                                  │
                   ◀────── W=2 ACKs received ─────────┘
                   │
                   ▼
               ACK to Client

  Tunable consistency:
  • W + R > N  → strong consistency (e.g., W=2, R=2, N=3)
  • W=1, R=1   → eventual consistency (fast, may read stale)
  • W=N, R=1   → strong writes, fast reads
```

**Conflict resolution:**
- **Last-write-wins (LWW):** Use timestamps — simple but can lose data
- **Vector clocks:** Track causal ordering, detect conflicts, let application resolve
- **CRDTs:** Conflict-free data types that merge automatically (counters, sets)

---

## 7. Partitioning Strategy

```
  Option A: Hash Partitioning
  ┌─────────────────────────────────────┐
  │  hash(key) mod N → partition ID     │
  │  Pro: Even distribution             │
  │  Con: Range queries not supported   │
  └─────────────────────────────────────┘

  Option B: Range Partitioning
  ┌─────────────────────────────────────┐
  │  key "a-f" → Shard 1               │
  │  key "g-m" → Shard 2               │
  │  Pro: Range scans supported         │
  │  Con: Hotspots on popular ranges    │
  └─────────────────────────────────────┘

  Recommended: Consistent hashing with virtual nodes (Option A variant)
```

---

## 8. Failure Handling

```
  Failure Detection:
  ┌──────────┐  heartbeat  ┌──────────┐
  │  Node A  │◀───────────▶│  Node B  │
  │          │  every 1s   │          │
  └──────────┘             └──────────┘
       │                        │
       │    Gossip Protocol     │
       ▼                        ▼
  If no heartbeat for 5s → mark suspected
  If confirmed by 3+ nodes → mark dead

  Recovery:
  • Hinted handoff: temporarily store writes for dead node
  • Anti-entropy: Merkle tree comparison to sync replicas
  • Read repair: fix stale replicas during read operations
```

**Sloppy quorum + hinted handoff (Dynamo-style):**
- If a replica node is down, write to the next healthy node on the ring
- That node stores a "hint" and forwards the data when the original node recovers
- Trades consistency for availability

---

## 9. Scaling Strategy

- **Horizontal scaling:** Add nodes → consistent hashing redistributes ~1/N of keys
- **Split shards:** When a shard grows too large, split its token range
- **Tiered storage:** Hot data on SSD, warm on HDD, cold on object store
- **Read replicas:** Add follower replicas for read-heavy workloads
- **Caching layer:** In-process LRU cache or external cache (Redis) for hot keys

---

## 10. Observability

- **Metrics:** Read/write latency (p50, p95, p99), QPS per shard, replication lag, compaction backlog
- **Alerts:** Replication lag > threshold, node down, disk usage > 80%, write latency spike
- **Distributed tracing:** Trace a request from client → coordinator → replicas
- **Health checks:** Per-node health endpoint, cluster-wide status dashboard

---

---

## 11. Low-Level Design (LLD)

### API Contract
```
  POST   /api/v1/kv/{key}         — Upsert key-value pair
         Body: { "value": bytes, "ttl_seconds": int, "consistency": "ONE|QUORUM|ALL" }
         Response: { "version": int, "timestamp": HLC }

  GET    /api/v1/kv/{key}?consistency=QUORUM
         Response: { "value": bytes, "version": int, "timestamp": HLC }

  DELETE /api/v1/kv/{key}
         Response: { "deleted": true, "version": int }
```

### Internal Data Structures
```
  MemTable (Red-Black Tree / Skip List):
  ┌──────────────────────────────────────────────┐
  │  Operations: Insert O(log n), Lookup O(log n)│
  │  Size threshold: 64 MB → flush to SSTable    │
  │  Concurrent writes: lock-free skip list      │
  └──────────────────────────────────────────────┘

  SSTable (Sorted String Table):
  ┌──────────────────────────────────────────────┐
  │  ┌──────────┬──────────┬──────────┐          │
  │  │ Data     │ Index    │ Bloom    │          │
  │  │ Block    │ Block    │ Filter   │          │
  │  │ (sorted  │ (sparse  │ (false   │          │
  │  │  KV      │  key→    │  positive │          │
  │  │  pairs)  │  offset) │  ~1%)    │          │
  │  └──────────┴──────────┴──────────┘          │
  │  Compression: LZ4 (fast) or Zstd (ratio)    │
  └──────────────────────────────────────────────┘

  Compaction (Leveled):
  ┌──────────────────────────────────────────────┐
  │  L0: 4 SSTables → merge into L1             │
  │  L1: 10 SSTables (10 MB each)               │
  │  L2: 100 SSTables (10 MB each → 1 GB)       │
  │  L3: 1000 SSTables (→ 10 GB)                │
  │  Read amp: O(1) per level                    │
  │  Write amp: ~10× (trade-off for read perf)  │
  └──────────────────────────────────────────────┘
```

### Coordinator Node Logic
```
  On PUT(key, value):
  1. Hash key → identify partition (token range)
  2. Look up preference list (N=3 replicas)
  3. Send write to all 3 replicas in parallel
  4. Wait for W ACKs (configurable: 1, 2, or 3)
  5. If W ACKs received → return success
  6. If timeout → check hinted handoff nodes
  7. Store hint for unavailable nodes
```

---

## 12. Scalability

```
  Horizontal Scaling:
  ┌──────────────────────────────────────────────────────────┐
  │  Current: 10 nodes × 1 TB = 10 TB capacity              │
  │  Add 2 nodes → consistent hashing moves ~2/12 of keys   │
  │  Streaming: throttled to 50 MB/s to avoid perf impact    │
  │                                                          │
  │  Auto-scaling triggers:                                  │
  │  • Disk usage > 70% → add storage nodes                  │
  │  • CPU > 80% sustained → add compute nodes               │
  │  • p99 latency > 20ms → investigate hotspots             │
  └──────────────────────────────────────────────────────────┘

  Data Tiering:
  ┌──────────────────────────────────────────────────────────┐
  │  Hot tier  (SSD)    : last 24h data, p99 < 5ms          │
  │  Warm tier (HDD)    : 1-30 day data, p99 < 50ms         │
  │  Cold tier (S3)     : 30+ day data, p99 < 500ms         │
  │  Automatic migration based on access frequency           │
  └──────────────────────────────────────────────────────────┘

  Shard splitting:
  • Monitor shard size & QPS per shard
  • If shard > 50 GB or > 10K QPS → split token range
  • New sub-shard placed on least-loaded node
```

---

## 13. No Data Loss

```
  Write Durability Pipeline:
  ┌─────────────────────────────────────────────────────────┐
  │  1. Client write → WAL (fsync to disk)                  │
  │  2. WAL → MemTable (in-memory)                          │
  │  3. MemTable → SSTable flush (periodic)                 │
  │  4. Replicate WAL entry to W-1 replicas                 │
  │  ACK only after WAL persisted + quorum replicated       │
  └─────────────────────────────────────────────────────────┘

  Anti-Entropy (background repair):
  ┌─────────────────────────────────────────────────────────┐
  │  Merkle Tree comparison between replicas                │
  │  • Each replica builds hash tree over key ranges        │
  │  • Compare root hash → if mismatch, drill down          │
  │  • Exchange only divergent key ranges                   │
  │  Frequency: every 1 hour per shard                      │
  └─────────────────────────────────────────────────────────┘

  Hinted Handoff:
  • If replica is temporarily down, write to another node
  • Hint stored with TTL (e.g., 3 hours)
  • On recovery, hints are replayed to original replica
  • If hint TTL expires → anti-entropy will repair later

  Snapshot & Backup:
  • Daily incremental backup to object store (S3)
  • Point-in-time recovery via WAL replay
  • Cross-region replication for disaster recovery
```

---

## 14. Latency

```
  Read Path Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  1. Bloom filters: skip 99% of irrelevant SSTables     │
  │  2. Block cache: LRU cache of SSTable data blocks       │
  │  3. Key cache: cache frequently accessed key→offset     │
  │  4. Row cache: cache entire hot rows in memory          │
  │  5. Read-ahead: OS page cache for sequential reads      │
  └─────────────────────────────────────────────────────────┘

  Write Path Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  1. Group commit: batch WAL fsyncs (every 1ms)          │
  │  2. Concurrent memtable: lock-free skip list            │
  │  3. Async replication for eventual consistency           │
  │  4. Write-back cache for hot keys                       │
  └─────────────────────────────────────────────────────────┘

  Latency Targets:
  ┌──────────┬──────────┬──────────┐
  │ Op       │ p50      │ p99      │
  ├──────────┼──────────┼──────────┤
  │ GET      │ 1 ms     │ 5 ms    │
  │ PUT      │ 2 ms     │ 10 ms   │
  │ DELETE   │ 1 ms     │ 5 ms    │
  └──────────┴──────────┴──────────┘

  Tail Latency Mitigation:
  • Speculative reads: send to 2 replicas, use fastest response
  • Hedged requests: re-send after p95 timeout (e.g., 3ms)
  • Compaction throttling: limit I/O during peak hours
```

---

## 15. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Single node crash      │ Quorum reads/writes from        │
  │                        │ remaining replicas              │
  ├────────────────────────┼─────────────────────────────────┤
  │ Disk corruption        │ Checksums per SSTable block;    │
  │                        │ rebuild from replicas           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Network partition      │ Sloppy quorum + hinted handoff; │
  │                        │ configurable CP vs AP           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Cascading failure      │ Circuit breakers on coordinator;│
  │                        │ backpressure on writes          │
  ├────────────────────────┼─────────────────────────────────┤
  │ Split brain            │ Fencing tokens + leader lease   │
  │                        │ epoch numbers                   │
  └────────────────────────┴─────────────────────────────────┘

  Chaos Engineering:
  • Randomly kill nodes in staging to validate recovery
  • Inject network latency/partition between AZs
  • Simulate disk full / slow disk scenarios
```

---

## 16. Availability

```
  Target: 99.99% (52.6 min downtime/year)

  Multi-AZ Deployment:
  ┌────────────────────────────────────────────────────────┐
  │  AZ-1        AZ-2        AZ-3                         │
  │  ┌──────┐   ┌──────┐   ┌──────┐                      │
  │  │Node A│   │Node B│   │Node C│  ← replica placement  │
  │  │Node D│   │Node E│   │Node F│  across AZs           │
  │  └──────┘   └──────┘   └──────┘                      │
  │  Any single AZ failure → still have quorum (2/3)      │
  └────────────────────────────────────────────────────────┘

  Rolling Upgrades:
  1. Drain node (stop accepting new requests)
  2. Wait for in-flight requests to complete
  3. Upgrade binary
  4. Rejoin cluster, stream missed updates
  5. Move to next node (one at a time per AZ)

  DNS Failover: Route53 health checks → remove unhealthy regions
```

---

## 17. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication & Authorization:                        │
  │  • mTLS between nodes (intra-cluster communication)    │
  │  • API keys + RBAC per namespace (read/write/admin)    │
  │  • IAM integration for cloud deployments               │
  │                                                         │
  │  Encryption:                                            │
  │  • TLS 1.3 for client ↔ node traffic                   │
  │  • At-rest encryption: AES-256 for SSTables + WAL      │
  │  • Key rotation via KMS (AWS KMS / HashiCorp Vault)    │
  │                                                         │
  │  Data Protection:                                       │
  │  • Namespace isolation (multi-tenant)                   │
  │  • Audit logging for all mutations                     │
  │  • IP allowlisting for admin operations                │
  │  • Rate limiting per client to prevent abuse            │
  └─────────────────────────────────────────────────────────┘
```

---

## 18. Cost Constraints

```
  Cost Optimization Strategies:
  ┌─────────────────────────────────────────────────────────┐
  │  Storage:                                               │
  │  • Compression (LZ4/Zstd): 2-4× reduction             │
  │  • Tiered storage: SSD for hot, HDD for warm, S3 cold │
  │  • TTL-based auto-deletion reduces storage growth      │
  │  • Compaction reclaims tombstone space                  │
  │                                                         │
  │  Compute:                                               │
  │  • Right-size instances based on read/write ratio       │
  │  • Use spot/preemptible instances for read replicas    │
  │  • Auto-scale based on QPS (not fixed provisioning)    │
  │                                                         │
  │  Network:                                               │
  │  • Prefer same-AZ reads to reduce cross-AZ costs      │
  │  • Batch replication to reduce network overhead         │
  │  • Compress inter-node replication traffic              │
  │                                                         │
  │  Cost Estimate (100-node cluster):                      │
  │  • Storage: ~$15K/month (mixed SSD/HDD)                │
  │  • Compute: ~$30K/month (m5.2xlarge equivalent)        │
  │  • Network: ~$5K/month (cross-AZ replication)          │
  │  • Total: ~$50K/month for 10 TB, 500K read QPS        │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How do you handle a hot key?** (e.g., a viral tweet) — Read replicas, key-level caching, request coalescing
2. **How do you handle network partitions?** — Quorum config determines CP vs AP behavior
3. **What happens when you add a new node?** — Minimal data movement with consistent hashing + virtual nodes
4. **How do you handle clock skew for LWW?** — Use NTP, HLC (Hybrid Logical Clocks), or vector clocks
5. **How do you perform range queries on a hash-partitioned store?** — Secondary indexes or scatter-gather across all partitions
