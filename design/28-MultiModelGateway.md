# Design a Multi-Model Gateway

Examples: LiteLLM, OpenRouter, AWS Bedrock, Azure OpenAI Gateway

---

## 1. Requirements

### Functional
- Unified API to access multiple LLM providers (OpenAI, Anthropic, Google, open-source)
- Automatic failover between providers
- Model routing based on cost, latency, and capability
- Request/response format translation
- Usage tracking and billing aggregation

### Non-Functional
- < 50ms added latency (proxy overhead)
- 99.99% availability (higher than any single provider)
- Support 50K+ concurrent requests
- Provider-agnostic: add new providers without code changes

---

## 2. High-Level Architecture

```
  ┌──────────┐     ┌──────────────────────────────────────┐
  │  Client   │────▶│  Multi-Model Gateway                │
  │  App      │     │  ┌──────────────────────────────┐   │
  │           │     │  │  Unified API Layer            │   │
  │           │     │  │  POST /v1/chat/completions    │   │
  │           │     │  └──────────┬───────────────────┘   │
  │           │     │             │                        │
  │           │     │  ┌──────────▼───────────────────┐   │
  │           │     │  │  Router / Policy Engine       │   │
  │           │     │  │  • model → provider mapping   │   │
  │           │     │  │  • cost optimization          │   │
  │           │     │  │  • latency-based routing      │   │
  │           │     │  │  • fallback chain              │   │
  │           │     │  └──────────┬───────────────────┘   │
  │           │     │             │                        │
  │           │     │  ┌──────────▼───────────────────┐   │
  │           │     │  │  Provider Adapters            │   │
  │           │     │  │  ┌────────┐ ┌────────┐       │   │
  │           │     │  │  │ OpenAI │ │Anthropic│       │   │
  │           │     │  │  └────────┘ └────────┘       │   │
  │           │     │  │  ┌────────┐ ┌────────┐       │   │
  │           │     │  │  │ Google │ │  Local  │       │   │
  │           │     │  │  │ Gemini │ │  vLLM   │       │   │
  │           │     │  │  └────────┘ └────────┘       │   │
  │           │     │  └──────────────────────────────┘   │
  └──────────┘     └──────────────────────────────────────┘
                              │
                   ┌──────────▼──────────┐
                   │  Usage Tracker      │
                   │  (tokens, cost,     │
                   │   latency metrics)  │
                   └─────────────────────┘
```

---

## 3. Routing & Fallback Strategy

```
  Routing decision tree:

  Request: model="gpt-4", max_tokens=1000

  ┌──────────────────────────────────────────────┐
  │  1. Model mapping:                           │
  │     "gpt-4" → [OpenAI, Azure OpenAI]        │
  │     "claude-3" → [Anthropic]                 │
  │     "best" → route to cheapest capable model │
  │                                              │
  │  2. Provider health check:                   │
  │     OpenAI: healthy (p99 = 2s)               │
  │     Azure:  degraded (p99 = 8s)              │
  │     → prefer OpenAI                          │
  │                                              │
  │  3. Rate limit check:                        │
  │     OpenAI:  80% of quota used               │
  │     Azure:   20% of quota used               │
  │     → split traffic 20/80 to balance quota   │
  │                                              │
  │  4. Fallback chain:                          │
  │     Primary:  OpenAI GPT-4                   │
  │     Fallback: Azure OpenAI GPT-4             │
  │     Fallback: Anthropic Claude 3 Opus        │
  │     Fallback: Local vLLM (Llama 70B)         │
  └──────────────────────────────────────────────┘

  Circuit breaker per provider:
  ┌──────────────────────────────────────────────┐
  │  If >30% failures in 1 min → OPEN            │
  │  Route to next in fallback chain              │
  │  After 30s → HALF-OPEN → test 1 request      │
  │  If success → CLOSE → resume normal routing  │
  └──────────────────────────────────────────────┘
```

---

## 4. Request/Response Translation

```
  Unified format → Provider-specific format:

  Input (unified):
  {
    "model": "gpt-4",
    "messages": [
      {"role": "system", "content": "You are helpful"},
      {"role": "user", "content": "Hello"}
    ],
    "max_tokens": 100
  }

  → OpenAI adapter: pass through (native format)

  → Anthropic adapter: translate to
  {
    "model": "claude-3-opus-20240229",
    "system": "You are helpful",
    "messages": [
      {"role": "user", "content": "Hello"}
    ],
    "max_tokens": 100
  }

  → Google adapter: translate to
  {
    "model": "gemini-1.5-pro",
    "contents": [
      {"role": "user", "parts": [{"text": "Hello"}]}
    ],
    "systemInstruction": {"parts": [{"text": "You are helpful"}]},
    "generationConfig": {"maxOutputTokens": 100}
  }

  Response: reverse-translate back to unified format
```

---

## 5. Cost Optimization

```
  Model cost comparison (per 1M tokens):
  ┌──────────────────┬────────┬────────┐
  │ Model            │ Input  │ Output │
  ├──────────────────┼────────┼────────┤
  │ GPT-4o           │ $2.50  │ $10.00 │
  │ Claude 3.5 Sonnet│ $3.00  │ $15.00 │
  │ Gemini 1.5 Pro   │ $1.25  │ $5.00  │
  │ Llama-70B (self) │ $0.50  │ $0.50  │
  │ GPT-4o mini      │ $0.15  │ $0.60  │
  └──────────────────┴────────┴────────┘

  Smart routing:
  ┌──────────────────────────────────────────────┐
  │  Simple queries (short, factual):            │
  │  → GPT-4o mini or Llama-70B (cheap)         │
  │                                              │
  │  Complex queries (reasoning, coding):        │
  │  → GPT-4o or Claude 3.5 Sonnet (expensive)  │
  │                                              │
  │  Classifier: lightweight model scores query  │
  │  complexity → route accordingly              │
  │  Savings: 40-60% vs always using best model  │
  └──────────────────────────────────────────────┘
```

---

## 6. Fault Tolerance

- **Provider outage:** Circuit breaker → automatic failover to next provider
- **Rate limiting:** Spread across multiple API keys; queue excess with backpressure
- **Increased latency:** Route to fastest provider based on real-time p99 tracking
- **Streaming failure:** Reconnect to same or fallback provider; resume from last token
- **Gateway crash:** Stateless design; load balancer routes to healthy instance

---

## 7. Low-Level Design (LLD)

### API Contracts

```
# Unified Chat Completion
POST /api/v1/chat/completions
Headers: Authorization: Bearer <gateway_api_key>
{
  "model": "auto",              // auto | gpt-4o | claude-3.5 | llama-70b
  "messages": [
    {"role": "system", "content": "Summarize the text."},
    {"role": "user", "content": "The transformer architecture..."}
  ],
  "max_tokens": 500,
  "temperature": 0.7,
  "stream": true,
  "routing": {
    "strategy": "cost_optimized",  // cost_optimized | quality_first | latency_first
    "fallback_models": ["claude-3.5-sonnet", "gpt-4o-mini"],
    "max_cost_per_request": 0.05   // USD
  }
}

# Response (unified format regardless of provider)
{
  "id": "gw-chatcmpl-abc123",
  "model": "claude-3.5-sonnet",    // actual model used
  "provider": "anthropic",
  "choices": [{"message": {"role": "assistant", "content": "..."}}],
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 200,
    "total_tokens": 350,
    "cost_usd": 0.0045
  },
  "routing_metadata": {
    "strategy": "cost_optimized",
    "candidates_considered": 3,
    "selection_reason": "lowest_cost_meeting_quality_threshold"
  }
}

# Provider Management (admin)
POST /admin/v1/providers
{
  "name": "anthropic",
  "base_url": "https://api.anthropic.com",
  "api_key_ref": "vault://anthropic/prod-key",
  "models": [
    {"id": "claude-3.5-sonnet", "input_cost_per_1k": 0.003, "output_cost_per_1k": 0.015,
     "max_tokens": 200000, "capabilities": ["vision", "tool_use"]}
  ],
  "rate_limits": {"rpm": 4000, "tpm": 400000},
  "health_check_interval_sec": 30
}

# Usage Dashboard
GET /api/v1/usage?period=30d&group_by=model
Response: {
  "total_cost_usd": 12450.00,
  "total_requests": 2500000,
  "by_model": [
    {"model": "gpt-4o-mini", "requests": 1800000, "cost": 3600, "avg_latency_ms": 800},
    {"model": "claude-3.5-sonnet", "requests": 500000, "cost": 7500, "avg_latency_ms": 1200}
  ]
}
```

### Router Decision Engine

```
Routing Pipeline:
┌──────────────────────────────────────────────────────────┐
│ 1. Request Classification (if model="auto")             │
│    ┌────────────────────────────────────────────┐        │
│    │ Lightweight classifier (fine-tuned DistilBERT)│     │
│    │ Input: system prompt + first 200 chars of user│     │
│    │ Output: complexity_score (0.0 - 1.0)          │     │
│    │                                                │     │
│    │ 0.0-0.3 → SIMPLE  (summarize, translate, QA)  │     │
│    │ 0.3-0.7 → MEDIUM  (analysis, writing)         │     │
│    │ 0.7-1.0 → COMPLEX (code, reasoning, math)     │     │
│    └────────────────────────────────────────────┘        │
├──────────────────────────────────────────────────────────┤
│ 2. Candidate Selection                                  │
│    Filter models by:                                     │
│    - Capability match (vision, tool_use, long_context)  │
│    - Context length sufficient for input                 │
│    - Provider healthy (circuit breaker CLOSED)           │
│    - Rate limit available (token bucket > 0)             │
├──────────────────────────────────────────────────────────┤
│ 3. Scoring (per strategy)                               │
│                                                          │
│    cost_optimized:                                       │
│      score = quality_meets_threshold ? 1/cost : -inf     │
│                                                          │
│    quality_first:                                        │
│      score = quality_score - 0.1 × normalized_cost      │
│                                                          │
│    latency_first:                                        │
│      score = 1/p99_latency - 0.05 × normalized_cost     │
├──────────────────────────────────────────────────────────┤
│ 4. Provider Adapter                                      │
│    Transform unified request → provider-specific format │
│    OpenAI:     messages[] as-is                          │
│    Anthropic:  messages[] → system separate, max_tokens  │
│    Google:     messages[] → contents[], generationConfig │
│    Self-hosted: messages[] → vLLM format                │
├──────────────────────────────────────────────────────────┤
│ 5. Response Normalization                                │
│    Provider response → unified format                   │
│    Streaming: adapt SSE format per provider              │
│    Token counting: use provider-reported or tiktoken     │
└──────────────────────────────────────────────────────────┘
```

### Cache Architecture

```
Multi-Level Cache:
┌──────────────────────────────────────────────────┐
│ L1: Exact Match Cache (Redis)                    │
│   Key: SHA-256(model + messages + temperature)   │
│   Value: full response                           │
│   TTL: 1 hour                                    │
│   Hit rate: ~15% (identical prompts)             │
├──────────────────────────────────────────────────┤
│ L2: Semantic Cache (Vector DB)                   │
│   Key: embedding(prompt)                         │
│   Match: cosine_similarity > 0.95                │
│   Value: cached response + metadata              │
│   Hit rate: ~25% (similar prompts)               │
│   Cost savings: avoid $0.01+ LLM call            │
├──────────────────────────────────────────────────┤
│ Cache Invalidation:                              │
│   - TTL-based (configurable per model)           │
│   - Manual purge via admin API                   │
│   - temperature > 0 responses: shorter TTL       │
│   - temperature = 0 responses: longer TTL        │
└──────────────────────────────────────────────────┘
```

## 8. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Request Volume** | Stateless gateway replicas behind L7 LB | 50K RPS per gateway cluster |
| **Provider Limits** | Distribute across multiple API keys per provider; round-robin | 10× single-key limits |
| **Model Count** | Plugin adapter architecture; add new providers without restart | 50+ models, 20+ providers |
| **Cache** | Redis cluster with read replicas; sharded by prompt hash | 10M cached responses |
| **Tenants** | Per-tenant rate limits, quotas, and routing preferences | 10K+ tenants |

**Scaling:**
```
Gateway Horizontal Scaling:
  Gateway is stateless → scale by adding replicas
  Provider rate limits → distributed token bucket (Redis-based)
  
  Rate Limiting Flow:
    1. Request arrives at gateway instance
    2. EVALSHA rate_limit.lua provider_key model_key
    3. If tokens available → decrement and proceed
    4. If exhausted → try next provider (fallback chain)
    5. All providers exhausted → 429 with retry_after header
```

## 9. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Request Logs** | All requests + routing decisions logged to Kafka → S3 (append-only, immutable) |
| **API Keys** | Stored in HashiCorp Vault; encrypted at rest; never in application config |
| **Provider Credentials** | Vault dynamic secrets; auto-rotation every 30 days |
| **Usage Metering** | Token counts logged synchronously before response ACK; billing reconciled daily |
| **Response Cache** | Redis with AOF persistence; cache loss is non-critical (re-fetched from provider) |
| **Routing Config** | GitOps-managed; all changes versioned; instant rollback |

## 10. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Router decision | 2ms | 10ms | <20ms |
| Cache lookup (L1 exact) | 1ms | 5ms | <10ms |
| Cache lookup (L2 semantic) | 10ms | 50ms | <100ms |
| Gateway overhead (no cache) | 5ms | 20ms | <30ms |
| Provider call (GPT-4o) | 800ms | 3s | Provider-dependent |
| Provider call (Claude 3.5) | 600ms | 2.5s | Provider-dependent |

**Optimization:**
- **Parallel provider calls**: For quality comparison, call 2 providers simultaneously; return first good response
- **Streaming pass-through**: Zero buffering; stream SSE tokens directly from provider to client
- **Connection pool**: Persistent HTTPS connections to each provider (avoid TLS handshake per request)
- **Classifier caching**: Cache complexity classification for repeated system prompts
- **Edge routing**: Deploy gateway POPs in multiple regions to minimize client→gateway RTT

## 11. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Provider outage** | Model unavailable | Circuit breaker (5 failures in 30s → OPEN 60s); auto-failover to next provider |
| **Provider rate limit** | 429 errors | Distributed rate limit tracking; pre-emptive routing to backup when near limit |
| **Provider degradation** | High latency | Latency threshold: if p99 > 5s, reduce traffic to that provider (weighted routing) |
| **Gateway crash** | Requests dropped | Stateless replicas; L7 LB detects failure in <5s; clients retry |
| **Bad model response** | Quality issue | Output validation: check for empty/truncated responses; auto-retry with different provider |
| **Cost spike** | Budget exceeded | Per-tenant spending limits; alert at 80%; hard block at 100% |

## 12. Availability

**Target: 99.99% (gateway), 99.9% per individual model**

```
Availability Strategy:
  Key insight: gateway availability > any single provider
  
  Provider A: 99.9% uptime
  Provider B: 99.9% uptime
  Together with failover: 99.9999% (1 - 0.001²)

Architecture:
  ┌──────────────────────────────────────────┐
  │            Global DNS + LB              │
  ├──────────────┬──────────────────────────┤
  │  US Region   │  EU Region              │
  │  Gateway ×3  │  Gateway ×3             │
  │  Redis ×3    │  Redis ×3               │
  │  Cache sync  │  Cache sync             │
  ├──────────────┴──────────────────────────┤
  │         Provider Connections            │
  │  OpenAI → Anthropic → Google → Self-   │
  │  hosted (fallback chain per model)     │
  └─────────────────────────────────────────┘

Graceful Degradation:
  1. Best model unavailable → Route to next-best model
  2. All premium models down → Route to self-hosted fallback
  3. Cache hit → Serve cached response (even if all providers down)
  4. Everything down → Return honest error with retry-after
```

## 13. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | Gateway API keys (per tenant); OAuth for dashboard |
| **Authorization** | Per-tenant model allowlist; spending limits; content policy |
| **Credential Management** | Provider API keys in Vault; rotated every 30 days; never logged |
| **Data Privacy** | Prompt/response data encrypted in transit (TLS 1.3) and at rest; tenant isolation |
| **Content Filtering** | Pre-flight: PII detection + redaction; post-flight: toxicity check |
| **Rate Limiting** | Per-key: RPM + TPM limits; per-IP: abuse prevention |
| **Audit** | Full request/response audit log; immutable; SOC2 compliant |
| **Prompt Injection** | Input validation; detect and block malicious prompt patterns |

## 14. Cost Constraints

**Estimated Cost (10M requests/month, mix of models):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **LLM Provider Fees** | 60% GPT-4o-mini, 30% Claude 3.5, 10% GPT-4o | $45,000 |
| **Gateway Instances** | 6× c6g.xlarge (2 regions × 3) | $2,000 |
| **Redis Cache** | r6g.large cluster (2 regions) | $1,500 |
| **Vector DB (semantic cache)** | Qdrant 3-node cluster | $1,200 |
| **Logging/Analytics** | Kafka + S3 + dashboard | $2,000 |
| **Total** | | **~$51,700/month** |

**Cost Optimization:**
- **Smart routing**: Route 60% to cheapest model → saves $15K/month vs all-premium
- **Semantic cache**: 25% hit rate at $0.01/request avg → saves $25K/month
- **Prompt compression**: Remove redundant tokens from long prompts → 10-20% token savings
- **Batch requests**: Aggregate non-urgent requests; submit in batch to providers (lower per-token cost)
- **Self-hosted fallback**: Deploy Llama 70B for baseline queries → $0.0003/request vs $0.003/request API

## Key Interview Discussion Points

1. **Why a gateway instead of direct provider calls?** — Unified API, automatic failover, cost optimization, centralized usage tracking
2. **How to handle streaming across different providers?** — Adapter layer normalizes SSE formats; buffer and re-emit in unified format
3. **How to add a new provider?** — Implement adapter interface (translate request, translate response, health check); no core changes
4. **Cost vs quality trade-off?** — Route simple queries to cheap models, complex to expensive; classifier overhead < $0.001/request
5. **Caching?** — Cache identical prompts (hash-based); semantic caching for similar queries (embedding similarity)
