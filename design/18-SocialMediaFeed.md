# Design a Social Media Feed System

Examples: Twitter/X Timeline, Facebook News Feed, Instagram Feed

---

## 1. Requirements

### Functional
- Users follow other users
- Users create posts (text, images, links)
- Home feed shows posts from followed users, ranked
- Support likes, retweets, and comments
- Real-time feed updates for new posts

### Non-Functional
- 300M DAU, 500M tweets/day
- Feed generation < 200ms
- Eventually consistent (slight delay in feed is OK)
- Handle celebrity users (millions of followers)

---

## 2. Scale Estimation

```
DAU:               300M
Posts/day:          500M → ~6K posts/sec
Feed reads/day:    300M × 10 opens = 3B → ~35K reads/sec
Average follows:   200 users
Feed size:         200 posts per timeline
Storage/post:      ~1 KB (text + metadata)
Posts/year:        500M × 365 × 1 KB = ~180 TB
```

---

## 3. High-Level Architecture

```
  ┌──────────┐     ┌──────────────┐     ┌──────────────────┐
  │  Client   │────▶│  API Gateway │────▶│  Post Service    │
  │           │     │              │     │  (write path)    │
  └──────────┘     └──────────────┘     └────────┬─────────┘
                                                  │
                          ┌───────────────────────┼──────────────┐
                          │                       │              │
                   ┌──────▼──────┐         ┌──────▼──────┐       │
                   │  Post DB    │         │  Fan-out    │       │
                   │  (posts     │         │  Service    │       │
                   │   store)    │         └──────┬──────┘       │
                   └─────────────┘                │              │
                                           ┌──────▼──────┐       │
                                           │  Timeline   │       │
                                           │  Cache      │       │
                                           │  (Redis)    │       │
                                           └──────┬──────┘       │
                                                  │              │
  ┌──────────┐     ┌──────────────┐        ┌──────▼──────┐       │
  │  Client   │────▶│  Feed Service│───────▶│  Timeline   │       │
  │  (read)   │     │  (read path) │       │  Builder    │       │
  └──────────┘     └──────────────┘        └─────────────┘       │
                                                                 │
                                           ┌─────────────┐       │
                                           │  Ranking    │◀──────┘
                                           │  Service    │
                                           └─────────────┘
```

---

## 4. Fan-out Strategy

```
  Two approaches:

  FAN-OUT ON WRITE (Push model):
  ┌──────────────────────────────────────────────┐
  │  User A posts a tweet                        │
  │  A has 500 followers                         │
  │                                              │
  │  Fan-out service:                            │
  │  For each follower:                          │
  │    LPUSH timeline:{follower_id} tweet_id     │
  │    LTRIM timeline:{follower_id} 0 799        │
  │                                              │
  │  500 Redis writes per tweet                  │
  │                                              │
  │  Pro: Feed reads are instant (pre-computed)  │
  │  Con: Slow for celebrities (millions writes) │
  │       Wasteful if follower never reads       │
  └──────────────────────────────────────────────┘

  FAN-OUT ON READ (Pull model):
  ┌──────────────────────────────────────────────┐
  │  User B opens feed                           │
  │  B follows 200 users                         │
  │                                              │
  │  Feed service:                               │
  │  For each followed user:                     │
  │    GET latest posts from post DB             │
  │  Merge + sort by timestamp/rank              │
  │  Return top 200                              │
  │                                              │
  │  200 DB queries per feed open                │
  │                                              │
  │  Pro: No wasted writes; works for celebrities│
  │  Con: Slow feed reads (merge at read time)   │
  └──────────────────────────────────────────────┘

  HYBRID (Twitter's actual approach):
  ┌──────────────────────────────────────────────┐
  │  Regular users (< 10K followers):            │
  │    → Fan-out on WRITE                        │
  │    → Pre-compute timelines in Redis          │
  │                                              │
  │  Celebrity users (> 10K followers):           │
  │    → Fan-out on READ                         │
  │    → Fetch celebrity tweets at read time      │
  │    → Merge with pre-computed timeline         │
  │                                              │
  │         Pre-computed       Celebrity tweets  │
  │         timeline    +      (fetched at read) │
  │         (Redis)            (merge + rank)    │
  │             │                     │          │
  │             └─────────┬───────────┘          │
  │                       ▼                      │
  │                 Final Feed                   │
  └──────────────────────────────────────────────┘
```

---

## 5. Data Model

```
  Users table:
  ┌──────────────────────────────────────┐
  │  user_id  (PK)     │ BIGINT         │
  │  username           │ VARCHAR        │
  │  follower_count     │ INT            │
  │  is_celebrity       │ BOOLEAN        │
  └──────────────────────────────────────┘

  Posts table (sharded by user_id):
  ┌──────────────────────────────────────┐
  │  post_id  (PK)     │ SNOWFLAKE_ID   │
  │  user_id            │ BIGINT         │
  │  content            │ TEXT           │
  │  media_urls         │ JSON           │
  │  created_at         │ TIMESTAMP      │
  │  like_count         │ INT            │
  │  retweet_count      │ INT            │
  └──────────────────────────────────────┘

  Follows table (Graph DB or adjacency list):
  ┌──────────────────────────────────────┐
  │  follower_id        │ BIGINT         │
  │  followee_id        │ BIGINT         │
  │  created_at         │ TIMESTAMP      │
  └──────────────────────────────────────┘

  Timeline cache (Redis sorted set):
  ┌──────────────────────────────────────┐
  │  Key: timeline:{user_id}             │
  │  Members: post_ids                   │
  │  Score: timestamp (or rank score)    │
  │  Max size: 800 entries               │
  └──────────────────────────────────────┘
```

---

## 6. Ranking

```
  Simple chronological:
    score = created_at

  ML-based ranking (modern feeds):
  ┌──────────────────────────────────────────────┐
  │  Features:                                   │
  │  • Recency (time decay)                      │
  │  • Engagement (likes, retweets, comments)    │
  │  • Author affinity (how often you interact)  │
  │  • Content type (image, video, text)         │
  │  • Social proof (friends liked it)           │
  │                                              │
  │  Model: lightweight ranker (logistic reg     │
  │         or small neural network)             │
  │  Latency budget: < 50ms for 200 candidates   │
  │                                              │
  │  Ranking pipeline:                           │
  │  1. Candidate generation (1000 posts)        │
  │  2. Feature extraction                       │
  │  3. Scoring (ML model)                       │
  │  4. Re-ranking (diversity, dedup)            │
  │  5. Return top 200                           │
  └──────────────────────────────────────────────┘
```

---

## 7. Fault Tolerance

- **Redis timeline cache loss:** Rebuild from post DB (fan-out on read fallback)
- **Post DB failure:** Read replicas serve reads; promote on primary failure
- **Fan-out service lag:** Users see slightly stale feed; catch up asynchronously
- **Celebrity post burst:** Dedicated queue for celebrity fan-out; rate limit processing

---

---

## 8. Low-Level Design (LLD)

### API Contract
```
  Feed:
  GET    /api/v1/feed?cursor=abc&limit=20
         Response: { "posts": [
           { "post_id": "123", "author": {...}, "content": "...",
             "media": ["url1"], "likes": 542, "comments": 23,
             "created_at": "...", "ranking_score": 0.95 }
         ], "next_cursor": "def" }

  Post:
  POST   /api/v1/posts
         { "content": "Hello world", "media_ids": ["m1","m2"],
           "visibility": "public", "reply_to": null }
  DELETE /api/v1/posts/{id}

  Social Graph:
  POST   /api/v1/follow    { "user_id": "456" }
  DELETE /api/v1/unfollow  { "user_id": "456" }
  GET    /api/v1/followers?user_id=456&cursor=...

  Engagement:
  POST   /api/v1/posts/{id}/like
  POST   /api/v1/posts/{id}/comment  { "text": "..." }
  POST   /api/v1/posts/{id}/share
```

### Data Model (Cassandra + Redis)
```
  Posts Table (Cassandra):
  ┌────────────────────────────────────────────────────────┐
  │  PK: (user_id, post_id)    — partition by author       │
  │  post_id     TIMEUUID      — time-ordered              │
  │  content     TEXT                                      │
  │  media_urls  LIST<TEXT>                                │
  │  visibility  TEXT           — public/friends/private    │
  │  like_count  COUNTER       — denormalized              │
  │  created_at  TIMESTAMP                                 │
  └────────────────────────────────────────────────────────┘

  Timeline Cache (Redis Sorted Set):
  ┌────────────────────────────────────────────────────────┐
  │  Key: timeline:{user_id}                               │
  │  Score: ranking_score (or timestamp for chronological) │
  │  Value: post_id                                        │
  │  Max size: 800 entries (trim oldest)                   │
  │                                                        │
  │  ZADD timeline:user_123 0.95 post_456                  │
  │  ZREVRANGE timeline:user_123 0 19  → top 20 posts     │
  └────────────────────────────────────────────────────────┘

  Social Graph (Adjacency List in Cassandra):
  ┌────────────────────────────────────────────────────────┐
  │  followers: PK=(user_id), CK=(follower_id)            │
  │  following: PK=(user_id), CK=(followee_id)            │
  │  Both tables updated atomically with BATCH             │
  └────────────────────────────────────────────────────────┘

  Fan-out Worker:
  ┌────────────────────────────────────────────────────────┐
  │  On new post by user X:                                │
  │  1. Read X's followers (paginated, 1000 at a time)    │
  │  2. For each follower:                                 │
  │     - ZADD timeline:{follower_id} score post_id       │
  │     - ZREMRANGEBYRANK to trim if > 800                 │
  │  3. If X has > 10K followers (celebrity):              │
  │     - Skip fan-out, mix in at read-time                │
  │     - Mark post in "celebrity_posts" index             │
  └────────────────────────────────────────────────────────┘
```

---

## 9. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Feed Read Path:                                        │
  │  • Timeline cache (Redis): 500K reads/sec               │
  │  • Cache miss: rebuild from following list + post DB    │
  │  • Celebrity posts: merge at read time from top-N index│
  │                                                         │
  │  Post Write Path:                                       │
  │  • New post → Kafka topic → fan-out workers             │
  │  • Workers: auto-scale based on Kafka consumer lag      │
  │  • Celebrity exemption: no fan-out for > 10K followers  │
  │                                                         │
  │  Social Graph:                                          │
  │  • 500M users × avg 200 following = 100Bn edges         │
  │  • Cassandra: partition by user_id, shard horizontally  │
  │                                                         │
  │  Data Volume:                                           │
  │  • 500M posts/day × 1KB = 500 GB/day                    │
  │  • Timeline cache: 500M users × 800 post IDs × 16B    │
  │    = ~6.4 TB Redis (sharded cluster)                    │
  │                                                         │
  │  ML Ranking:                                            │
  │  • Feature store (Redis): pre-computed user features    │
  │  • Ranking model: served via TensorFlow Serving         │
  │  • Batch scoring for timeline rebuild on cache miss     │
  └─────────────────────────────────────────────────────────┘
```

---

## 10. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Posts (user-generated content — critical):              │
  │  • Write to Cassandra (RF=3, CL=QUORUM)                │
  │  • Multi-DC replication for DR                          │
  │  • Soft delete: retain for 30 days                      │
  │                                                         │
  │  Timeline Cache (Redis — ephemeral, rebuildable):       │
  │  • OK to lose: rebuild from post DB + social graph     │
  │  • Redis AOF for persistence (async, best-effort)      │
  │  • Cache miss → full timeline rebuild (expensive but OK)│
  │                                                         │
  │  Fan-out Events:                                        │
  │  • Kafka: acks=all, replication factor 3                │
  │  • Consumer offsets: committed after timeline updated   │
  │  • If fan-out worker crashes: resume from last offset   │
  │                                                         │
  │  Media (images, videos):                                │
  │  • Stored in S3 (11 nines durability)                    │
  │  • Post references media by URL (not embedded)          │
  │  • Orphan cleanup: scheduled job removes unreferenced   │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Feed load (cache hit)        │ 20 ms    │ 100 ms  │
  │ Feed load (cache miss)       │ 200 ms   │ 1 s     │
  │ Create post                  │ 50 ms    │ 200 ms  │
  │ Like / comment               │ 20 ms    │ 100 ms  │
  │ Fan-out completion (non-celeb)│ 5 s     │ 30 s    │
  │ Image upload                 │ 500 ms   │ 3 s     │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Pre-computed timeline cache: one Redis read per feed│
  │  • Pagination with cursor (no offset counting)         │
  │  • CDN for media: images served from nearest PoP       │
  │  • Denormalized counts: like/comment counts in post    │
  │  • Async engagement: like is fire-and-forget to Kafka  │
  │  • Batch fan-out: group 1000 followers per worker task │
  │  • Prefetch: ML predicts and pre-ranks next page       │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Redis cache failure    │ Rebuild timeline on demand from │
  │                        │ post DB; higher latency but     │
  │                        │ functional                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Kafka consumer lag     │ Auto-scale workers; alert if    │
  │                        │ fan-out delay > 5 min            │
  ├────────────────────────┼─────────────────────────────────┤
  │ Celebrity post storm   │ Exempt from fan-out; merge at   │
  │                        │ read time; circuit breaker on   │
  │                        │ fan-out worker queue             │
  ├────────────────────────┼─────────────────────────────────┤
  │ Ranking model crash    │ Fallback to chronological order │
  │                        │ (degraded but functional)        │
  ├────────────────────────┼─────────────────────────────────┤
  │ Post DB partition      │ Cassandra CL=LOCAL_QUORUM;      │
  │                        │ reads from local DC continue    │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 13. Availability

```
  Target: 99.99% for feed reads; 99.9% for post creation

  ┌─────────────────────────────────────────────────────────┐
  │  Feed Read (highest priority):                          │
  │  • Redis cluster: 6 nodes × 3 AZs                      │
  │  • Cassandra: multi-DC, LOCAL_QUORUM reads              │
  │  • CDN for media (99.99% SLA)                           │
  │  • If Redis down: serve from DB (slower but works)      │
  │                                                         │
  │  Post Write:                                            │
  │  • Cassandra: write to any node (QUORUM consistency)    │
  │  • Kafka: 3 brokers × 3 AZs for fan-out pipeline       │
  │  • If fan-out delayed: post visible to author instantly │
  │    (async fan-out catches up)                            │
  │                                                         │
  │  Graceful Degradation:                                  │
  │  • Ranking model down → chronological feed              │
  │  • Media CDN down → serve from origin (slow)            │
  │  • Fan-out backlog → feed slightly stale (tolerable)   │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Content Safety:                                        │
  │  • Image/video moderation: ML model scans uploads      │
  │  • Text content: profanity filter + hate speech detect │
  │  • Report system: user reports → human review queue     │
  │  • CSAM detection: PhotoDNA hash matching               │
  │                                                         │
  │  Privacy:                                               │
  │  • Visibility controls: public/friends/private          │
  │  • Block/mute: excluded from feed and interactions     │
  │  • GDPR: export all data, delete on request             │
  │  • Location data: opt-in only, coarsened                │
  │                                                         │
  │  Authentication:                                        │
  │  • OAuth2 + JWT for API access                          │
  │  • 2FA for account changes                              │
  │  • Session management: rotate tokens, geo-anomaly detect│
  │                                                         │
  │  Anti-Abuse:                                            │
  │  • Rate limiting: posts/min, likes/min per user         │
  │  • Bot detection: CAPTCHA, device fingerprinting        │
  │  • Spam: ML classifier on post content                  │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  At Scale (500M users, 100M DAU):                       │
  │                                                         │
  │  Timeline Cache (Redis):                                │
  │  • 6.4 TB → ElastiCache (20 × r6g.4xl): ~$40K/month   │
  │                                                         │
  │  Post Storage (Cassandra):                              │
  │  • 500 GB/day × 90d retention = 45 TB                  │
  │  • 30 nodes (i3.2xl): ~$30K/month                       │
  │                                                         │
  │  Fan-out Workers + Kafka:                               │
  │  • Kafka (6 brokers): ~$5K/month                        │
  │  • Workers (auto-scale, avg 20): ~$3K/month            │
  │                                                         │
  │  Media CDN (images/videos):                             │
  │  • 50 TB/day transfer: ~$100K/month (largest cost!)    │
  │                                                         │
  │  Total: ~$180K/month                                    │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Trim timeline cache to 500 posts (vs 800)            │
  │  • Celebrity hybrid: saves 90% of fan-out writes        │
  │  • Image optimization: WebP/AVIF → 30% bandwidth save │
  │  • Cold storage: archive posts > 1 year to S3           │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Why hybrid fan-out?** — Pure push doesn't scale for celebrities (1M writes per tweet). Pure pull is slow for feed reads. Hybrid balances both.
2. **How to handle unfollow?** — Remove from timeline cache (lazy: old tweets naturally age out)
3. **How to show real-time updates?** — WebSocket for new tweet notification; client polls or long-polls for feed refresh
4. **How to shard posts?** — By user_id (all posts for a user on same shard); use Snowflake IDs for global ordering
5. **Timeline cache vs computation?** — Cache for read-heavy workload; recompute on cache miss; TTL to prevent stale data
