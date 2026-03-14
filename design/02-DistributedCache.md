# Design a Distributed Cache

Examples: Redis Cluster, Memcached, Hazelcast

---

## 1. Requirements

### Functional
- `get(key)` — retrieve cached value
- `set(key, value, ttl)` — store with expiration
- `delete(key)` — invalidate entry
- Support multiple data types (string, hash, list, set)
- Support atomic operations (increment, compare-and-swap)

### Non-Functional
- Sub-millisecond read latency (< 1ms p99)
- High throughput (millions of ops/sec per cluster)
- Horizontal scalability
- High availability (replicated)
- Memory-efficient storage

---

## 2. Scale Estimation

```
Cache size:          500 GB across cluster
Average entry:       500 bytes (key + value)
Total entries:       ~1 billion
Read QPS:            2 million
Write QPS:           200K
Hit ratio target:    > 95%
Nodes needed:        ~16 nodes × 32 GB RAM each
```

---

## 3. High-Level Architecture

```
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ App Srv 1│    │ App Srv 2│    │ App Srv 3│
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       └───────────┬───┘───────────────┘
                   │
           ┌───────▼───────┐
           │  Cache Client  │
           │  (consistent   │
           │   hash ring)   │
           └───────┬───────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │ Cache   │ │ Cache   │ │ Cache   │
  │ Node 1  │ │ Node 2  │ │ Node 3  │
  │ (master)│ │ (master)│ │ (master)│
  └────┬────┘ └────┬────┘ └────┬────┘
       │           │           │
  ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
  │ Replica │ │ Replica │ │ Replica │
  └─────────┘ └─────────┘ └─────────┘
       │           │           │
       └───────────┴───────────┘
                   │
           ┌───────▼───────┐
           │   Database     │
           │  (source of    │
           │   truth)       │
           └───────────────┘
```

**Flow:**
1. Application calls cache client with a key
2. Client hashes key to determine which cache node owns it
3. On cache **hit** → return value (sub-ms)
4. On cache **miss** → query database, store result in cache, return to app

---

## 4. Caching Strategies

```
  ┌─────────────────────────────────────────────────────┐
  │              Cache-Aside (Lazy Loading)              │
  │                                                     │
  │  App ──GET──▶ Cache ──HIT──▶ return                │
  │                │                                    │
  │               MISS                                  │
  │                │                                    │
  │                ▼                                    │
  │            Database ──▶ write to cache ──▶ return   │
  └─────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │              Write-Through                           │
  │                                                     │
  │  App ──WRITE──▶ Cache ──WRITE──▶ Database           │
  │                   │                                  │
  │              ACK after both succeed                  │
  │  Pro: cache always consistent                       │
  │  Con: write latency increases                       │
  └─────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │              Write-Behind (Write-Back)               │
  │                                                     │
  │  App ──WRITE──▶ Cache ──ACK──▶ App                  │
  │                   │                                  │
  │              async queue                             │
  │                   │                                  │
  │                   ▼                                  │
  │               Database (batched writes)              │
  │  Pro: fast writes                                   │
  │  Con: data loss risk if cache crashes before flush   │
  └─────────────────────────────────────────────────────┘
```

**Recommendation:** Cache-aside is the most common pattern. Use write-through for strong consistency needs. Write-behind for write-heavy workloads where some data loss is acceptable.

---

## 5. Cache Eviction Strategies

```
  ┌────────────────────────────────────────────────────────┐
  │  LRU (Least Recently Used)                             │
  │  ├── HashMap + Doubly Linked List                      │
  │  ├── O(1) get, O(1) put                                │
  │  └── Most commonly used in production                  │
  │                                                        │
  │  LFU (Least Frequently Used)                           │
  │  ├── HashMap + Frequency counter + Min-Heap            │
  │  └── Better for skewed access patterns                 │
  │                                                        │
  │  TTL (Time-To-Live)                                    │
  │  ├── Each key has an expiration timestamp               │
  │  ├── Lazy expiration: check on access                   │
  │  └── Active expiration: background thread samples keys  │
  │                                                        │
  │  Random Eviction                                       │
  │  └── Simple, good enough for uniform access patterns    │
  └────────────────────────────────────────────────────────┘

  Redis approach: Approximated LRU
  - Sample N random keys, evict the one with oldest access time
  - Avoids overhead of maintaining a full LRU list
```

---

## 6. Cache Invalidation

```
  Problem: Stale data in cache after DB update

  Strategy 1: TTL-based expiration
  ┌──────────────────────────────────────┐
  │  Set TTL = 60s on every cache entry  │
  │  Max staleness = 60 seconds          │
  │  Simple, but not real-time           │
  └──────────────────────────────────────┘

  Strategy 2: Event-driven invalidation
  ┌──────────┐    ┌─────────┐    ┌───────┐
  │ DB Write │───▶│ Message │───▶│ Cache │
  │          │    │ Queue   │    │ Delete│
  └──────────┘    └─────────┘    └───────┘
  Near real-time, but adds complexity

  Strategy 3: Versioned keys
  ┌──────────────────────────────────────┐
  │  key = "user:123:v5"                 │
  │  On update: increment version → v6  │
  │  Old entry naturally expires via TTL │
  └──────────────────────────────────────┘
```

---

## 7. Horizontal Scaling

```
  Consistent Hash Ring with Slots:

  ┌──────────────────────────────────────────┐
  │  16,384 hash slots (Redis Cluster)       │
  │                                          │
  │  Node A: slots 0 – 5460                  │
  │  Node B: slots 5461 – 10922             │
  │  Node C: slots 10923 – 16383            │
  │                                          │
  │  Adding Node D:                          │
  │  → Migrate ~25% of slots from A, B, C   │
  │  → Key routing updated atomically        │
  │  → Clients get MOVED redirect            │
  └──────────────────────────────────────────┘
```

**Replication:**
- Each master has 1+ replicas (async replication)
- On master failure → replica promoted automatically
- Redis Cluster uses gossip protocol for failure detection

---

## 8. Thundering Herd Problem

```
  Problem: Popular key expires → 1000s of requests hit DB simultaneously

  ┌───────────────────────────────────────────────────┐
  │  Cache Miss for hot key "trending_post:1"         │
  │                                                   │
  │  Thread 1 ──▶ DB query ──▶ set cache              │
  │  Thread 2 ──▶ DB query ──▶ set cache  ← wasted   │
  │  Thread 3 ──▶ DB query ──▶ set cache  ← wasted   │
  │  ...                                              │
  │  Thread 1000 ──▶ DB query ← DB overwhelmed       │
  └───────────────────────────────────────────────────┘

  Solutions:

  1. Locking / Request Coalescing:
     First thread acquires lock → fetches from DB → others wait

  2. Probabilistic Early Expiration:
     Refresh cache before TTL expires (add jitter)

  3. Circuit Breaker:
     If DB is overloaded, serve stale data temporarily
```

---

## 9. Fault Tolerance

- **Master failure:** Replica promoted via leader election (Raft in Redis Cluster)
- **Network partition:** Split-brain risk — use minimum quorum to accept writes
- **Memory pressure:** Eviction kicks in; monitor eviction rate as a metric
- **Full cluster failure:** Warm cache from DB on restart (cache warming)

---

## 10. Observability

- **Metrics:** Hit ratio, miss ratio, eviction rate, memory usage, latency p99, connections
- **Alerts:** Hit ratio < 90%, memory > 85%, replication lag > threshold
- **Slow log:** Track commands exceeding latency threshold
- **Key analysis:** Identify hot keys and big keys (memory hogs)

---

---

## 11. Low-Level Design (LLD)

### API Contract
```
  GET    /cache/v1/{namespace}/{key}
         Headers: X-Consistency: local|quorum
         Response: { "value": bytes, "ttl_remaining": int, "version": int }

  PUT    /cache/v1/{namespace}/{key}
         Body: { "value": bytes, "ttl": int, "flags": "NX|XX|CAS" }
         NX = set if not exists, XX = set if exists, CAS = compare-and-swap
         Response: { "stored": bool, "version": int }

  DELETE /cache/v1/{namespace}/{key}
         Response: { "deleted": bool }

  MGET   /cache/v1/{namespace}/batch
         Body: { "keys": ["k1","k2","k3"] }
         Response: { "results": { "k1": val, "k2": val, "k3": null } }
```

### Internal Data Structures
```
  Per-Node Memory Layout:
  ┌──────────────────────────────────────────────┐
  │  Hash Table (open addressing):               │
  │  • Bucket array: 2^N entries                 │
  │  • Load factor < 0.75 → rehash              │
  │  • Key → pointer to Slab                     │
  │                                              │
  │  Slab Allocator (Memcached-style):           │
  │  ┌────────┬────────┬────────┐               │
  │  │ 64B    │ 128B   │ 256B   │  ...1MB       │
  │  │ class  │ class  │ class  │               │
  │  └────────┴────────┴────────┘               │
  │  Reduces memory fragmentation               │
  │  Pre-allocate pages per size class           │
  │                                              │
  │  LRU per size class:                         │
  │  head ↔ entry ↔ entry ↔ entry ↔ tail        │
  │  Access moves entry to head (O(1))           │
  └──────────────────────────────────────────────┘

  TTL Expiration Engine:
  ┌──────────────────────────────────────────────┐
  │  Lazy: check TTL on every GET                │
  │  Active: sample 20 random keys every 100ms   │
  │    if >25% expired → sample again            │
  │  Timing wheel for scheduled bulk expirations │
  └──────────────────────────────────────────────┘
```

### Replication Protocol
```
  Master → Replica (async streaming):
  1. Master executes command
  2. Command appended to replication buffer (ring buffer, 1 MB)
  3. Replica receives and replays command
  4. Offset tracking: replica reports consumed offset
  5. Full resync if replica falls too far behind (RDB snapshot)
```

---

## 12. Scalability

```
  Scaling Dimensions:
  ┌────────────────────────────────────────────────────────┐
  │  Read scaling:                                        │
  │  • Add read replicas (each master: 1-3 replicas)      │
  │  • Client-side caching (local LRU, invalidated by     │
  │    server push notifications)                         │
  │  • Connection pooling to reduce overhead               │
  │                                                        │
  │  Write scaling:                                        │
  │  • Add more shards (hash slots redistributed)          │
  │  • Pipeline multiple commands per round trip           │
  │  • Batch writes to reduce network overhead             │
  │                                                        │
  │  Memory scaling:                                       │
  │  • Add nodes → redistribute hash slots                 │
  │  • Data compression (LZF for values > 1 KB)           │
  │  • Use memory-efficient encodings (ziplist, intset)   │
  │                                                        │
  │  10 nodes × 64 GB = 640 GB usable cache               │
  │  At 2M QPS, ~200K QPS per node (achievable w/ pipelining)│
  └────────────────────────────────────────────────────────┘
```

---

## 13. No Data Loss

```
  Cache is NOT the source of truth — but data loss causes:
  ┌────────────────────────────────────────────────────────┐
  │  Risk 1: Master crash before replica sync              │
  │  → Accept: up to 1s of writes lost (async replication)│
  │  → Mitigate: WAIT command for synchronous replication │
  │    of critical keys                                    │
  │                                                        │
  │  Risk 2: Full cluster restart → cold cache             │
  │  → Mitigation: RDB snapshots every 5 min              │
  │  → AOF (Append Only File) with fsync every 1s         │
  │  → Cache warming from access logs on restart           │
  │                                                        │
  │  Risk 3: Write-behind buffer lost                      │
  │  → Mitigation: persist write-behind queue to disk     │
  │  → Dual-write to cache + DB for critical data          │
  │                                                        │
  │  Source of Truth Guarantee:                             │
  │  → Database is always authoritative                    │
  │  → Cache miss → DB read → cache populate               │
  │  → Maximum data loss = TTL window of staleness         │
  └────────────────────────────────────────────────────────┘
```

---

## 14. Latency

```
  Latency Targets:
  ┌──────────┬──────────┬──────────┐
  │ Op       │ p50      │ p99      │
  ├──────────┼──────────┼──────────┤
  │ GET      │ 0.1 ms   │ 0.5 ms  │
  │ SET      │ 0.1 ms   │ 0.5 ms  │
  │ MGET(10) │ 0.2 ms   │ 1.0 ms  │
  └──────────┴──────────┴──────────┘

  Optimizations:
  ┌────────────────────────────────────────────────────────┐
  │  Network:                                              │
  │  • Unix domain sockets for co-located apps: <0.05ms   │
  │  • Connection pooling (avoid TCP handshake overhead)   │
  │  • Pipelining: send N commands, read N responses       │
  │    → amortize RTT across batch                        │
  │                                                        │
  │  Memory:                                               │
  │  • Keep dataset < RAM to avoid swap → latency spike   │
  │  • Disable THP (Transparent Huge Pages) to reduce jitter│
  │  • Use jemalloc for predictable allocation             │
  │                                                        │
  │  CPU:                                                  │
  │  • I/O multiplexing (epoll): single thread handles    │
  │    100K+ connections                                   │
  │  • Avoid O(N) commands on large collections (KEYS *)  │
  │  • Use SCAN for iteration instead                      │
  └────────────────────────────────────────────────────────┘
```

---

## 15. Reliability

```
  Failure Modes & Recovery:
  ┌────────────────────────┬───────────────────────────────┐
  │ Failure                │ Recovery                      │
  ├────────────────────────┼───────────────────────────────┤
  │ Master crash           │ Replica auto-promoted via     │
  │                        │ Raft consensus among sentinels│
  │                        │ Failover time: 1-5 seconds    │
  ├────────────────────────┼───────────────────────────────┤
  │ Network partition      │ Isolated master steps down    │
  │                        │ after node-timeout; prevents  │
  │                        │ split-brain writes            │
  ├────────────────────────┼───────────────────────────────┤
  │ Memory exhaustion      │ Eviction policy (allkeys-lru);│
  │                        │ alert at 80% usage            │
  ├────────────────────────┼───────────────────────────────┤
  │ Slow commands          │ Slowlog tracking; kill long-  │
  │                        │ running scripts; TIMEOUT cfg  │
  ├────────────────────────┼───────────────────────────────┤
  │ Thundering herd on DB  │ Request coalescing + circuit  │
  │                        │ breaker; serve stale on error │
  └────────────────────────┴───────────────────────────────┘
```

---

## 16. Availability

```
  Target: 99.99% availability

  Architecture:
  ┌────────────────────────────────────────────────────────┐
  │  Each shard: 1 master + 2 replicas across 3 AZs       │
  │  Sentinel quorum: 3 sentinels (or Cluster gossip)     │
  │                                                        │
  │  AZ-1          AZ-2          AZ-3                      │
  │  ┌────────┐   ┌────────┐   ┌────────┐                 │
  │  │Master A│   │Replica │   │Replica │                 │
  │  │Master B│   │Master C│   │Replica │                 │
  │  │Replica │   │Replica │   │Master D│                 │
  │  └────────┘   └────────┘   └────────┘                 │
  │                                                        │
  │  Any single AZ failure → all shards still have quorum  │
  │                                                        │
  │  Multi-Region (DR):                                    │
  │  Active-passive: async replicate to secondary region   │
  │  Failover: DNS switch → 2-5 min RTO                   │
  │  RPO: last replication offset (typically < 1s)         │
  └────────────────────────────────────────────────────────┘

  Graceful Degradation:
  • Cache unavailable → app falls back to DB (higher latency)
  • Partial cluster failure → remaining nodes serve their slots
  • Client-side caching as L1 provides resilience to L2 cache issues
```

---

## 17. Security

```
  ┌────────────────────────────────────────────────────────┐
  │  Authentication:                                       │
  │  • AUTH command with password (Redis 6+: ACL users)    │
  │  • Per-user permissions: read-only, write-only, admin  │
  │  • ACL: restrict commands per user (no FLUSHALL, etc.) │
  │                                                        │
  │  Encryption:                                           │
  │  • TLS 1.3 for client ↔ cache connections              │
  │  • TLS for replication traffic (master ↔ replica)      │
  │  • No at-rest encryption (data is in memory);          │
  │    encrypt sensitive values at application layer       │
  │                                                        │
  │  Network:                                              │
  │  • Bind to private VPC / internal network only         │
  │  • No public internet exposure                         │
  │  • Security groups: allow only app server CIDRs        │
  │  • Disable dangerous commands: KEYS, FLUSHDB, DEBUG    │
  │                                                        │
  │  Multi-Tenancy:                                        │
  │  • Namespace isolation via key prefixes                 │
  │  • Per-namespace memory quotas                         │
  │  • Separate ACL users per tenant                       │
  └────────────────────────────────────────────────────────┘
```

---

## 18. Cost Constraints

```
  ┌────────────────────────────────────────────────────────┐
  │  Memory is the primary cost driver:                    │
  │                                                        │
  │  Optimization:                                         │
  │  • Right-size TTLs: shorter TTL = less memory needed   │
  │  • Remove unused keys: track idle time, evict          │
  │  • Use compact encodings (Redis ziplist for small sets)│
  │  • Compress values > 1KB at app layer (LZ4, Snappy)   │
  │  • Tiered: hot keys in cache, warm keys miss to DB     │
  │                                                        │
  │  Instance Selection:                                   │
  │  • Memory-optimized instances (r6g series)             │
  │  • Reserved instances for baseline: 40-60% savings    │
  │  • Spot instances for read replicas (non-critical)     │
  │                                                        │
  │  Cost Estimate (500 GB cluster):                       │
  │  • 16 nodes × r6g.2xlarge (64GB): ~$8K/month          │
  │  • Network (cross-AZ replication): ~$1K/month          │
  │  • Total: ~$9K/month                                   │
  │                                                        │
  │  vs. Managed (ElastiCache/MemoryDB):                   │
  │  • ~$12K/month (20-30% premium for managed ops)       │
  │  • Worth it for < 5 person ops team                    │
  └────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Cache-aside vs write-through?** — Cache-aside is simpler but can serve stale data; write-through guarantees consistency but adds write latency
2. **How do you handle cache warming after a cold start?** — Pre-populate from DB using access logs to identify hot keys
3. **How do you prevent thundering herd?** — Request coalescing with distributed locks + probabilistic early refresh
4. **Memcached vs Redis?** — Memcached is simpler (multi-threaded, string only); Redis has richer data types, persistence, replication, Lua scripting
5. **How do you handle a hot key?** — Replicate to multiple nodes, client-side caching, or key splitting (e.g., `key:shard:1`, `key:shard:2`)
