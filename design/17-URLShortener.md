# Design a URL Shortener

Examples: Bitly, TinyURL, goo.gl

---

## 1. Requirements

### Functional
- Shorten a long URL to a short link (e.g., `short.ly/abc123`)
- Redirect short URL to original URL
- Custom aliases (optional)
- Link analytics (click count, referrer, geo)
- URL expiration (TTL)

### Non-Functional
- 100M URLs created per month
- 10:1 read:write ratio → 1B redirects per month
- Low latency redirection (< 10ms)
- High availability (99.99%)
- Short URLs should not be guessable

---

## 2. Scale Estimation

```
Writes:   100M / month → ~40 URLs/sec
Reads:    1B / month → ~400 redirects/sec
Peak:     ~2000 redirects/sec (5× peak factor)
Storage:  100M × 1 KB/URL × 12 months × 5 years = 6 TB
URL length: 7 chars base62 → 62^7 = 3.5 trillion unique URLs
```

---

## 3. High-Level Architecture

```
  ┌──────────┐       ┌──────────────┐       ┌──────────────┐
  │  Client   │──────▶│  API Gateway │──────▶│  URL Service │
  │  Browser  │       │  (rate limit)│       │              │
  └──────────┘       └──────────────┘       └──────┬───────┘
                                                    │
                            ┌───────────────────────┼───────────────┐
                            │                       │               │
                     ┌──────▼──────┐         ┌──────▼──────┐ ┌─────▼──────┐
                     │  Cache       │         │  Database    │ │ Analytics  │
                     │  (Redis)     │         │  (Key-Value) │ │ (Kafka →   │
                     │  hot URLs    │         │  shortURL →  │ │  ClickHouse│
                     │              │         │  longURL     │ │  )         │
                     └─────────────┘         └─────────────┘ └────────────┘

  Redirect flow:
  ┌──────────┐  GET /abc123   ┌──────────┐  cache    ┌───────┐
  │  Browser  │──────────────▶│  Service  │──hit?───▶│ Redis │
  └──────────┘               └────┬─────┘          └───────┘
       ▲                          │ miss
       │  301/302 redirect        ▼
       │                    ┌──────────┐
       └────────────────────│    DB    │
         Location: long_url └──────────┘
```

---

## 4. Short URL Generation

```
  Option A: Counter-based (Snowflake ID)
  ┌──────────────────────────────────────────────┐
  │  Auto-increment counter → encode base62      │
  │  ID: 1000000 → base62: "4c92"               │
  │                                              │
  │  Pro: No collision, simple                   │
  │  Con: Predictable, single point of failure   │
  │       for counter                            │
  │  Fix: Use distributed ID generator (Snowflake│
  │       or pre-allocated ID ranges)            │
  └──────────────────────────────────────────────┘

  Option B: Hash-based
  ┌──────────────────────────────────────────────┐
  │  MD5(longURL) → take first 7 chars base62    │
  │  "https://example.com/..." → "a8f3bc2"      │
  │                                              │
  │  Pro: Deterministic, same URL = same short   │
  │  Con: Collisions possible                    │
  │  Fix: Check DB for collision; if exists,     │
  │       append counter and retry               │
  └──────────────────────────────────────────────┘

  Option C: Pre-generated keys (recommended)
  ┌──────────────────────────────────────────────┐
  │  Key Generation Service (KGS):               │
  │  Pre-generate millions of unique 7-char keys │
  │  Store in key pool DB                        │
  │  On create: fetch & mark key as used         │
  │                                              │
  │  Two tables:                                 │
  │  ┌──────────────┐  ┌──────────────┐         │
  │  │ unused_keys  │  │ used_keys    │         │
  │  │ abc1234      │  │ xyz9876      │         │
  │  │ def5678      │  │ ...          │         │
  │  │ ...          │  │              │         │
  │  └──────────────┘  └──────────────┘         │
  │                                              │
  │  Pro: No collision, fast assignment          │
  │  Con: Extra service to maintain              │
  └──────────────────────────────────────────────┘
```

---

## 5. Database Design

```
  URL Table:
  ┌──────────────────────────────────────┐
  │  short_url  (PK)   │ VARCHAR(7)     │
  │  long_url           │ TEXT           │
  │  user_id            │ BIGINT         │
  │  created_at         │ TIMESTAMP      │
  │  expires_at         │ TIMESTAMP      │
  │  click_count        │ BIGINT         │
  └──────────────────────────────────────┘

  Choice: Key-Value store (DynamoDB, Cassandra)
  • Write pattern: single key insert
  • Read pattern: single key lookup
  • No joins needed
  • Partition key: short_url
```

---

## 6. Caching Strategy

```
  Cache: Redis / Memcached
  ┌──────────────────────────────────────────────┐
  │  Key: "abc123"                               │
  │  Value: "https://example.com/long/path"      │
  │  TTL: 24 hours                               │
  │                                              │
  │  80/20 rule: 20% of URLs get 80% of traffic  │
  │  Cache the hot 20% → ~100M × 20% × 1KB      │
  │  = 20 GB (fits in a single Redis instance)   │
  │                                              │
  │  Cache eviction: LRU                         │
  │  Cache warming: preload popular URLs on      │
  │                deploy                        │
  └──────────────────────────────────────────────┘

  301 vs 302 redirect:
  • 301 (Permanent): Browser caches → less server load
                     But you lose analytics (browser skips you)
  • 302 (Temporary): Every request hits server → accurate analytics
                     Higher load
  → Use 302 if analytics matter, 301 otherwise
```

---

## 7. Analytics Pipeline

```
  Every redirect → emit click event:
  ┌─────────────────────────────────────┐
  │  {                                  │
  │    "short_url": "abc123",           │
  │    "timestamp": "2026-03-12T10:00", │
  │    "ip": "1.2.3.4",                │
  │    "user_agent": "Chrome/120",      │
  │    "referrer": "twitter.com",       │
  │    "country": "US"                  │
  │  }                                  │
  └──────────┬──────────────────────────┘
             │
             ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │  Kafka        │────▶│  Flink /     │────▶│  ClickHouse  │
  │  (buffer)     │     │  Aggregator  │     │  (analytics) │
  └──────────────┘     └──────────────┘     └──────────────┘
                                                    │
                                              Dashboard:
                                              clicks/day, top referrers,
                                              geo distribution
```

---

## 8. Fault Tolerance

- **Database replication:** Primary-secondary with async replication; promote on failure
- **Cache failure:** Fall back to DB; slightly higher latency but still works
- **Multi-region:** DNS-based routing to nearest region; each region has full copy
- **Data loss:** WAL on DB; periodic snapshots; replicate to standby

---

---

## 9. Low-Level Design (LLD)

### API Contract
```
  Shorten:
  POST   /api/v1/shorten
         { "url": "https://example.com/very/long/path?q=abc",
           "custom_alias": "my-link",       (optional)
           "expiry_days": 30,               (optional, default: never)
           "password": "secret"             (optional) }
         Response: { "short_url": "https://sho.rt/aB3kd9",
                     "short_code": "aB3kd9", "expires_at": "..." }

  Redirect:
  GET    /{short_code}
         301 Redirect → Location: https://example.com/very/long/path?q=abc
         (or 302 for temporary / A/B test links)

  Analytics:
  GET    /api/v1/stats/{short_code}
         { "total_clicks": 15234, "unique_visitors": 8921,
           "by_country": {"US": 5000, "IN": 3000, ...},
           "by_device": {"mobile": 60, "desktop": 35, "tablet": 5},
           "by_day": [{"date": "2024-03-13", "clicks": 500}, ...] }

  Management:
  DELETE /api/v1/urls/{short_code}
  PUT    /api/v1/urls/{short_code}    (update destination URL)
  GET    /api/v1/urls?user_id=123     (list user's URLs)
```

### Database Schema
```
  Table: urls
  ┌────────────────────────────────────────────────────────┐
  │  short_code   VARCHAR(10)  PRIMARY KEY                 │
  │  original_url TEXT         NOT NULL                    │
  │  user_id      BIGINT       (nullable, for registered)  │
  │  created_at   TIMESTAMP    DEFAULT NOW()               │
  │  expires_at   TIMESTAMP    (nullable)                  │
  │  password_hash VARCHAR(60) (nullable, bcrypt)           │
  │  click_count  BIGINT       DEFAULT 0                   │
  │  is_active    BOOLEAN      DEFAULT TRUE                │
  │                                                        │
  │  INDEX: idx_user_urls (user_id, created_at DESC)       │
  │  INDEX: idx_expiry (expires_at) WHERE expires_at IS NOT NULL │
  └────────────────────────────────────────────────────────┘

  Table: clicks (analytics, append-only)
  ┌────────────────────────────────────────────────────────┐
  │  click_id     BIGINT       AUTO_INCREMENT PK           │
  │  short_code   VARCHAR(10)  INDEX                       │
  │  clicked_at   TIMESTAMP    INDEX                       │
  │  ip_hash      VARCHAR(64)  (hashed for privacy)        │
  │  country      CHAR(2)                                  │
  │  device_type  ENUM(mobile,desktop,tablet)              │
  │  referrer     VARCHAR(255)                             │
  │  user_agent   VARCHAR(255)                             │
  │                                                        │
  │  Partitioned by: clicked_at (monthly)                  │
  └────────────────────────────────────────────────────────┘

  Key Generation (Base62):
  ┌────────────────────────────────────────────────────────┐
  │  Option 1: Counter-based (ZooKeeper range allocation)  │
  │    Server A gets range [1M-2M], Server B gets [2M-3M] │
  │    Convert counter to base62: 1000000 → "4C92"         │
  │    Pro: no collision. Con: sequential, guessable       │
  │                                                        │
  │  Option 2: MD5/SHA hash + truncate                     │
  │    MD5("https://example.com") → take first 7 chars     │
  │    On collision: append counter, rehash                 │
  │    Pro: deterministic. Con: collision handling           │
  │                                                        │
  │  Option 3: Snowflake ID → Base62                       │
  │    Globally unique, time-ordered, not guessable         │
  │    Pro: best overall. Con: 8-10 char codes              │
  └────────────────────────────────────────────────────────┘
```

---

## 10. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Read Heavy (100:1 read/write ratio):                   │
  │  • 100K redirects/sec, 1K creates/sec                   │
  │  • Multi-tier caching absorbs 99% of reads              │
  │                                                         │
  │  Caching Tiers:                                         │
  │  • CDN: cache 301 redirects at edge (TTL 1h)           │
  │  • Application cache (Redis): top 20% URLs = 80% hits  │
  │  • DB read replicas: for cache misses                   │
  │                                                         │
  │  Database Scaling:                                      │
  │  • Shard by short_code hash (consistent hashing)        │
  │  • Each shard: 1Bn URLs at 200 bytes = ~200 GB          │
  │  • 10 shards = 10Bn URLs capacity                       │
  │  • Analytics: separate DB (ClickHouse or Cassandra)     │
  │                                                         │
  │  Stateless API Servers:                                 │
  │  • Scale horizontally behind load balancer              │
  │  • 10 servers × 10K RPS each = 100K RPS                │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  URL Mapping (critical — lost mapping = broken links):  │
  │  • Primary DB: synchronous replication to standby        │
  │  • Daily snapshots: S3 backup with PITR capability       │
  │  • WAL archiving: continuous to S3                       │
  │  • Cross-region replica for DR                           │
  │                                                         │
  │  Analytics (less critical — eventual consistency OK):    │
  │  • Click events: write to Kafka first (durable)          │
  │  • Kafka retention: 7 days (reprocessing window)         │
  │  • ClickHouse: replicated tables                         │
  │  • Acceptable to lose a few click events                 │
  │                                                         │
  │  Key Generation:                                         │
  │  • Counter ranges persisted in ZooKeeper (Raft-backed)   │
  │  • Never reuse deleted short codes (soft delete)         │
  │  • Tombstone: keep mapping for 1 year after deletion     │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Redirect (cache hit)         │ 1 ms     │ 5 ms    │
  │ Redirect (cache miss)        │ 5 ms     │ 20 ms   │
  │ Create short URL             │ 10 ms    │ 50 ms   │
  │ Analytics query (7d)         │ 100 ms   │ 500 ms  │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • 301 (permanent) redirect: browser caches locally    │
  │  • CDN caching: 0ms for CDN-cached popular URLs        │
  │  • Redis: single GET for lookup (~0.5ms)                │
  │  • Connection pooling to DB (avoid connect overhead)   │
  │  • Async analytics: write click to Kafka, redirect     │
  │    immediately (don't wait for analytics write)         │
  │  • DNS: use Anycast for global low-latency routing     │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ DB primary down        │ Promote standby (30s failover); │
  │                        │ cache serves reads during gap   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Cache (Redis) down     │ Fallback to DB; higher latency  │
  │                        │ but still functional             │
  ├────────────────────────┼─────────────────────────────────┤
  │ Key collision          │ Check-and-retry: append counter │
  │                        │ and regenerate; max 3 retries   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Spam / abuse           │ Rate limit per IP + per user;   │
  │                        │ URL reputation check before     │
  │                        │ creating; Google Safe Browse API│
  ├────────────────────────┼─────────────────────────────────┤
  │ CDN outage             │ DNS fallback to origin servers  │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 14. Availability

```
  Target: 99.99% (short URLs must always resolve)

  ┌─────────────────────────────────────────────────────────┐
  │  Architecture:                                          │
  │  • Multi-AZ API servers (N+2 redundancy)                │
  │  • Multi-AZ database with sync standby                  │
  │  • Redis cluster across AZs                             │
  │  • CDN: global edge network (99.99% SLA)                │
  │                                                         │
  │  Graceful Degradation:                                  │
  │  • DB down → serve from cache (stale OK for redirects) │
  │  • Analytics pipeline down → redirect still works       │
  │  • URL creation down → redirects unaffected             │
  │                                                         │
  │  No Maintenance Downtime:                               │
  │  • Rolling deploys: zero-downtime                        │
  │  • DB migrations: online schema change (gh-ost)          │
  │  • Cache warming pre-deployment                          │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Abuse Prevention:                                      │
  │  • URL reputation check: Google Safe Browsing API       │
  │  • Phishing detection: scan destination on creation     │
  │  • Rate limiting: 100 creates/hour per user             │
  │  • CAPTCHA for anonymous URL creation                    │
  │  • Report abuse: flag URL for manual review              │
  │                                                         │
  │  Privacy:                                               │
  │  • Hash IP addresses in click analytics                  │
  │  • Password-protected URLs (bcrypt hash)                 │
  │  • Don't expose sequential IDs (use random codes)        │
  │  • GDPR: delete user's URLs and analytics on request    │
  │                                                         │
  │  Infrastructure:                                        │
  │  • HTTPS everywhere (HSTS headers)                       │
  │  • SQL injection: parameterized queries only             │
  │  • Input validation: reject malformed URLs               │
  │  • API keys for authenticated users                      │
  │  • Admin panel: 2FA, separate network                    │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Storage (very efficient):                              │
  │  • 1 Bn URL mappings × 200 bytes = 200 GB               │
  │  • PostgreSQL RDS (db.r6g.xl): ~$500/month              │
  │  • Click analytics (ClickHouse): ~$300/month            │
  │                                                         │
  │  Compute:                                               │
  │  • API servers (3 × c5.xl): ~$400/month                 │
  │  • Redis (r6g.lg, 50GB cache): ~$250/month              │
  │                                                         │
  │  CDN & Network:                                        │
  │  • CloudFront: $0.085/GB, 301 responses are tiny         │
  │  • 100K RPS × 500 bytes = ~$50/month transfer           │
  │                                                         │
  │  Total: ~$1.5K/month for 1Bn URLs, 100K RPS             │
  │                                                         │
  │  Managed Service Comparison:                            │
  │  • Bitly Business: $199/month (limited features)        │
  │  • At enterprise scale: self-hosted is cheaper           │
  │  • Key cost driver: CDN (solved by 301 browser caching) │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How to handle hash collisions?** — Check-and-retry with appended counter; or use pre-generated keys
2. **How to make URLs non-guessable?** — Don't use sequential IDs; use random base62 or crypto-random
3. **How to handle URL expiration at scale?** — TTL index in DB; lazy deletion (check on read) + periodic cleanup job
4. **Why not use SQL?** — No complex queries needed; key-value pattern; NoSQL scales better horizontally
5. **How to prevent abuse?** — Rate limiting per user/IP; CAPTCHA for anonymous; spam URL detection
