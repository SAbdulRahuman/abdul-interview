# Design a Web Crawler

Examples: Googlebot, Bingbot, Scrapy at scale

---

## 1. Requirements

### Functional
- Crawl billions of web pages across the internet
- Respect robots.txt and politeness policies
- Extract and normalize content (text, links, metadata)
- Prioritize pages by importance and freshness
- Detect and avoid crawler traps and duplicate content

### Non-Functional
- Crawl 1B+ pages per day
- Distributed across 1000+ workers
- Polite: max 1 request/second per domain
- Fresh: re-crawl important pages within hours
- Fault tolerant: resume after failures

---

## 2. Scale Estimation

```
Target:     1B pages/day
Per second: 1B / 86400 ≈ 11,600 pages/sec
Workers:    With 1 req/s politeness → ~12K workers
            But pipelining across domains → ~1K workers
Page size:  ~100 KB average (HTML)
Bandwidth:  11,600 × 100 KB = 1.16 GB/s ≈ 10 Gbps
Storage:    1B × 100 KB = 100 TB/day (raw HTML)
URLs known: ~100B URLs in frontier
```

---

## 3. High-Level Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                     Seed URLs                            │
  │              (news sites, Wikipedia, etc.)                │
  └──────────────────────────┬───────────────────────────────┘
                             │
                  ┌──────────▼──────────┐
                  │  URL Frontier       │
                  │  (priority queue +  │
                  │   politeness queue) │
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │  Scheduler          │
                  │  (assign URLs to    │
                  │   workers)          │
                  └──────────┬──────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Worker 1    │  │  Worker 2    │  │  Worker N    │
  │  • DNS       │  │              │  │              │
  │  • Fetch     │  │              │  │              │
  │  • Parse     │  │              │  │              │
  │  • Extract   │  │              │  │              │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │  Content     │ │  URL         │ │  Link        │
  │  Store       │ │  Dedup       │ │  Extractor   │
  │  (S3)        │ │  (Bloom      │ │  (new URLs   │
  │              │ │   filter)    │ │   → frontier) │
  └──────────────┘ └──────────────┘ └──────────────┘
```

---

## 4. URL Frontier Design

```
  The frontier is the core data structure:
  Two queues working together:

  PRIORITY QUEUE (what to crawl):
  ┌──────────────────────────────────────────────┐
  │  Priority based on:                          │
  │  • PageRank / domain authority               │
  │  • Content change frequency                  │
  │  • Time since last crawl                     │
  │  • Manual boost (news sites = high priority) │
  │                                              │
  │  Queue levels:                               │
  │  ┌──────────────────────────┐               │
  │  │ P0: news.com, cnn.com   │  re-crawl 1h  │
  │  │ P1: popular blogs       │  re-crawl 1d  │
  │  │ P2: general sites       │  re-crawl 1w  │
  │  │ P3: long-tail pages     │  re-crawl 1m  │
  │  └──────────────────────────┘               │
  └──────────────────────────────────────────────┘

  POLITENESS QUEUE (when to crawl):
  ┌──────────────────────────────────────────────┐
  │  Per-domain queues:                          │
  │                                              │
  │  example.com:  [url1, url5, url9]  next: +1s │
  │  news.com:     [url2, url7]        next: +2s │
  │  blog.org:     [url3, url4, url8]  next: +1s │
  │                                              │
  │  Rule: at most 1 request per domain per      │
  │  second (or per robots.txt Crawl-delay)      │
  │                                              │
  │  Each domain has a timer:                    │
  │  Don't dequeue url from domain D             │
  │  until D's timer expires                     │
  └──────────────────────────────────────────────┘
```

---

## 5. Worker Processing Pipeline

```
  Worker receives URL from scheduler:

  Step 1: Check robots.txt
  ┌──────────────────────────────────────────────┐
  │  Cache robots.txt per domain (TTL: 24h)      │
  │  Parse: User-agent: Googlebot                │
  │         Disallow: /private/                  │
  │         Crawl-delay: 2                       │
  │  If URL matches Disallow → skip              │
  └──────────────────────────────────────────────┘

  Step 2: DNS Resolution (cached)
  ┌──────────────────────────────────────────────┐
  │  Local DNS cache: domain → IP                │
  │  Reduces DNS queries from billions to millions│
  └──────────────────────────────────────────────┘

  Step 3: Fetch page
  ┌──────────────────────────────────────────────┐
  │  HTTP GET with timeout (30s)                 │
  │  Follow redirects (max 5 hops)               │
  │  Handle: 200 OK, 301/302 redirect,           │
  │          404 not found, 429 rate limited,     │
  │          5xx server error                     │
  │  Store raw HTML + response headers           │
  └──────────────────────────────────────────────┘

  Step 4: Parse & Extract
  ┌──────────────────────────────────────────────┐
  │  Parse HTML → DOM tree                       │
  │  Extract:                                    │
  │  • Title, meta description, canonical URL    │
  │  • Body text (strip tags)                    │
  │  • All href links → normalize to absolute    │
  │  • Structured data (schema.org)              │
  │  • Language detection                        │
  └──────────────────────────────────────────────┘

  Step 5: Content fingerprinting
  ┌──────────────────────────────────────────────┐
  │  SimHash / MinHash of page content           │
  │  Compare with existing fingerprints          │
  │  If near-duplicate → skip indexing           │
  │  (Detects mirrors, syndicated content)       │
  └──────────────────────────────────────────────┘

  Step 6: Feed extracted links back to frontier
  ┌──────────────────────────────────────────────┐
  │  New URLs discovered: [link1, link2, ...]    │
  │  Check Bloom filter: already seen?           │
  │  If new → add to URL frontier for crawling   │
  └──────────────────────────────────────────────┘
```

---

## 6. URL Deduplication

```
  Bloom Filter for URL dedup:
  ┌──────────────────────────────────────────────┐
  │  100B URLs seen → Bloom filter               │
  │  False positive rate: 1%                     │
  │  Size: ~120 GB (at 1% FPR for 100B elements) │
  │                                              │
  │  Check: is URL in Bloom filter?              │
  │  Yes → probably seen, skip                   │
  │  No  → definitely new, add to frontier       │
  │                                              │
  │  Distributed: partition by hash(URL)         │
  │  Each worker checks its partition             │
  │                                              │
  │  URL normalization before dedup:             │
  │  • Remove trailing slash                     │
  │  • Lowercase hostname                        │
  │  • Remove default port (:80, :443)           │
  │  • Sort query parameters                     │
  │  • Remove fragment (#section)                │
  │  • Decode percent-encoding                   │
  └──────────────────────────────────────────────┘
```

---

## 7. Crawler Trap Detection

```
  Common traps:
  ┌──────────────────────────────────────────────┐
  │  1. Infinite calendars                       │
  │     /calendar/2026/01/01                     │
  │     /calendar/2026/01/02                     │
  │     /calendar/2026/01/03  ... (forever)      │
  │     → Detect: max URL depth per domain       │
  │                                              │
  │  2. Session ID in URLs                       │
  │     /page?session=abc123                     │
  │     /page?session=def456  (same content)     │
  │     → Detect: strip session parameters       │
  │                                              │
  │  3. Soft 404s                                │
  │     Server returns 200 for non-existent pages│
  │     → Detect: content fingerprint similarity │
  │                                              │
  │  4. Dynamically generated pages              │
  │     /products?sort=price&page=1&color=red    │
  │     → Detect: limit parameter combinations   │
  │                                              │
  │  Protection:                                 │
  │  • Max URLs per domain: 1M                   │
  │  • Max URL length: 2048 chars                │
  │  • Max depth from seed: 15 hops              │
  │  • Timeout per page: 30 seconds              │
  └──────────────────────────────────────────────┘
```

---

## 8. Fault Tolerance

- **Worker crash:** URL returned to frontier; another worker picks it up
- **DNS failure:** Retry with exponential backoff; skip domain if persistent
- **Network timeout:** Retry once; skip and schedule for later
- **Frontier data loss:** Checkpoint frontier to durable storage periodically
- **Bloom filter loss:** Rebuild from crawl log; accept temporary duplicates

---

## 9. Low-Level Design (LLD)

### API Contracts

```
# Submit Seed URLs
POST /api/v1/crawl/seeds
{
  "urls": ["https://example.com", "https://news.ycombinator.com"],
  "priority": "high",          // high | medium | low
  "max_depth": 5,
  "crawl_policy": {
    "respect_robots_txt": true,
    "max_pages_per_domain": 100000,
    "crawl_delay_ms": 1000
  }
}

# Crawl Status
GET /api/v1/crawl/stats
Response: {
  "pages_crawled": 4200000000,
  "pages_in_frontier": 850000000,
  "crawl_rate_pages_per_sec": 12000,
  "domains_active": 45000000,
  "avg_page_size_kb": 85,
  "error_rate": 0.03,
  "storage_used_tb": 320
}

# Domain Policy Override
PUT /api/v1/crawl/domains/{domain}/policy
{
  "crawl_delay_ms": 2000,
  "max_pages": 50000,
  "priority_boost": 1.5,
  "blocked": false
}

# Internal: Worker Fetch Task
GET /internal/v1/frontier/next?worker_id=w_42&batch_size=100
Response: {
  "tasks": [
    {
      "url": "https://example.com/page1",
      "domain": "example.com",
      "depth": 2,
      "priority": 0.85,
      "last_crawled": null,
      "etag": null
    }
  ]
}
```

### URL Frontier Internal Design

```
URL Frontier Architecture:
┌──────────────────────────────────────────────────────┐
│                 Front Queue (Priority)               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │
│  │ High   │ │ Medium │ │ Low    │ │ Re-    │       │
│  │ Prio   │ │ Prio   │ │ Prio   │ │ crawl  │       │
│  │ Queue  │ │ Queue  │ │ Queue  │ │ Queue  │       │
│  └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘       │
│       └──────────┼──────────┼──────────┘            │
│                  ▼                                    │
│         Priority Selector                            │
│    (weighted: 60% high, 25% med, 10% low, 5% re)   │
├──────────────────────────────────────────────────────┤
│                 Back Queue (Politeness)               │
│  One queue per domain (ensures crawl delay)          │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ example.com  │  │ wikipedia.org│  │ github.com │ │
│  │ next_fetch:  │  │ next_fetch:  │  │ next_fetch:│ │
│  │ 10:30:05.000 │  │ 10:30:05.500 │  │ 10:30:06.0 │ │
│  │ delay: 1000ms│  │ delay: 500ms │  │ delay: 2s  │ │
│  │ [url1,url2]  │  │ [url3,url4]  │  │ [url5]     │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│                                                      │
│  Heap ordered by next_fetch_time                     │
│  Worker polls: pop domain where now >= next_fetch    │
└──────────────────────────────────────────────────────┘
```

### Worker Processing Pipeline (Internal)

```
Worker Pipeline (per URL):
┌─────────────────────────────────────────────────┐
│ 1. DNS Resolution (cached, TTL-aware)           │
│    dns_cache[domain] → IP                       │
│    If miss: resolve + cache for min(TTL, 300s)  │
├─────────────────────────────────────────────────┤
│ 2. robots.txt Check (cached per domain)         │
│    Parse directives for User-agent: *           │
│    Respect Crawl-delay, Disallow paths          │
│    Cache TTL: 24 hours                          │
├─────────────────────────────────────────────────┤
│ 3. HTTP Fetch                                   │
│    GET url with If-None-Match: {etag}           │
│    Timeout: connect=5s, read=30s                │
│    Follow redirects: max 5 hops                 │
│    If 304 Not Modified → skip processing        │
│    If 429 → exponential backoff per domain      │
├─────────────────────────────────────────────────┤
│ 4. Content Processing                           │
│    Detect encoding (charset from header/meta)   │
│    Parse HTML (fast: lxml, not full browser)     │
│    Extract: title, text, links, metadata        │
│    Normalize URLs (remove fragments, canonicalize│
├─────────────────────────────────────────────────┤
│ 5. Dedup Check                                  │
│    URL fingerprint: SHA-1(canonicalized_url)    │
│    Content fingerprint: SimHash(text)           │
│    Check Bloom filter (in-memory, FPR=0.1%)     │
│    If seen → skip                               │
├─────────────────────────────────────────────────┤
│ 6. Store + Enqueue                              │
│    Store raw HTML → S3 (key: sha1(url))         │
│    Send extracted text → Indexer (Kafka topic)  │
│    Enqueue discovered URLs → Frontier           │
└─────────────────────────────────────────────────┘
```

## 10. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Crawl Rate** | Horizontal worker scaling; each worker handles 50-100 pages/sec | 10K+ pages/sec with 200 workers |
| **Frontier Size** | Disk-backed priority queue (RocksDB); in-memory heap for scheduling | 10Bn URLs |
| **URL Dedup** | Distributed Bloom filter (partitioned by URL hash) + periodic consolidation | 50Bn URLs with 0.1% FPR |
| **Content Store** | S3 with date-partitioned prefixes; lifecycle policy for old crawls | Petabyte scale |
| **DNS Resolution** | Local DNS cache + dedicated DNS resolver cluster | 100K lookups/sec |

**Scaling Architecture:**
```
Worker Scaling:
  10 workers    → 500 pages/sec   (small site crawl)
  100 workers   → 5K pages/sec    (medium scale)
  1000 workers  → 50K pages/sec   (web-scale)
  10000 workers → 500K pages/sec  (Google-scale)

Domain Partitioning:
  Frontier sharded by hash(domain)
  Each frontier shard serves N workers
  Worker-to-shard assignment: consistent hashing
  Benefits: per-domain politeness guaranteed within single shard
```

## 11. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Crawled Pages** | Raw HTML stored in S3 (11 nines durability); content-addressable by SHA-1(url) |
| **Frontier Queue** | Disk-backed (RocksDB WAL); crash recovery replays from checkpoint |
| **URL State** | Bloom filter is approximate — false positives OK (skip re-crawl); rebuild from crawl log if needed |
| **Extracted Content** | Kafka topics with acks=all; consumer commits offset after indexer ACK |
| **Crawl Metadata** | Per-URL crawl record (status, etag, last_crawl) in Cassandra RF=3 |

**Recovery:**
```
Worker Crash Recovery:
  1. URLs in-flight for crashed worker → timeout after 5min
  2. Frontier marks URLs as "available" again
  3. Another worker picks them up (at-least-once crawling)
  4. Dedup prevents double-indexing of already-processed pages

Frontier Crash Recovery:
  1. RocksDB WAL replayed on restart
  2. Worst case: rebuild frontier from crawl log + seed URLs
  3. Bloom filter rebuilt from URL store scan (takes hours, but safe)
```

## 12. Latency

| Operation | p50 | p99 | Notes |
|-----------|-----|-----|-------|
| DNS resolution (cached) | 0.1ms | 1ms | Local cache hit |
| DNS resolution (miss) | 20ms | 200ms | Recursive resolver |
| HTTP fetch | 200ms | 2s | Depends on target server |
| HTML parse + extract | 5ms | 50ms | lxml, not headless browser |
| Bloom filter lookup | 0.01ms | 0.1ms | In-memory |
| S3 write | 10ms | 100ms | Async, non-blocking |
| End-to-end per page | 250ms | 3s | Dominated by HTTP fetch |

**Latency Optimization:**
- **Async I/O**: Use asyncio/epoll for concurrent HTTP fetches per worker (100 concurrent connections)
- **Connection pooling**: HTTP keep-alive per domain; reuse TCP connections
- **Conditional GET**: If-None-Match/If-Modified-Since to skip unchanged pages (saves bandwidth+processing)
- **Streaming parse**: Parse HTML as it downloads (chunked transfer)
- **Pre-fetching DNS**: Resolve DNS for next batch while current batch processes

## 13. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Worker crash** | In-flight URLs lost temporarily | Frontier timeout + re-enqueue; at-least-once guarantee |
| **Target site down** | URLs stuck in retry | Exponential backoff per domain; max retries=3; move to low-priority queue |
| **Crawler trap** | Infinite URL generation | Depth limit (max 15); URL pattern detection; per-domain page cap |
| **DNS failure** | Can't resolve domains | Fallback: secondary DNS provider; cache with extended TTL on failure |
| **Frontier overload** | Backpressure on workers | Workers slow down via adaptive rate limiting; frontier disk spills to larger volume |
| **Bloom filter full** | High false-positive rate | Monitor FPR; rebuild with larger filter when FPR > 1%; use counting Bloom for deletions |

## 14. Availability

**Target: 99.9% crawl availability (crawling is batch, not user-facing)**

```
Availability Architecture:
┌────────────────────────────────────────────────┐
│              Crawl Orchestrator                │
│    (Assigns domains to worker groups)          │
│    Active-passive: heartbeat + failover        │
├──────────────────┬─────────────────────────────┤
│  Worker Pool A   │  Worker Pool B              │
│  (500 workers)   │  (500 workers)              │
│  Domains: A-M    │  Domains: N-Z              │
│  Auto-scaling    │  Auto-scaling              │
│  Min: 100        │  Min: 100                  │
│  Max: 1000       │  Max: 1000                 │
├──────────────────┴─────────────────────────────┤
│            Frontier (3 replicas)               │
│   RocksDB + Raft consensus for state           │
├────────────────────────────────────────────────┤
│            Content Store (S3)                  │
│   11 nines durability; multi-AZ               │
└────────────────────────────────────────────────┘

Graceful Degradation:
  1. Worker pool < 50% → Reduce crawl rate, prioritize important domains
  2. Frontier degraded → Pause enqueue of new URLs, continue processing existing
  3. S3 write failures → Buffer locally, retry with exponential backoff
  4. DNS outage → Serve from cache (extended TTL), skip new domains
```

## 15. Security

| Layer | Mechanism |
|-------|-----------|
| **robots.txt** | Strict compliance; respect Crawl-delay; honor noindex/nofollow |
| **Rate Limiting** | Per-domain rate limiting (respect server capacity); adaptive based on response times |
| **Content Filtering** | Skip malicious content (malware pages, phishing); virus scan downloaded files |
| **Network Isolation** | Crawl workers in isolated VPC; no access to internal services |
| **Outbound Filtering** | Block requests to internal IP ranges (SSRF prevention: reject 10.x, 172.16.x, 192.168.x) |
| **TLS Verification** | Validate server certificates; log certificate transparency violations |
| **Abuse Prevention** | Identify and block honeypot traps; respect legal requirements (DMCA, GDPR) |
| **Credential Safety** | Never submit forms; never follow login redirects; stateless crawling (no cookies stored) |

## 16. Cost Constraints

**Estimated Cost (4 Billion pages/month, refresh cycle: 30 days):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **Crawl Workers** | 500× c6g.medium (spot instances, 70% savings) | $8,000 |
| **Frontier Servers** | 3× r6g.2xlarge (64GB RAM, NVMe) | $2,500 |
| **Content Storage** | S3: 320TB raw HTML (compressed) | $7,400 |
| **Bandwidth (egress)** | Minimal (crawl is inbound) | $500 |
| **Kafka (indexer queue)** | 6-broker MSK | $3,000 |
| **DNS Resolution** | Dedicated resolver cluster | $500 |
| **Total** | | **~$22,000/month** |

**Cost Optimization:**
- **Spot instances for workers**: 70% savings; workers are stateless, interruption-tolerant
- **Conditional GET**: 30-40% pages unchanged → 304 saves bandwidth + processing
- **Compression**: gzip/brotli for stored HTML (5:1 ratio typical) — saves 80% storage
- **Tiered re-crawl**: Important domains = daily; average = weekly; tail = monthly
- **Dedup before store**: Content-hash dedup prevents storing mirror/syndicated content multiple times

## Key Interview Discussion Points

1. **BFS vs DFS vs Best-First crawling?** — BFS gives breadth; best-first prioritizes high-PageRank pages. Best-first recommended
2. **How to handle JavaScript-rendered pages?** — Headless browser (Puppeteer); expensive, use only for high-value sites
3. **How to allocate crawl budget?** — Important domains get more budget; less important = crawl less frequently
4. **How to ensure freshness?** — Adaptive re-crawl intervals based on change frequency (if page changes hourly, re-crawl hourly)
5. **How to scale to 1000 workers?** — Partition frontier by domain hash; each worker assigned domain partitions; no contention
