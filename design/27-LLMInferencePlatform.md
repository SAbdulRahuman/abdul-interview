# Design an LLM Inference Platform

Examples: OpenAI API, Anthropic Claude API, vLLM, TensorRT-LLM, Hugging Face TGI

---

## 1. Requirements

### Functional
- Serve multiple LLM models (7B to 405B parameters)
- Support chat completions and text completions APIs
- Token-by-token streaming responses
- Multi-turn conversation context management
- Function calling / tool use
- Fine-tuned model hosting per customer

### Non-Functional
- Time to first token (TTFT) < 500ms
- Token throughput: > 100 tokens/sec per request
- Serve 100K+ concurrent requests
- Efficient GPU utilization (> 70%)
- Graceful degradation under load

---

## 2. Scale Estimation

```
Concurrent users:    100K
Avg input tokens:    500
Avg output tokens:   200
Request rate:        10K requests/sec
GPU memory per model:
  7B (FP16):   ~14 GB  (fits on 1× A100 80GB)
  70B (FP16):  ~140 GB (needs 2× A100 80GB)
  405B (FP16): ~810 GB (needs 10× A100 80GB)
  With INT4 quantization: ~4× reduction
Tokens/sec per A100: ~2000 tokens/sec (7B model)
A100s needed for 10K rps × 200 tokens: ~1000 GPUs
```

---

## 3. High-Level Architecture

```
  ┌──────────┐     ┌──────────────┐     ┌──────────────────┐
  │  Client   │────▶│  API Gateway │────▶│  Request Router  │
  │  (SDK)    │     │  (auth, rate │     │  (model routing, │
  │           │◀────│   limit)     │◀────│   load balance)  │
  │  SSE      │     └──────────────┘     └────────┬─────────┘
  └──────────┘                                    │
                                    ┌─────────────┼─────────────┐
                                    ▼             ▼             ▼
                             ┌───────────┐ ┌───────────┐ ┌───────────┐
                             │ GPU Pool  │ │ GPU Pool  │ │ GPU Pool  │
                             │ Model: 7B │ │ Model:70B │ │ Model:405B│
                             │ 50 GPUs   │ │ 200 GPUs  │ │ 500 GPUs  │
                             │           │ │           │ │           │
                             │ ┌───────┐ │ │ ┌───────┐ │ │ ┌───────┐ │
                             │ │vLLM   │ │ │ │vLLM   │ │ │ │vLLM   │ │
                             │ │Engine │ │ │ │Engine │ │ │ │Engine │ │
                             │ └───────┘ │ │ └───────┘ │ │ └───────┘ │
                             └───────────┘ └───────────┘ └───────────┘
                                    │             │             │
                                    └─────────────┼─────────────┘
                                                  │
                             ┌────────────────────▼────────────────┐
                             │  KV Cache Store (distributed)      │
                             │  (for multi-turn conversations)    │
                             └────────────────────────────────────┘
```

---

## 4. GPU Inference Engine

```
  Key optimization techniques:

  1. CONTINUOUS BATCHING:
  ┌──────────────────────────────────────────────┐
  │  Naive: wait for batch of N requests → infer │
  │  Problem: short requests wait for long ones  │
  │                                              │
  │  Continuous batching:                        │
  │  • New requests join batch mid-execution     │
  │  • Completed requests leave immediately      │
  │                                              │
  │  Iteration 1: [R1, R2, R3]                  │
  │  Iteration 2: [R1, R2, R3, R4_new]          │
  │  Iteration 3: [R1, R3, R4, R5_new] (R2 done)│
  │                                              │
  │  Result: 2-5× higher throughput vs static    │
  └──────────────────────────────────────────────┘

  2. KV CACHE MANAGEMENT (PagedAttention / vLLM):
  ┌──────────────────────────────────────────────┐
  │  Problem: KV cache uses 30-50% of GPU memory │
  │  Each request reserves max_tokens × cache    │
  │  Most requests don't use max_tokens → waste  │
  │                                              │
  │  PagedAttention:                             │
  │  • Allocate cache in pages (like virtual mem)│
  │  • Pages mapped non-contiguously             │
  │  • Allocate on demand, not upfront           │
  │  • Shared prefix caching (system prompts)    │
  │                                              │
  │  Result: fit 2-4× more concurrent requests   │
  └──────────────────────────────────────────────┘

  3. QUANTIZATION:
  ┌──────────────────────────────────────────────┐
  │  FP16 → INT8: 2× less memory, ~5% quality   │
  │  FP16 → INT4: 4× less memory, ~10% quality  │
  │  GPTQ, AWQ, GGUF formats                    │
  │                                              │
  │  70B FP16: 140 GB → 2× A100                 │
  │  70B INT4:  35 GB → 1× A100                 │
  └──────────────────────────────────────────────┘

  4. TENSOR PARALLELISM (for large models):
  ┌──────────────────────────────────────────────┐
  │  Split model layers across GPUs:             │
  │                                              │
  │  GPU 0: attention heads 0-31                 │
  │  GPU 1: attention heads 32-63               │
  │  (within same node, NVLink)                  │
  │                                              │
  │  Pipeline parallelism (across nodes):        │
  │  Node 0: layers 0-39                         │
  │  Node 1: layers 40-79                        │
  └──────────────────────────────────────────────┘
```

---

## 5. Request Routing & Load Balancing

```
  Router decisions:
  ┌──────────────────────────────────────────────┐
  │  1. Model selection:                         │
  │     request.model = "gpt-4" → GPU pool 405B │
  │     request.model = "gpt-3.5" → GPU pool 7B │
  │                                              │
  │  2. Load-aware routing:                      │
  │     • Track: pending requests per instance   │
  │     • Track: KV cache utilization per GPU    │
  │     • Route to least-loaded instance         │
  │                                              │
  │  3. Prefix caching affinity:                 │
  │     Same system prompt → same instance       │
  │     (reuse cached KV for system prompt)      │
  │                                              │
  │  4. Priority queues:                         │
  │     Paid tier: P0 (immediate)                │
  │     Free tier: P2 (best effort, may queue)   │
  └──────────────────────────────────────────────┘
```

---

## 6. Streaming Response

```
  Server-Sent Events (SSE):

  Client                              Server
    │  POST /v1/chat/completions        │
    │  stream: true                     │
    │──────────────────────────────────▶│
    │                                   │
    │  data: {"choices":[{"delta":      │
    │         {"content":"Hello"}}]}    │
    │◀──────────────────────────────────│
    │                                   │
    │  data: {"choices":[{"delta":      │
    │         {"content":" world"}}]}   │
    │◀──────────────────────────────────│
    │                                   │
    │  data: [DONE]                     │
    │◀──────────────────────────────────│

  Why streaming?
  • TTFT < 500ms (user sees output immediately)
  • Without streaming: wait 5-30 seconds for full response
  • Better UX: progressive rendering like typing
```

---

## 7. Multi-Turn Conversation

```
  Option A: Stateless (client sends full context)
  ┌──────────────────────────────────────────────┐
  │  Request 1: [system, user1]                  │
  │  Request 2: [system, user1, asst1, user2]    │
  │  Request 3: [system, user1, asst1, user2,    │
  │              asst2, user3]                    │
  │                                              │
  │  Pro: Simple, no server state                │
  │  Con: Redundant computation (re-encode prefix│
  │       every request)                         │
  └──────────────────────────────────────────────┘

  Option B: Prefix caching (KV cache reuse)
  ┌──────────────────────────────────────────────┐
  │  Cache KV for conversation prefix            │
  │  Request 3: only compute user3 tokens        │
  │  Reuse KV cache from previous turns          │
  │                                              │
  │  TTFT: 500ms → 100ms for follow-up turns     │
  │                                              │
  │  Implementation:                             │
  │  • Hash(conversation_id) → route to same GPU │
  │  • KV cache persisted in GPU memory or       │
  │    offloaded to CPU/SSD between turns        │
  │  • TTL: evict after 10 min of inactivity     │
  └──────────────────────────────────────────────┘
```

---

## 8. Fault Tolerance

- **GPU failure:** Health check every 5s; remove from pool; redistribute requests
- **Request retry:** On timeout, retry on different GPU instance (idempotent generation with same seed)
- **KV cache loss:** Regenerate from conversation history (slightly higher TTFT)
- **Model loading:** Pre-load models on standby GPUs; hot-swap on failure
- **Overload:** Queue with priority; shed free-tier requests; return 503 with retry-after

---

## 9. Low-Level Design (LLD)

### API Contracts

```
# Chat Completion (OpenAI-compatible)
POST /api/v1/chat/completions
Headers: Authorization: Bearer <api_key>
{
  "model": "llama-3.1-70b",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Explain transformers in 3 sentences."}
  ],
  "max_tokens": 256,
  "temperature": 0.7,
  "top_p": 0.9,
  "stream": true,
  "stop": ["\n\n"],
  "user": "user_abc123"         // for rate limiting & logging
}

# Streaming Response (SSE)
data: {"id":"chatcmpl-xyz","choices":[{"delta":{"content":"Trans"},"index":0}]}
data: {"id":"chatcmpl-xyz","choices":[{"delta":{"content":"formers"},"index":0}]}
...
data: {"id":"chatcmpl-xyz","choices":[{"delta":{},"finish_reason":"stop"}],
       "usage":{"prompt_tokens":28,"completion_tokens":42,"total_tokens":70}}
data: [DONE]

# Model Management (internal)
POST /internal/v1/models/deploy
{
  "model": "llama-3.1-70b",
  "quantization": "AWQ-INT4",
  "tensor_parallel": 4,           // GPUs per instance
  "min_replicas": 2,
  "max_replicas": 10,
  "gpu_type": "A100-80GB"
}

# Health & Metrics
GET /internal/v1/health
Response: {
  "models": {
    "llama-3.1-70b": {
      "replicas": 4,
      "gpu_utilization_avg": 0.78,
      "kv_cache_utilization": 0.62,
      "queue_depth": 12,
      "tokens_per_second": 2400
    }
  }
}
```

### GPU Inference Engine Internals

```
vLLM / TensorRT-LLM Engine Architecture:

┌──────────────────────────────────────────────────────────┐
│                   Request Scheduler                      │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Waiting Queue (sorted by arrival time)           │   │
│  │   [req_1(28 tok), req_2(512 tok), req_3(100 tok)]│   │
│  └──────────────┬───────────────────────────────────┘   │
│                 ▼                                        │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Continuous Batching Scheduler                    │   │
│  │                                                  │   │
│  │  Active batch: [req_1(gen), req_5(gen), req_7(pf)]│  │
│  │  - pf = prefill phase (process all prompt tokens)│   │
│  │  - gen = generation phase (one token at a time)  │   │
│  │                                                  │   │
│  │  Each iteration:                                 │   │
│  │    1. Check if any req finished → evict from batch│  │
│  │    2. Check if new reqs can join → add to batch  │   │
│  │    3. Run forward pass for entire batch           │   │
│  │    4. Sample next token for each req in batch    │   │
│  └──────────────┬───────────────────────────────────┘   │
│                 ▼                                        │
│  ┌──────────────────────────────────────────────────┐   │
│  │ KV Cache Manager (PagedAttention)                │   │
│  │                                                  │   │
│  │  Physical GPU memory: 80GB                       │   │
│  │  Model weights:       35GB (70B @ INT4)          │   │
│  │  KV cache pool:       40GB                       │   │
│  │  OS/overhead:          5GB                       │   │
│  │                                                  │   │
│  │  Block size: 16 tokens                           │   │
│  │  Block memory: 2MB (per layer per head)          │   │
│  │  Total blocks: ~20,000                           │   │
│  │                                                  │   │
│  │  Per request allocation:                         │   │
│  │    req_1: blocks [0,1,2] (48 tokens)             │   │
│  │    req_5: blocks [3,4,5,6,7] (80 tokens)        │   │
│  │    req_7: blocks [8..15] (128 tokens prefill)    │   │
│  │                                                  │   │
│  │  Prefix caching:                                 │   │
│  │    System prompt "You are..." → shared block [99] │  │
│  │    All requests with same prefix reuse block     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  GPU Execution:                                          │
│    Tensor Parallelism (TP=4 across 4 GPUs):             │
│    GPU 0: layers 0-19, attention heads 0-15             │
│    GPU 1: layers 0-19, attention heads 16-31            │
│    GPU 2: layers 0-19, attention heads 32-47            │
│    GPU 3: layers 0-19, attention heads 48-63            │
│    All-reduce after every attention + FFN layer         │
│    NVLink bandwidth: 900 GB/s (A100 SXM)               │
└──────────────────────────────────────────────────────────┘
```

### Model Serving Pipeline

```
Request Lifecycle:
  1. API Gateway receives request
  2. Auth + rate limit check (Redis token bucket)
  3. Route to model pool based on model name
  4. Load balancer selects GPU instance:
     Strategy: least_kv_cache_utilization
     (not least_connections — GPU memory matters more)
  5. Request enters waiting queue on chosen instance
  6. Scheduler admits to batch when KV cache available
  7. Prefill: process all prompt tokens in one forward pass
  8. Generation loop: one token per iteration
     - Token streamed to client via SSE after each iteration
     - Stop conditions: max_tokens, stop_sequence, EOS token
  9. KV cache blocks freed; next waiting request admitted
  
  Throughput math (70B model, 4×A100):
    Prefill: ~5000 tokens/sec
    Generation: ~60 tokens/sec per request
    Batch size 32: ~1920 tokens/sec decode throughput
    One 8-GPU node: ~3840 tokens/sec
```

## 10. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Concurrent Requests** | PagedAttention: 2-4× more concurrent requests per GPU vs static allocation | 64 concurrent per 4-GPU node |
| **Throughput** | Continuous batching: near-100% GPU utilization; batch sizes up to 256 | 4000 tokens/sec per 8-GPU node |
| **Model Variety** | Each model gets dedicated GPU pool; no model switching overhead | 10+ models simultaneously |
| **Peak Traffic** | Auto-scale GPU instances based on queue depth + KV cache utilization | Scale 2× in 5min |
| **Long Context** | Ring attention / sequence parallelism for 128K+ context | 128K tokens with 8 GPUs |

**Scaling Architecture:**
```
Auto-Scaling Policy:
  Scale UP when:
    kv_cache_utilization > 80% for 2min  OR
    queue_depth > 50 for 1min            OR
    p99_TTFT > 5s for 2min (time to first token)
  
  Scale DOWN when:
    kv_cache_utilization < 30% for 10min AND
    queue_depth < 5 for 10min

  GPU Pool Topology:
    Small models (7B):   1 GPU per instance, TP=1
    Medium models (70B): 4 GPUs per instance, TP=4  
    Large models (405B): 8 GPUs per instance, TP=8
    
  Multi-node (405B+):
    Pipeline parallelism across 2 nodes (16 GPUs total)
    Layers 0-39: Node 1 (8 GPUs, TP=8)
    Layers 40-79: Node 2 (8 GPUs, TP=8)
    Micro-batching to hide inter-node latency
```

## 11. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Request/Response Logs** | All requests + responses logged to Kafka (acks=all) → S3 archival for audit |
| **Conversation State** | Multi-turn context stored in Redis (with persistence) + PostgreSQL; not dependent on GPU state |
| **Model Weights** | Stored in S3 with versioning; local NVMe cache; integrity checksums (SHA-256) |
| **KV Cache** | Ephemeral (exists only during request processing); loss = request retry, not data loss |
| **Usage Metering** | Token counts logged synchronously before response; billing based on log, not memory |
| **Fine-tuned Models** | Model artifacts stored in S3 with version tags; training checkpoints every epoch |

**Recovery:**
```
GPU Node Crash:
  1. In-flight requests fail (KV cache lost)
  2. Load balancer detects health check failure (5s)
  3. Clients retry automatically (streaming reconnect)
  4. Retry hits other healthy GPU nodes
  5. No data loss: conversations stored externally
  
Model Corruption:
  1. Checksum mismatch detected on load
  2. Re-download from S3 (model registry)
  3. Validate checksums before serving
  4. Zero requests served from corrupt model
```

## 12. Latency

| Metric | 7B Model | 70B Model | 405B Model |
|--------|----------|-----------|------------|
| **Time to First Token (TTFT)** | 50ms | 200ms | 800ms |
| **Inter-Token Latency (ITL)** | 8ms | 15ms | 30ms |
| **100-token response total** | 850ms | 1.7s | 3.8s |
| **Queue wait (p50)** | 0ms | 50ms | 200ms |
| **Queue wait (p99)** | 200ms | 2s | 10s |

**Optimization Techniques:**
- **Speculative decoding**: Draft model (7B) generates 5 candidate tokens; large model verifies in one pass — 2-3× faster
- **Prefix caching**: System prompts shared across requests; skip repeated prefill — 30-50% TTFT reduction
- **Quantization**: INT4 AWQ → 4× less memory, 2× faster decode, <1% quality loss
- **Flash Attention**: O(N) memory attention implementation; faster for long sequences
- **KV cache compression**: Grouped Query Attention (GQA) in newer models → 8× less KV cache
- **Streaming**: First token sent immediately; user perceives faster response

## 13. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **GPU failure (hardware)** | Node loses all active requests | Health check every 5s; retry on other nodes; N+1 redundancy |
| **OOM (KV cache full)** | New requests rejected | Admission control: reject when >90% KV utilization; preempt lower-priority requests |
| **Model produces garbage** | Bad user experience | Output guardrails: toxicity filter, length checks, format validation |
| **Slow generation** | Timeout | Client-side timeout 60s; server-side max_tokens limit; kill long requests |
| **NVLink failure** | TP communication broken | Monitor NVLink health; failover to another multi-GPU node |
| **Hot model** | Queue buildup | Auto-scale; spillover to alternate model (e.g., 70B → 8B for simple queries) |

## 14. Availability

**Target: 99.9% for API, 99.99% for cached/simple requests**

```
Availability Architecture:
┌──────────────────────────────────────────────────┐
│              API Gateway + LB                    │
│    (L7, model-aware routing)                     │
├─────────────┬──────────────┬─────────────────────┤
│  GPU Pool 1 │  GPU Pool 2  │  GPU Pool 3         │
│  70B model  │  70B model   │  7B model           │
│  4×A100     │  4×A100      │  1×A100             │
│  (primary)  │  (primary)   │  (fast fallback)    │
└─────────────┴──────────────┴─────────────────────┘

Fallback Chain:
  1. Primary model instance → 2. Other replica → 
  3. Smaller quantized model → 4. Cached response (if similar prompt seen)

Graceful Degradation:
  - All GPU pools overloaded → Return "service busy, retry in 30s"
  - Single model down → Route to alternative model + disclaimer
  - Partial GPU failure → Reduce batch size; serve with lower throughput
  - Network to GPU cluster down → Serve from response cache (stale but fast)
```

## 15. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | API keys per tenant; JWT for dashboard; mTLS for internal |
| **Rate Limiting** | Per-key token-bucket: 100K tokens/min; per-model limits |
| **Input Safety** | Prompt injection detection; PII scrubbing before logging |
| **Output Safety** | Toxicity filter (Llama Guard); content policy enforcement |
| **Data Privacy** | No prompt/response stored beyond 30 days (configurable per tenant); opt-out of training |
| **Model Security** | Weights encrypted at rest; access controls on model registry; no model exfiltration |
| **Network** | GPU cluster in private subnet; API gateway is only public-facing component |
| **Audit** | Full request/response logging; token-level usage per API key |

## 16. Cost Constraints

**Estimated Cost (serving 70B model, 1000 req/sec, ~100M tokens/day):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **GPU Instances** | 10× 4×A100-80GB nodes (p4d.24xlarge) | $250,000 |
| **GPU (spot/reserved)** | 3-year reserved = 60% discount | $100,000 |
| **API Gateway** | 10× c6g.xlarge | $3,500 |
| **Redis (context cache)** | r6g.2xlarge cluster | $3,000 |
| **Logging/Storage** | Kafka + S3 (request logs) | $5,000 |
| **Networking** | Inter-AZ GPU traffic | $2,000 |
| **Total (reserved)** | | **~$113,500/month** |

**Cost per token: ~$0.000035 (vs OpenAI GPT-4 at $0.00003 input)**

**Cost Optimization:**
- **Quantization (INT4)**: 4× fewer GPUs needed → $62K savings/month
- **Speculative decoding**: 2× throughput → halve GPU count for same traffic
- **Request batching**: Continuous batching → 3-4× throughput vs sequential
- **Prefix caching**: Shared system prompts → 30% fewer prefill FLOPs
- **Spot instances**: Use for non-real-time batch inference → 70% savings
- **Model routing**: Route 60% of simple queries to 7B model → 10× cheaper per token
- **Right-size models**: Use 8B for summarization, 70B for coding, 405B only for complex reasoning

## Key Interview Discussion Points

1. **Why is GPU memory the bottleneck?** — Model weights + KV cache fill GPU memory; more concurrent requests = more KV cache needed
2. **PagedAttention benefit?** — Virtual memory for KV cache; allocate on demand; 2-4× more concurrent requests
3. **Continuous batching vs static batching?** — Static wastes GPU cycles waiting for longest request; continuous achieves near-100% GPU utilization
4. **How to serve a 405B model?** — Tensor parallelism across 8-10 GPUs within a node; pipeline parallelism across nodes
5. **How to reduce cost?** — Quantization (INT4), smaller model routing for simple queries, spot/preemptible GPU instances, speculative decoding
