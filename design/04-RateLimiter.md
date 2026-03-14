# Design a Rate Limiter

Examples: API Gateway rate limiting, Cloudflare, Kong, Envoy

---

## 1. Requirements

### Functional
- Limit requests per client/API key to N requests per time window
- Support multiple rate limit rules (per user, per IP, per endpoint)
- Return `429 Too Many Requests` when limit exceeded
- Include rate limit headers in response (remaining, reset time)
- Support global (distributed) and local rate limiting

### Non-Functional
- Ultra-low latency (< 1ms overhead per request)
- High availability (rate limiter down should not block all traffic)
- Distributed (works across multiple API servers)
- Accurate (minimal over-counting or under-counting)

---

## 2. High-Level Architecture

```
                    ┌──────────────────┐
                    │    Client        │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  Load Balancer   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ API Srv 1│   │ API Srv 2│   │ API Srv 3│
        │ ┌──────┐ │   │ ┌──────┐ │   │ ┌──────┐ │
        │ │Rate  │ │   │ │Rate  │ │   │ │Rate  │ │
        │ │Limit │ │   │ │Limit │ │   │ │Limit │ │
        │ │Check │ │   │ │Check │ │   │ │Check │ │
        │ └──┬───┘ │   │ └──┬───┘ │   │ └──┬───┘ │
        └────┼─────┘   └────┼─────┘   └────┼─────┘
             │              │              │
             └──────────────┼──────────────┘
                            │
                   ┌────────▼────────┐
                   │  Redis Cluster   │
                   │  (shared state)  │
                   │  ┌────────────┐  │
                   │  │ Counters   │  │
                   │  │ per client │  │
                   │  └────────────┘  │
                   └─────────────────┘
```

**Flow:**
1. Request arrives at API server
2. Rate limiter middleware extracts client identifier (API key, IP, user ID)
3. Checks counter in Redis against configured limit
4. If under limit → increment counter, allow request
5. If over limit → return 429 with retry-after header

---

## 3. Rate Limiting Algorithms

### Token Bucket

```
  ┌─────────────────────────────────────────┐
  │          Token Bucket                    │
  │                                         │
  │  Bucket capacity: 10 tokens             │
  │  Refill rate: 2 tokens/second           │
  │                                         │
  │  ┌─────────────────────────┐            │
  │  │ [T] [T] [T] [T] [ ] [ ]│ ← bucket   │
  │  │  ▲                     │            │
  │  │  │ refill (2/sec)      │            │
  │  └──┼─────────────────────┘            │
  │     │                                   │
  │  Request arrives:                       │
  │  • Token available? → consume 1, ALLOW  │
  │  • No tokens? → REJECT (429)            │
  │                                         │
  │  Pros: Allows bursts up to bucket size  │
  │  Cons: Need to track last_refill_time   │
  └─────────────────────────────────────────┘

  Redis implementation (atomic Lua script):
  tokens = min(capacity, tokens + rate * (now - last_refill))
  if tokens >= 1:
      tokens -= 1
      return ALLOW
  else:
      return REJECT
```

### Sliding Window Log

```
  ┌─────────────────────────────────────────┐
  │        Sliding Window Log                │
  │                                         │
  │  Limit: 5 requests per 60 seconds       │
  │                                         │
  │  Store each request timestamp in a       │
  │  sorted set:                             │
  │                                         │
  │  Timeline: ──────────────────────────▶  │
  │            │  1:00  1:15  1:30  1:45 │   │
  │            │   ●     ●     ●     ●   │   │
  │            │       window (60s)       │   │
  │            └─────────────────────────┘   │
  │                                         │
  │  New request at 1:50:                   │
  │  1. Remove entries older than 0:50       │
  │  2. Count remaining: 4                   │
  │  3. 4 < 5 → ALLOW, add timestamp        │
  │                                         │
  │  Pros: Very precise                     │
  │  Cons: High memory (store all ts)       │
  └─────────────────────────────────────────┘
```

### Sliding Window Counter (Recommended)

```
  ┌─────────────────────────────────────────┐
  │      Sliding Window Counter              │
  │                                         │
  │  Limit: 100 requests per 60 seconds     │
  │  Current time = 1:15 (15s into window) │
  │                                         │
  │  ┌──────────────┬──────────────┐        │
  │  │ Prev Window  │ Curr Window  │        │
  │  │ (0:00-1:00)  │ (1:00-2:00)  │        │
  │  │  count = 80  │  count = 30  │        │
  │  └──────────────┴──────────────┘        │
  │                                         │
  │  Weighted count:                        │
  │  = prev × (1 - elapsed%) + curr         │
  │  = 80 × (1 - 15/60) + 30               │
  │  = 80 × 0.75 + 30                      │
  │  = 60 + 30 = 90                        │
  │                                         │
  │  90 < 100 → ALLOW                      │
  │                                         │
  │  Pros: Low memory (2 counters)          │
  │  Cons: Approximate but good enough      │
  └─────────────────────────────────────────┘
```

### Leaky Bucket

```
  ┌─────────────────────────────────────────┐
  │          Leaky Bucket                    │
  │                                         │
  │  Incoming requests fill the bucket       │
  │  Requests drain at a fixed rate          │
  │                                         │
  │      ┌──────┐                           │
  │      │ req  │ ← incoming (variable)     │
  │      │ req  │                           │
  │      │ req  │ bucket (FIFO queue)       │
  │      │ req  │                           │
  │      └──┬───┘                           │
  │         │                               │
  │         ▼  outflow = fixed rate         │
  │      ● ● ● (processed evenly)          │
  │                                         │
  │  Bucket full → reject new requests      │
  │                                         │
  │  Pros: Smooth output rate               │
  │  Cons: Bursts are queued, not served    │
  └─────────────────────────────────────────┘
```

---

## 4. Algorithm Comparison

| Algorithm | Memory | Burst | Precision | Best For |
|---|---|---|---|---|
| Token Bucket | Low | Allows controlled bursts | Good | API rate limiting |
| Leaky Bucket | Low (queue) | Smoothed output | Good | Traffic shaping |
| Sliding Window Log | High | Precise burst control | Exact | Small-scale, strict limits |
| Sliding Window Counter | Low | Approximate | ~99% accurate | Large-scale APIs |

**Recommendation:** Token Bucket for most API rate limiting. Sliding Window Counter when you need simplicity + low memory.

---

## 5. Distributed Rate Limiting

```
  Problem: Rate limit = 100/min per user
  User sends requests to 3 different API servers
  Each server has a local counter → user can do 300/min!

  Solution 1: Centralised (Redis)
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │ API Srv1│     │ API Srv2│     │ API Srv3│
  └────┬────┘     └────┬────┘     └────┬────┘
       │               │               │
       └───────────┬───┘───────────────┘
                   │
              ┌────▼────┐
              │  Redis   │  ← single source of truth
              │ INCR key │     per user counter
              └──────────┘

  Solution 2: Local + Sync
  • Each server gets local_limit = global_limit / num_servers
  • Periodically sync counts to central store
  • Trade-off: less precise but no Redis dependency per request

  Solution 3: Sticky Sessions
  • Load balancer routes same user to same server always
  • Each server rate limits locally
  • Downside: uneven load distribution
```

---

## 6. Redis Implementation (Token Bucket)

```lua
  -- Atomic Lua script for Redis
  local key = KEYS[1]
  local capacity = tonumber(ARGV[1])
  local rate = tonumber(ARGV[2])       -- tokens per second
  local now = tonumber(ARGV[3])
  local requested = tonumber(ARGV[4])

  local data = redis.call('HMGET', key, 'tokens', 'last_refill')
  local tokens = tonumber(data[1]) or capacity
  local last_refill = tonumber(data[2]) or now

  -- Refill tokens
  local elapsed = now - last_refill
  tokens = math.min(capacity, tokens + elapsed * rate)

  local allowed = tokens >= requested
  if allowed then
      tokens = tokens - requested
  end

  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
  redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)

  return allowed and 1 or 0
```

---

## 7. Rate Limit Rules Engine

```
  Rules configuration:
  ┌─────────────────────────────────────────────────┐
  │ Rule 1: Per API key                              │
  │   key_pattern: "rl:{api_key}"                    │
  │   limit: 1000 req/min                            │
  │                                                  │
  │ Rule 2: Per user per endpoint                    │
  │   key_pattern: "rl:{user_id}:{endpoint}"         │
  │   limit: 100 req/min                             │
  │                                                  │
  │ Rule 3: Global per endpoint                      │
  │   key_pattern: "rl:global:{endpoint}"            │
  │   limit: 50000 req/min                           │
  │                                                  │
  │ Rule 4: Per IP (DDoS protection)                 │
  │   key_pattern: "rl:ip:{client_ip}"               │
  │   limit: 50 req/sec                              │
  └─────────────────────────────────────────────────┘

  Evaluation order: most specific → least specific
  First rule that rejects → return 429
```

---

## 8. Response Headers

```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710288000
Retry-After: 30
```

---

## 9. Fault Tolerance

- **Redis down:** Fail open (allow requests) or fail closed (reject). Usually fail open to avoid blocking legitimate traffic
- **Clock skew:** Use Redis server time (`TIME` command) rather than client time
- **Race conditions:** Lua scripts ensure atomicity; `MULTI/EXEC` for simple cases
- **Monitoring:** Track rate limit hits/rejections per client, alert on spikes

---

## 10. Low-Level Design (LLD)

### API Contract
```
  Rate Limit Check (internal middleware):
  POST   /internal/ratelimit/check
         Body: { "client_id": str, "resource": str, "tokens_requested": 1 }
         Response: { "allowed": bool, "remaining": int, "reset_at": epoch,
                     "retry_after_ms": int }

  Rate Limit Configuration (admin):
  PUT    /admin/v1/ratelimit/rules/{rule_id}
         Body: { "match": { "client_type": "api_key", "pattern": "*" },
                 "limit": 1000, "window_seconds": 60,
                 "algorithm": "token_bucket", "burst": 50 }
  GET    /admin/v1/ratelimit/rules
  DELETE /admin/v1/ratelimit/rules/{rule_id}

  Metrics:
  GET    /admin/v1/ratelimit/stats?client_id=X&window=1h
         Response: { "total_requests": int, "allowed": int, "rejected": int }
```

### Rate Limiter Middleware Flow
```
  Request → Extract client ID (API key / IP / user ID)
          → Build rate limit key: "rl:{rule}:{client_id}"
          → Execute Lua script on Redis (atomic)
          → If allowed: set response headers, proceed
          → If rejected: return 429 + headers

  Headers set on EVERY response:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1710288000  (epoch seconds)
```

### Data Model in Redis
```
  Token Bucket:
    HASH  "rl:tb:{client}:{resource}"
      tokens      → float
      last_refill → epoch_ms

  Sliding Window Counter:
    STRING "rl:sw:{client}:{resource}:{window_start}"  → count
    TTL = 2 × window_size

  Sliding Window Log:
    ZSET "rl:swl:{client}:{resource}"
      member = request_id, score = timestamp
    ZREMRANGEBYSCORE to remove expired entries
```

---

## 11. Scalability

```
  ┌────────────────────────────────────────────────────────┐
  │  Single Redis: ~200K rate-limit checks/sec             │
  │  Need higher? Scale horizontally:                      │
  │                                                        │
  │  Option A: Sharded Redis                               │
  │  • Shard by hash(client_id) % N shards                 │
  │  • Each shard handles subset of clients                │
  │  • 4 shards → 800K checks/sec                         │
  │                                                        │
  │  Option B: Local + Global Hybrid                       │
  │  • Each API server: local in-memory counter            │
  │  • Sync to Redis every 1s (batched)                    │
  │  • Local limit = global_limit / num_servers            │
  │  • Accuracy: ±10% (acceptable for most use cases)     │
  │                                                        │
  │  Option C: Cell-based Architecture                     │
  │  • Each cell (region) has its own rate limiter         │
  │  • Global aggregation via async gossip protocol        │
  │  • Each cell gets proportional share of global limit   │
  │                                                        │
  │  Scaling Rules Engine:                                  │
  │  • Cache compiled rules in-memory per API server       │
  │  • Push rule updates via pub/sub (not poll)            │
  │  • Hot-reload without restart                          │
  └────────────────────────────────────────────────────────┘
```

---

## 12. No Data Loss

```
  Rate limiter counters are ephemeral — "data loss" means:
  ┌─────────────────────────────────────────────────────────┐
  │  Risk 1: Redis restart → counters reset                 │
  │  Impact: Clients get a brief burst above limits         │
  │  Mitigation: Redis persistence (RDB every 1min);       │
  │    acceptable because counters reset naturally by TTL   │
  │                                                         │
  │  Risk 2: Counters lost during failover                  │
  │  Impact: Brief over-limit allowance (~1 window)         │
  │  Mitigation: Redis Sentinel auto-failover with         │
  │    AOF persistence on replicas                          │
  │                                                         │
  │  Audit Trail (for compliance):                          │
  │  • Log all 429 rejections to audit store                │
  │  • Async write to Kafka → long-term analytics DB       │
  │  • Never lose rejection records (acks=all to Kafka)    │
  │                                                         │
  │  Rate limit configs:                                    │
  │  • Stored in durable DB (PostgreSQL)                    │
  │  • Version-controlled config files (GitOps)             │
  │  • Cached in Redis for runtime performance              │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Latency

```
  Target: < 1ms overhead per rate-limit check

  Latency Breakdown:
  ┌────────────────────────┬──────────┐
  │ Component              │ Time     │
  ├────────────────────────┼──────────┤
  │ Client ID extraction   │ 0.01 ms  │
  │ Rule lookup (in-memory)│ 0.01 ms  │
  │ Redis Lua eval         │ 0.1-0.5ms│
  │ Response header set    │ 0.01 ms  │
  │ Total overhead         │ < 0.5 ms │
  └────────────────────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Connection pooling to Redis (avoid TCP per request) │
  │  • Lua script caching (EVALSHA vs EVAL)                │
  │  • Pipelining: batch multiple limit checks in one RTT  │
  │  • Local in-process cache: skip Redis for warm clients │
  │    (sync every 100ms) — reduces Redis calls by 90%+   │
  │  • Use Unix domain socket if Redis is co-located       │
  │  • Async logging (don't block request for audit trail) │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Reliability

```
  Failure Modes & Behavior:
  ┌────────────────────────┬───────────────────────────────┐
  │ Failure                │ Behavior                      │
  ├────────────────────────┼───────────────────────────────┤
  │ Redis unavailable      │ Fail open: allow all requests │
  │                        │ (protect availability over    │
  │                        │ rate limiting accuracy)        │
  ├────────────────────────┼───────────────────────────────┤
  │ Redis slow (> 5ms)     │ Timeout + fail open; switch   │
  │                        │ to local counters temporarily │
  ├────────────────────────┼───────────────────────────────┤
  │ API server restart     │ Local counters lost; Redis    │
  │                        │ counters still accurate        │
  ├────────────────────────┼───────────────────────────────┤
  │ DDoS attack            │ Progressive rate limiting:    │
  │                        │ IP → subnet → geo block       │
  ├────────────────────────┼───────────────────────────────┤
  │ Config error           │ Canary deployment of rules;   │
  │ (too restrictive)      │ rollback on high rejection %  │
  └────────────────────────┴───────────────────────────────┘

  Circuit Breaker on Redis:
  • 5 consecutive failures → open circuit (fail open)
  • Half-open after 10s: try single request
  • If success → close circuit, resume Redis checks
```

---

## 15. Availability

```
  Target: 99.999% (rate limiter should never be a SPOF)

  Architecture:
  ┌─────────────────────────────────────────────────────────┐
  │  Redis: master + 2 replicas across 3 AZs               │
  │  Sentinel: 3 instances for auto-failover                │
  │  Failover time: < 5 seconds                             │
  │                                                         │
  │  Graceful Degradation Hierarchy:                        │
  │  1. Redis available → accurate distributed rate limit   │
  │  2. Redis down → local in-memory counters (approximate) │
  │  3. All counters fail → fail open (allow all traffic)   │
  │  4. Under extreme DDoS → upstream CDN / WAF protection │
  │                                                         │
  │  Rate limiter MUST NOT block legitimate traffic:        │
  │  • No request should fail because the rate limiter      │
  │    itself is unavailable                                │
  │  • Always prefer allowing over-limit traffic to         │
  │    blocking legitimate requests                         │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Preventing Bypass:                                     │
  │  • Client ID must be unforgeable (signed token, not     │
  │    user-supplied header)                                │
  │  • IP-based limits: handle X-Forwarded-For spoofing     │
  │    → trust only from known proxies/LBs                  │
  │  • API key rotation without rate limit reset             │
  │                                                         │
  │  DDoS Protection:                                       │
  │  • Layer 7: rate limiter at API gateway                  │
  │  • Layer 4: TCP SYN flood protection at LB/CDN          │
  │  • Geo-blocking for known bad regions                    │
  │  • CAPTCHA challenge for suspicious clients              │
  │                                                         │
  │  Admin Security:                                         │
  │  • Rate limit rule changes require admin role + 2FA     │
  │  • Audit log for all rule changes                       │
  │  • Canary deployment for rule changes                    │
  │                                                         │
  │  Privacy:                                                │
  │  • Hash client IPs in rate limit keys                    │
  │  • TTL on all rate limit data (auto-cleanup)             │
  │  • Don't log full request bodies in rejection records   │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Rate limiter is very cost-efficient:                   │
  │                                                         │
  │  Redis Memory:                                          │
  │  • Token bucket: ~100 bytes per client per rule         │
  │  • 10M unique clients × 5 rules = 5 GB Redis           │
  │  • Single r6g.xlarge (32 GB): ~$200/month               │
  │                                                         │
  │  Optimization:                                          │
  │  • Short TTLs: counters expire naturally (no cleanup)   │
  │  • Local + sync: reduce Redis calls by 90%             │
  │  • Pipeline batch checks: fewer network round trips    │
  │                                                         │
  │  Cost vs Benefit:                                       │
  │  • Rate limiter cost: ~$500/month                       │
  │  • Prevents: DDoS damage, abuse, overprovisioning      │
  │  • Without it: need 10× more backend capacity for      │
  │    peak/abuse scenarios = $50K+ saved/month            │
  │                                                         │
  │  Managed alternatives:                                  │
  │  • AWS API Gateway: $3.50 per million requests          │
  │  • Cloudflare Rate Limiting: $0.05 per 10K good reqs  │
  │  • Build vs Buy: build when > 100M requests/month      │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Where to place the rate limiter?** — Middleware in API server, API Gateway (Kong/Envoy), or dedicated service
2. **What if Redis is a bottleneck?** — Local counters + periodic sync, or shard across multiple Redis instances
3. **How to handle multi-tier limits?** — Apply all rules; reject on first failure. 100/min per user AND 10K/min global
4. **How to rate limit in a microservices architecture?** — At the API gateway layer for external; service mesh (Envoy) for internal
5. **Token bucket vs sliding window?** — Token bucket allows bursts (good for user experience); sliding window is more predictable
