# Caching — System Design Concepts & Interview Questions

---

## 1. What Is Caching?

Caching is the process of storing copies of data in a **high-speed storage layer** (cache) so that future requests for that data can be served faster than fetching it from the original, slower source (database, disk, remote API).

**Key goals:** reduce latency, reduce load on the backend/database, and improve throughput.

---

## 2. Where Can You Place a Cache? (Caching Layers)

| Layer | Examples | Typical Latency |
|---|---|---|
| **Client-side** | Browser cache, mobile app cache | < 1 ms |
| **CDN** | CloudFront, Akamai, Fastly | 5–50 ms |
| **API Gateway / Reverse Proxy** | Nginx, Varnish | 1–5 ms |
| **Application-level (in-process)** | HashMap, Guava, Caffeine | < 1 ms |
| **Distributed cache** | Redis, Memcached | 1–5 ms (network hop) |
| **Database query cache** | MySQL query cache, materialized views | varies |
| **Disk / OS page cache** | Linux page cache, mmap | µs range |

---

## 3. Caching Strategies (Read & Write Policies)

### 3.1 Read Strategies

#### Cache-Aside (Lazy Loading)
- Application checks cache first.
- On **cache miss**, application reads from DB, writes result to cache, then returns.
- On **cache hit**, return directly from cache.

```
App → Cache (hit?) → return
         ↓ (miss)
       DB → write to Cache → return
```

**Pros:** Only requested data is cached; cache failure doesn't break the system.
**Cons:** Cache miss = 3 round trips (check cache + read DB + write cache); data can become stale.

#### Read-Through
- Cache sits **in-line** between app and DB.
- On a miss, the **cache itself** fetches from DB and populates itself.
- App always talks to the cache, never directly to DB.

**Pros:** Simpler application code.
**Cons:** Initial request for each key is slow; tightly couples cache library to data source.

### 3.2 Write Strategies

#### Write-Through
- Every write goes to **cache AND DB synchronously**.
- Data in cache is always consistent with DB.

**Pros:** Strong consistency; reads after writes always see fresh data.
**Cons:** Higher write latency (two writes per operation); cache may hold rarely-read data.

#### Write-Behind (Write-Back)
- Write goes to **cache immediately**; cache asynchronously flushes to DB (batched/delayed).

**Pros:** Low write latency; can batch DB writes.
**Cons:** Risk of data loss if cache crashes before flushing; eventual consistency.

#### Write-Around
- Write goes **directly to DB**, bypassing the cache.
- Cache is only populated on reads (cache-aside).

**Pros:** Avoids filling cache with write-heavy data that may never be read.
**Cons:** Read after write = cache miss until the data is read and loaded.

---

## 4. Cache Eviction Policies

| Policy | Description | Use Case |
|---|---|---|
| **LRU** (Least Recently Used) | Evicts the item that hasn't been accessed for the longest time | General-purpose, most popular |
| **LFU** (Least Frequently Used) | Evicts the item with the fewest accesses | Data with stable popularity |
| **FIFO** (First In, First Out) | Evicts the oldest item | Simple, predictable |
| **TTL** (Time-To-Live) | Item expires after a fixed duration | Session data, tokens |
| **Random** | Evicts a random item | When all items are equally likely to be accessed |
| **TLRU** (Time-aware LRU) | LRU with TTL constraints | Content with validity windows |

### How LRU Works
- Uses a **doubly linked list + hash map**.
- On access: move node to head (O(1)).
- On eviction: remove tail node (O(1)).
- Lookup via hash map: O(1).

---

## 5. Cache Invalidation

Keeping cached data consistent with the source of truth is one of the **hardest problems** in caching.

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

### Invalidation Approaches

| Approach | How It Works |
|---|---|
| **TTL-based** | Set an expiry time; data auto-evicts after TTL. Simple but may serve stale data until expiry. |
| **Event-driven** | Publish an event on data change (e.g., DB trigger, CDC, message queue). Cache subscribes and invalidates. |
| **Write-through invalidation** | Cache is updated/invalidated on every write. |
| **Manual / API invalidation** | Application explicitly deletes or updates the cache key on write. |
| **Version-based** | Append a version/hash to the key. New versions = new cache keys; old keys expire naturally. |

---

## 6. Cache Consistency Patterns

### Strong Consistency
- Write-through cache or synchronous invalidation.
- Every read returns the latest write.
- Higher latency on writes.

### Eventual Consistency
- Write-behind cache or TTL-based invalidation.
- Reads may return stale data for a short window.
- Lower latency, higher throughput.

### Common Pitfalls
- **Thundering Herd (Cache Stampede):** Many concurrent requests hit a cache miss for the same key simultaneously, all query the DB.
  - **Fix:** Use a **lock/semaphore** so only one request fetches from DB; others wait. Or use **request coalescing**.
- **Cache Penetration:** Queries for keys that will **never** exist in DB keep bypassing cache.
  - **Fix:** Cache negative results (null) with short TTL; use a **Bloom filter** in front of the cache.
- **Cache Avalanche (Mass Expiry):** Many keys expire at the same time, causing a flood of DB requests.
  - **Fix:** Add **random jitter** to TTLs; warm the cache proactively; stagger expiry.
- **Hot Key Problem:** A single key receives disproportionate traffic.
  - **Fix:** Replicate the hot key across multiple cache nodes; use local in-process cache in front of distributed cache.

---

## 7. Distributed Caching

### Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | Strings, Lists, Sets, Hashes, Sorted Sets, Streams, HyperLogLog | Strings (key-value only) |
| Persistence | RDB snapshots + AOF | None (pure in-memory) |
| Replication | Master-replica | None built-in |
| Clustering | Redis Cluster (auto-sharding) | Client-side sharding |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Multi-threaded | Single-threaded (I/O threads in 6.x+) | Multi-threaded |
| Use case | Rich data needs, pub/sub, persistence | Simple, high-throughput key-value |

### Data Partitioning in Distributed Caches
- **Consistent Hashing:** Distributes keys across nodes on a hash ring. Adding/removing a node only moves ~1/n keys. Prevents massive re-hashing.
- **Hash Slots (Redis Cluster):** 16,384 slots distributed across nodes; each key hashes to a slot.

### Replication & High Availability
- **Redis Sentinel:** Monitors master, auto-failover to replica.
- **Redis Cluster:** Data sharding + replication combined.

---

## 8. CDN Caching

- CDNs (Content Delivery Networks) cache static & dynamic content at **edge locations** close to users.
- Controlled via HTTP headers: `Cache-Control`, `Expires`, `ETag`, `Last-Modified`.
- **Pull CDN:** Edge fetches from origin on first request, caches it.
- **Push CDN:** Origin uploads content to edge proactively.

### Key HTTP Cache Headers

| Header | Purpose |
|---|---|
| `Cache-Control: max-age=3600` | Cache for 3600 seconds |
| `Cache-Control: no-cache` | Must revalidate with origin before using cached copy |
| `Cache-Control: no-store` | Do not cache at all |
| `ETag` | Hash/version of the resource for conditional requests |
| `Last-Modified` / `If-Modified-Since` | Date-based conditional caching |
| `Vary` | Cache different versions based on request headers (e.g., `Accept-Encoding`) |

---

## 9. Application-Level Caching Patterns

### Memoization
- Cache the return value of a function based on its inputs.
- Common in recursive algorithms (e.g., Fibonacci, DP problems).

### Object / Session Caching
- Store serialized objects, user sessions, or computed results in Redis/Memcached.

### Query Result Caching
- Cache the result of expensive DB queries keyed by query hash + parameters.
- Invalidate on writes to underlying tables.

### Precomputation / Cache Warming
- Proactively populate the cache before traffic hits (e.g., on deploy, scheduled job).
- Prevents cold-start cache misses.

---

## 10. Caching Metrics

| Metric | Formula | Good Target |
|---|---|---|
| **Hit Rate** | hits / (hits + misses) | > 95% for read-heavy workloads |
| **Miss Rate** | 1 − hit rate | < 5% |
| **Latency (p50, p99)** | Time to respond from cache | < 1–5 ms |
| **Eviction Rate** | Evictions per second | Low and stable |
| **Memory Usage** | Current memory / max memory | Monitor to avoid OOM |

---

## 11. When NOT to Cache

- **Highly dynamic data** that changes on every request (e.g., real-time stock prices at tick level).
- **Write-heavy workloads** where invalidation overhead exceeds read benefit.
- **Low-read-frequency data** — caching wastes memory if data is rarely re-read.
- **Security-sensitive data** — caching PII/tokens increases attack surface.
- **When strong consistency is mandatory** and the invalidation cost is too high.

---

## 12. System Design — Where Caching Fits

### Example: Design a URL Shortener
- Cache popular short→long URL mappings in Redis.
- Strategy: Cache-aside with TTL.
- Eviction: LRU.

### Example: Design Twitter / News Feed
- Cache precomputed feed per user in Redis (fan-out on write).
- Invalidate/update on new tweet.
- Hot users: replicate cache across nodes.

### Example: Design an E-Commerce Product Page
- CDN caches static assets (images, CSS, JS).
- Application cache for product details (Redis, TTL = 5 min).
- Session cache for cart data (Redis, TTL = 30 min).
- DB-level: query cache for catalog searches.

---

## 13. Interview Questions

### Conceptual / Theory

1. **What is caching and why is it used in system design?**
2. **Explain the difference between cache-aside, read-through, write-through, and write-behind strategies. When would you choose each?**
3. **What are the common cache eviction policies? Explain LRU with its data structure.**
4. **What is cache invalidation? Why is it considered a hard problem?**
5. **Explain the thundering herd / cache stampede problem. How do you mitigate it?**
6. **What is cache penetration? How does a Bloom filter help?**
7. **What is cache avalanche? How do you prevent mass expiry?**
8. **Compare Redis vs Memcached. When would you pick one over the other?**
9. **What is consistent hashing and why is it important for distributed caches?**
10. **How does a CDN cache work? Explain pull vs push CDN.**
11. **What HTTP headers control caching behavior? Explain `Cache-Control`, `ETag`, and `Vary`.**
12. **What is the hot key problem? How would you solve it?**
13. **When should you NOT use caching?**
14. **How do you achieve cache consistency in a distributed system?**
15. **What metrics would you monitor for a caching system?**

### Design / Scenario-Based

16. **Design a distributed cache like Redis. What components would you need?**
    - Hash slots / consistent hashing, replication, persistence (RDB/AOF), eviction, pub/sub, cluster coordination.

17. **You have a service with 100K RPS reads and 1K RPS writes. Design the caching layer.**
    - Cache-aside with Redis; TTL for staleness tolerance; write-through or event-driven invalidation; monitor hit rate.

18. **Your cache hit rate dropped from 98% to 60% overnight. How do you debug?**
    - Check: key distribution change, TTL misconfiguration, traffic pattern shift, cache size/eviction spike, deployment that changed key format, new query patterns.

19. **How would you cache paginated API results?**
    - Key by `(query, page, page_size)`; invalidate on underlying data change; consider caching only first N pages.

20. **Design a multi-level caching architecture for a global e-commerce platform.**
    - L1: In-process cache (Caffeine), L2: Distributed cache (Redis Cluster), L3: CDN for static assets. Invalidation via event bus (Kafka). TTL decreases at each level.

21. **A single cache key is getting 50K RPS (hot key). What do you do?**
    - Add local in-process cache, replicate key across shards with suffix (`key:1`, `key:2`, ...), rate-limit or queue requests.

22. **How would you implement a cache that supports both TTL expiry and LRU eviction?**
    - Maintain an LRU list + per-key TTL. On access: check TTL first (lazy expiry), then promote in LRU. Background thread also sweeps expired keys.

23. **Compare caching at the application layer vs the database layer. Trade-offs?**
    - App-layer: flexible, key-value, works across data sources, but requires manual invalidation. DB-layer: transparent, auto-invalidated on writes, but limited to query results and DB-specific.

24. **How does cache warming work? When and why would you use it?**
    - Pre-populate cache before traffic arrives (on deploy, via batch job). Avoids cold-start penalty. Use when predictable access patterns exist.

25. **Explain how you would handle caching in a microservices architecture.**
    - Each service owns its cache; shared cache (Redis) for cross-service data; event-driven invalidation via message broker; avoid distributed cache as a shared database.

---

## 14. Quick Revision Cheat Sheet

```
Cache-Aside      → App checks cache, on miss reads DB & fills cache
Read-Through     → Cache auto-fetches from DB on miss
Write-Through    → Write to cache + DB synchronously
Write-Behind     → Write to cache, async flush to DB
Write-Around     → Write to DB only, cache on reads

LRU = Doubly Linked List + HashMap → O(1) get/put/evict
Consistent Hashing = Hash Ring → minimal key remapping on node change

Thundering Herd  → Lock/coalesce duplicate fetches
Cache Penetration→ Bloom filter + cache nulls
Cache Avalanche  → Random jitter on TTLs
Hot Key          → Replicate key + local cache

Redis > Memcached when you need: data structures, persistence, pub/sub, replication
Memcached > Redis when you need: simple key-value, multi-threaded performance
```

---
