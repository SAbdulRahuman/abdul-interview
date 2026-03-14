# Design an API Gateway

Examples: Kong, Envoy, AWS API Gateway, Nginx

---

## 1. Requirements

### Functional
- Route requests to backend services based on path/host
- Authentication and authorization (JWT, API keys, OAuth2)
- Rate limiting per client/endpoint
- Request/response transformation
- Load balancing across service instances
- API versioning

### Non-Functional
- Ultra-low added latency (< 5ms overhead)
- High throughput (100K+ requests/sec per node)
- High availability (no single point of failure)
- Extensible via plugins/middleware

---

## 2. High-Level Architecture

```
  ┌────────────┐   ┌────────────┐   ┌────────────┐
  │ Mobile App │   │ Web Client │   │ Partner API│
  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
        │               │               │
        └───────────────┬┘───────────────┘
                        │
               ┌────────▼────────┐
               │  Load Balancer   │
               └────────┬────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
  ┌─────────────┐┌─────────────┐┌─────────────┐
  │ API Gateway ││ API Gateway ││ API Gateway │
  │  Node 1     ││  Node 2     ││  Node 3     │
  │             ││             ││             │
  │ ┌─────────┐ ││ ┌─────────┐ ││ ┌─────────┐ │
  │ │Plugins: │ ││ │Plugins: │ ││ │Plugins: │ │
  │ │• Auth   │ ││ │• Auth   │ ││ │• Auth   │ │
  │ │• Rate   │ ││ │• Rate   │ ││ │• Rate   │ │
  │ │• Log    │ ││ │• Log    │ ││ │• Log    │ │
  │ │• Transform││ │• Transform││ │• Transform│
  │ └─────────┘ ││ └─────────┘ ││ └─────────┘ │
  └──────┬──────┘└──────┬──────┘└──────┬──────┘
         │              │              │
         └──────────────┼──────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
  ┌────────────┐ ┌────────────┐ ┌────────────┐
  │ User Svc   │ │ Order Svc  │ │ Payment Svc│
  └────────────┘ └────────────┘ └────────────┘
```

---

## 3. Request Processing Pipeline

```
  Incoming Request
       │
  ┌────▼─────────────┐
  │  1. TLS Termination│
  └────┬─────────────┘
       │
  ┌────▼─────────────┐
  │  2. Authentication │  ← Validate JWT / API key
  └────┬─────────────┘    ← Reject 401 if invalid
       │
  ┌────▼─────────────┐
  │  3. Rate Limiting  │  ← Check per-client limits
  └────┬─────────────┘    ← Reject 429 if exceeded
       │
  ┌────▼─────────────┐
  │  4. Request       │  ← Add headers, rewrite path
  │     Transformation│    ← Request validation
  └────┬─────────────┘
       │
  ┌────▼─────────────┐
  │  5. Routing       │  ← Match path/host → backend
  └────┬─────────────┘    ← /api/v2/users → user-svc
       │
  ┌────▼─────────────┐
  │  6. Load Balance  │  ← Round-robin / least-conn
  └────┬─────────────┘    ← Health-checked backends
       │
  ┌────▼─────────────┐
  │  7. Proxy to      │  ← Forward request to backend
  │     Backend        │
  └────┬─────────────┘
       │
  ┌────▼─────────────┐
  │  8. Response      │  ← Transform response
  │     Transformation│    ← Add CORS headers
  └────┬─────────────┘
       │
  ┌────▼─────────────┐
  │  9. Logging &     │  ← Emit access log, metrics
  │     Metrics       │    ← Distributed trace span
  └────┬─────────────┘
       │
  Return to Client
```

---

## 4. Routing Configuration

```
  Route table:
  ┌──────────────────────────────────────────────────┐
  │  Host: api.example.com                           │
  │                                                  │
  │  /api/v1/users/*    → user-service:8080          │
  │  /api/v1/orders/*   → order-service:8080         │
  │  /api/v1/payments/* → payment-service:8080       │
  │  /api/v2/users/*    → user-service-v2:8080       │
  │                                                  │
  │  Host: admin.example.com                         │
  │  /*                 → admin-service:8080          │
  │                                                  │
  │  Canary routing:                                 │
  │  /api/v1/users/* (10% traffic) → user-svc-canary │
  │  /api/v1/users/* (90% traffic) → user-svc-stable │
  └──────────────────────────────────────────────────┘

  Service discovery integration:
  • Poll Kubernetes API for service endpoints
  • Watch Consul/etcd for service registration
  • DNS-based resolution with SRV records
```

---

## 5. Key Features Detail

```
  Circuit Breaker:
  ┌──────────────────────────────────────────────┐
  │  CLOSED → errors > 50% in 10s → OPEN        │
  │  OPEN → reject all requests for 30s          │
  │  HALF-OPEN → allow 1 probe request           │
  │  If probe succeeds → CLOSED                  │
  │  If probe fails → OPEN again                 │
  └──────────────────────────────────────────────┘

  Retry with backoff:
  ┌──────────────────────────────────────────────┐
  │  On 5xx or timeout:                          │
  │  Attempt 1: immediate                        │
  │  Attempt 2: wait 100ms                       │
  │  Attempt 3: wait 400ms                       │
  │  Give up after 3 attempts → return 502       │
  │  Only retry on idempotent methods (GET)       │
  └──────────────────────────────────────────────┘

  Request timeout:
  ┌──────────────────────────────────────────────┐
  │  Connection timeout: 5s                      │
  │  Read timeout: 30s (per backend)             │
  │  Global timeout: 60s (including retries)     │
  └──────────────────────────────────────────────┘
```

---

## 6. Scaling Strategy

- **Stateless:** API gateway nodes share no state → scale horizontally behind LB
- **Config sync:** Config stored in DB (PostgreSQL) or etcd; nodes poll or watch for updates
- **Connection pooling:** Reuse backend connections to reduce overhead
- **Caching:** Cache responses for GET endpoints at gateway level (with TTL)
- **Regional deployment:** Gateway per region for latency

---

---

## 7. Low-Level Design (LLD)

### API Contract (Gateway Management)
```
  Route Configuration:
  POST   /admin/routes
         { "path": "/api/v1/users/*", "backend": "user-service:8080",
           "methods": ["GET","POST"], "plugins": ["rate-limit","jwt-auth"],
           "timeout_ms": 3000, "retries": 2 }

  Plugin Configuration:
  POST   /admin/plugins
         { "name": "rate-limit", "config": { "rate": 100, "per": "second",
           "policy": "sliding-window", "key": "consumer_id" } }

  Consumer Registration:
  POST   /admin/consumers
         { "username": "mobile-app", "groups": ["external"],
           "credentials": { "type": "api-key", "key": "abc123..." } }

  Health / Metrics:
  GET    /admin/health          → { "status": "healthy", "db": "ok" }
  GET    /admin/metrics         → Prometheus-format metrics
```

### Request Processing Internals
```
  Plugin Chain (OpenResty / Envoy Filter Chain):
  ┌────────────────────────────────────────────────────────┐
  │  Phases (per request):                                 │
  │                                                        │
  │  1. ACCESS phase:                                      │
  │     ├─ IP whitelist / blacklist                        │
  │     ├─ mTLS client cert validation                     │
  │     ├─ JWT / OAuth2 token validation                   │
  │     ├─ API key lookup (in-memory hash map)             │
  │     └─ Rate limiting (Redis EVALSHA)                   │
  │                                                        │
  │  2. REWRITE phase:                                     │
  │     ├─ Path rewrite (/v1/users → /users)               │
  │     ├─ Header injection (X-Request-ID, trace headers)  │
  │     ├─ Request transformation (REST → gRPC)            │
  │     └─ Schema validation (JSON Schema / protobuf)      │
  │                                                        │
  │  3. PROXY phase:                                       │
  │     ├─ Upstream selection (round-robin / weighted)      │
  │     ├─ Circuit breaker check                           │
  │     ├─ Connection pooling (persistent connections)     │
  │     └─ Request forwarded to backend                    │
  │                                                        │
  │  4. RESPONSE phase:                                    │
  │     ├─ Response transformation                         │
  │     ├─ Compression (gzip/brotli)                       │
  │     ├─ Response caching (if cacheable)                 │
  │     └─ Logging (async, buffered)                       │
  └────────────────────────────────────────────────────────┘
```

### Routing Table
```
  Data Structure (in-memory):
  ┌────────────────────────────────────────────────────────┐
  │  Radix tree / trie for path matching:                  │
  │                                                        │
  │  /api/v1/                                              │
  │  ├── users/                                            │
  │  │   ├── {id}  → user-service (GET,PUT,DELETE)         │
  │  │   └── *     → user-service (POST)                   │
  │  ├── orders/                                           │
  │  │   └── {id}  → order-service                         │
  │  └── products/ → product-service                       │
  │                                                        │
  │  Matching priority:                                     │
  │  1. Exact match (/api/v1/users/me)                     │
  │  2. Parameterized (/api/v1/users/{id})                 │
  │  3. Wildcard (/api/v1/users/*)                         │
  │  4. Regex (last resort, expensive)                     │
  │                                                        │
  │  Hot reload: watch config store → rebuild trie in-place│
  │  Zero-downtime route update: atomic pointer swap       │
  └────────────────────────────────────────────────────────┘
```

---

## 8. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Horizontal Scaling:                                    │
  │  • Gateway nodes are stateless (shared-nothing)         │
  │  • Scale with NLB / L4 load balancer in front           │
  │  • Single node: ~50K concurrent connections (Envoy)     │
  │  • 10 nodes: ~500K concurrent, ~200K RPS               │
  │                                                         │
  │  Config Store Scaling:                                  │
  │  • Route/plugin config in PostgreSQL or etcd            │
  │  • Watch-based push to all gateway nodes                │
  │  • In-memory config cache: no DB call per request       │
  │                                                         │
  │  Per-Tenant Scaling:                                    │
  │  • Separate gateway fleet per environment (prod/staging)│
  │  • Dedicated gateways for high-traffic tenants          │
  │  • Shared gateway with per-consumer rate limits         │
  │                                                         │
  │  Global Scaling:                                        │
  │  • Deploy gateway in each region / edge PoP             │
  │  • Anycast IP: route to nearest gateway                 │
  │  • Edge TLS termination reduces latency                 │
  └─────────────────────────────────────────────────────────┘
```

---

## 9. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Gateway State is Minimal (stateless by design):        │
  │  • No user data stored in gateway                      │
  │  • Route config: persisted in config store (PostgreSQL) │
  │  • Rate limit counters: Redis (ephemeral, OK to lose)   │
  │                                                         │
  │  Request Durability:                                    │
  │  • At-least-once delivery: retry on 502/503/504         │
  │  • Idempotency: pass X-Idempotency-Key to backends     │
  │  • Request logging: async to Kafka (buffered, durable)  │
  │  • DLQ for failed requests that exhaust retries         │
  │                                                         │
  │  Configuration Protection:                              │
  │  • Config DB: multi-AZ PostgreSQL with WAL archiving    │
  │  • Config versioning: rollback to previous version      │
  │  • Declarative config in Git (GitOps)                   │
  │  • Canary config rollout: apply to 1 node first         │
  └─────────────────────────────────────────────────────────┘
```

---

## 10. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Gateway overhead (passthrough)│ 0.5 ms  │ 2 ms    │
  │ With JWT validation          │ 1 ms     │ 5 ms    │
  │ With rate limiting (Redis)   │ 0.3 ms   │ 1 ms    │
  │ With request transformation  │ 1 ms     │ 3 ms    │
  │ TLS termination              │ 0.5 ms   │ 2 ms    │
  │ Full pipeline (all plugins)  │ 2 ms     │ 10 ms   │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Connection pooling to backends (avoid TCP handshake)│
  │  • HTTP/2 multiplexing (single connection, many reqs)  │
  │  • JWT validation: cache JWKS keys, fast local verify  │
  │  • TLS session resumption (avoid full handshake)       │
  │  • Async logging (buffer → batch write to Kafka)       │
  │  • Zero-copy proxying (splice/sendfile)                │
  │  • Avoid regex routing (use trie-based path matching)  │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Backend service down   │ Circuit breaker: stop sending   │
  │                        │ after N failures; return 503    │
  │                        │ or cached stale response        │
  ├────────────────────────┼─────────────────────────────────┤
  │ Gateway node crash     │ NLB health check removes node;  │
  │                        │ client retries to healthy node  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Config store down      │ Use last cached config in memory│
  │                        │ Continue serving current routes │
  ├────────────────────────┼─────────────────────────────────┤
  │ Rate limit store down  │ Fail open: allow all traffic    │
  │ (Redis failure)        │ Fall back to local rate counter │
  ├────────────────────────┼─────────────────────────────────┤
  │ Bad config deployed    │ Canary rollout to 1 node first; │
  │                        │ health check catches errors;    │
  │                        │ auto-rollback to previous       │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 12. Availability

```
  Target: 99.999% (gateway is on the critical path for ALL traffic)

  ┌─────────────────────────────────────────────────────────┐
  │  Architecture:                                          │
  │  • N+2 gateway instances (tolerate 2 failures)          │
  │  • Spread across 3+ AZs                                 │
  │  • NLB in front (AWS NLB: 99.99% SLA)                  │
  │  • Active health checks every 5s                        │
  │                                                         │
  │  Zero-Downtime Operations:                              │
  │  • Rolling restart during config update                 │
  │  • Connection draining: wait for in-flight requests     │
  │  • Binary upgrade: blue-green gateway fleet             │
  │                                                         │
  │  Graceful Degradation:                                  │
  │  • If auth service down → allow cached sessions         │
  │  • If rate limiter down → fail open                     │
  │  • If logging down → drop logs (never block requests)  │
  │  • If backend slow → timeout + return 504 quickly       │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication (first line of defense):                │
  │  • API keys: fast lookup in hash map                    │
  │  • JWT / OAuth2: validate signature + expiry            │
  │  • mTLS: client certificate for service-to-service     │
  │  • HMAC signed requests (AWS Signature v4 style)        │
  │                                                         │
  │  DDoS Protection:                                       │
  │  • Connection limits per IP                             │
  │  • Request rate limiting per consumer + per IP          │
  │  • Payload size limits (reject > 10 MB bodies)          │
  │  • Slowloris protection: read timeout on headers        │
  │  • WAF integration: OWASP rules, SQL injection, XSS    │
  │                                                         │
  │  Data Protection:                                       │
  │  • TLS 1.3 (terminate at gateway, re-encrypt to backend)│
  │  • Sensitive header stripping (remove internal headers) │
  │  • Response masking: hide backend error details          │
  │  • PII redaction in access logs                          │
  │                                                         │
  │  Access Control:                                        │
  │  • Per-route ACLs: which consumers can call which API   │
  │  • Scope-based authorization within JWT claims          │
  │  • Admin API: separate network, mTLS + RBAC             │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Gateway Compute:                                       │
  │  • Envoy / Kong: ~4 vCPU per 50K RPS                    │
  │  • 200K RPS → 4 × c5.2xlarge ≈ $2K/month               │
  │                                                         │
  │  Supporting Infrastructure:                             │
  │  • Config store (PostgreSQL RDS): ~$200/month           │
  │  • Rate limiting (ElastiCache Redis): ~$300/month       │
  │  • NLB: ~$100/month + $0.006/GB processed              │
  │  • Logging (Kafka → S3): ~$500/month                    │
  │                                                         │
  │  Total: ~$3K/month for 200K RPS                         │
  │                                                         │
  │  Managed Gateway Costs:                                 │
  │  • AWS API Gateway: $3.50/million requests              │
  │  • At 200K RPS: $3.50 × 518M ≈ $1.8M/month (!)       │
  │  • Self-hosted: 600× cheaper at high volume             │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Use managed for low-traffic APIs (< 10M req/month)  │
  │  • Self-hosted for high-traffic APIs                     │
  │  • Response caching reduces backend calls by 30-50%     │
  │  • Request coalescing for duplicate concurrent requests │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **API gateway vs service mesh?** — Gateway: north-south traffic (external). Service mesh: east-west (internal service-to-service)
2. **How to handle gateway as a bottleneck?** — Horizontally scale, offload TLS to hardware LB, minimize plugin overhead
3. **How to avoid single point of failure?** — Multiple gateway nodes behind LB; graceful degradation if config store is down (use cached config)
4. **gRPC support?** — Gateway must handle HTTP/2, protocol translation (REST ↔ gRPC)
5. **How to do canary deployments through the gateway?** — Weight-based routing: send % of traffic to canary backend; monitor error rates before promoting
