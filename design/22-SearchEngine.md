# Design a Search Engine / Autocomplete System

Examples: Google Search, Elasticsearch, Algolia

---

## 1. Requirements

### Functional
- Full-text search across billions of documents
- Autocomplete / typeahead suggestions (< 50ms)
- Ranked results by relevance + popularity
- Spelling correction and query suggestions
- Faceted search and filtering

### Non-Functional
- 10B+ documents indexed
- Search latency < 200ms (p99)
- Autocomplete latency < 50ms
- Index freshness: minutes for high-priority content
- 99.99% availability

---

## 2. High-Level Architecture

```
  ┌──────────┐  query   ┌──────────────┐     ┌──────────────────┐
  │  User     │────────▶│  Query       │────▶│  Search Cluster  │
  │  Browser  │         │  Service     │     │  (distributed    │
  │           │◀────────│  (parse,     │◀────│   inverted index)│
  │           │ results │   rank)      │     │                  │
  └──────────┘         └──────────────┘     └──────────────────┘
                              │
                       ┌──────▼──────┐
                       │  Spelling   │
                       │  Correction │
                       └─────────────┘

  ┌──────────┐  type    ┌──────────────┐     ┌──────────────────┐
  │  User     │────────▶│  Autocomplete│────▶│  Prefix Index    │
  │  (keystrk)│◀────────│  Service     │     │  (Trie / Redis)  │
  └──────────┘ suggest  └──────────────┘     └──────────────────┘

  ┌──────────┐  crawl   ┌──────────────┐     ┌──────────────────┐
  │  Crawler  │────────▶│  Indexing    │────▶│  Search Cluster  │
  │           │         │  Pipeline    │     │  (build index)   │
  └──────────┘         └──────────────┘     └──────────────────┘
```

---

## 3. Inverted Index

```
  Document corpus:
  Doc1: "distributed systems are fun"
  Doc2: "distributed databases scale"
  Doc3: "systems design interview"

  Forward index (doc → terms):
  ┌──────┬─────────────────────────────┐
  │ Doc1 │ distributed, systems, fun   │
  │ Doc2 │ distributed, databases, scale│
  │ Doc3 │ systems, design, interview  │
  └──────┴─────────────────────────────┘

  Inverted index (term → docs):
  ┌──────────────┬─────────────────────────┐
  │ Term         │ Posting List            │
  ├──────────────┼─────────────────────────┤
  │ distributed  │ [Doc1:pos0, Doc2:pos0]  │
  │ systems      │ [Doc1:pos1, Doc3:pos0]  │
  │ databases    │ [Doc2:pos1]             │
  │ fun          │ [Doc1:pos2]             │
  │ scale        │ [Doc2:pos2]             │
  │ design       │ [Doc3:pos1]             │
  │ interview    │ [Doc3:pos2]             │
  └──────────────┴─────────────────────────┘

  Query: "distributed systems"
  → posting("distributed") ∩ posting("systems")
  → [Doc1:pos0, Doc2:pos0] ∩ [Doc1:pos1, Doc3:pos0]
  → [Doc1]  (both terms found)
```

---

## 4. Text Processing Pipeline

```
  Raw text: "The Quick Brown Fox Jumps Over The Lazy Dog!!!"

  Step 1: Tokenize
  → ["The", "Quick", "Brown", "Fox", "Jumps", "Over", "The", "Lazy", "Dog"]

  Step 2: Lowercase
  → ["the", "quick", "brown", "fox", "jumps", "over", "the", "lazy", "dog"]

  Step 3: Remove stop words
  → ["quick", "brown", "fox", "jumps", "lazy", "dog"]

  Step 4: Stem / Lemmatize
  → ["quick", "brown", "fox", "jump", "lazi", "dog"]

  Step 5: Store in inverted index with positions and term frequencies
```

---

## 5. Ranking: BM25

```
  BM25 scoring formula:
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  score(q, d) = Σ  IDF(qi) ×                 │
  │                    tf(qi, d) × (k1 + 1)      │
  │                    ─────────────────────────  │
  │                    tf(qi,d) + k1×(1-b+b×|d|/│
  │                                     avgdl)   │
  │                                              │
  │  Where:                                      │
  │  • IDF = log((N - n + 0.5) / (n + 0.5))     │
  │    N = total docs, n = docs containing term  │
  │    Rare terms get higher weight              │
  │                                              │
  │  • tf = term frequency in document           │
  │    More occurrences = higher relevance        │
  │                                              │
  │  • |d|/avgdl = document length normalization │
  │    Penalize very long documents              │
  │                                              │
  │  • k1=1.2, b=0.75 (typical values)          │
  │                                              │
  │  Combined with:                              │
  │  • PageRank (link-based authority)           │
  │  • Click-through rate from past queries      │
  │  • Freshness boost for recent content        │
  └──────────────────────────────────────────────┘
```

---

## 6. Autocomplete / Typeahead

```
  User types: "dist"
  Show suggestions: ["distributed systems", "distance calculator", ...]

  Implementation: Trie + popularity scores
  ┌──────────────────────────────────────────────┐
  │            (root)                             │
  │              │                               │
  │              d                               │
  │              │                               │
  │              i                               │
  │              │                               │
  │              s                               │
  │              │                               │
  │              t                               │
  │             / \                              │
  │            a   r                             │
  │            │    │                            │
  │            n    i                            │
  │            │    │                            │
  │            c    b                            │
  │            │    │                            │
  │            e    u                            │
  │                 │                            │
  │                 t                            │
  │                 │                            │
  │                 e                            │
  │                 │                            │
  │                 d                            │
  │                                              │
  │  Each node stores: top-5 completions by score│
  │  "dist" node → ["distributed systems" (9.2), │
  │                  "distance calculator" (7.1), │
  │                  "distribution center" (5.4)] │
  └──────────────────────────────────────────────┘

  Scaling autocomplete:
  • Redis sorted sets: ZRANGEBYLEX for prefix matching
  • Pre-computed: build trie offline, deploy as in-memory service
  • Update periodically (every few hours) from query logs
  • Shard by prefix range: a-f → shard1, g-m → shard2, etc.
```

---

## 7. Distributed Index Sharding

```
  Two strategies:

  Option A: Shard by Document (recommended)
  ┌──────────────────────────────────────────────┐
  │  Each shard contains ALL terms for a subset  │
  │  of documents                                │
  │                                              │
  │  Shard 1: Doc 1-1M    (full inverted index)  │
  │  Shard 2: Doc 1M-2M   (full inverted index)  │
  │  Shard 3: Doc 2M-3M   (full inverted index)  │
  │                                              │
  │  Query: fan out to ALL shards                │
  │         merge top-K results                  │
  │                                              │
  │  Pro: Independent index builds per shard     │
  │  Con: Every query hits every shard           │
  └──────────────────────────────────────────────┘

  Option B: Shard by Term
  ┌──────────────────────────────────────────────┐
  │  Each shard contains ALL docs for a subset   │
  │  of terms                                    │
  │                                              │
  │  Shard 1: terms a-f  (posting lists for a-f) │
  │  Shard 2: terms g-m                          │
  │  Shard 3: terms n-z                          │
  │                                              │
  │  Query "distributed systems":                │
  │  → shard for 'd' + shard for 's'             │
  │  → intersect posting lists across shards     │
  │                                              │
  │  Pro: Query touches fewer shards             │
  │  Con: Cross-shard intersection is complex    │
  └──────────────────────────────────────────────┘

  Each shard has replicas for read scalability:
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Shard 1  │    │ Shard 1  │    │ Shard 1  │
  │ Primary  │    │ Replica  │    │ Replica  │
  └──────────┘    └──────────┘    └──────────┘
```

---

## 8. Fault Tolerance

- **Shard failure:** Replicas serve reads; new replica rebuilt from primary
- **Index corruption:** Rebuild from source documents; checksum verification
- **Stale index:** Near-real-time indexing for high-priority docs; batch reindex for full refresh
- **Query overload:** Circuit breaker; return cached results; shed low-priority queries

---

## 9. Low-Level Design (LLD)

### API Contracts

```
# Search API
POST /api/v1/search
{
  "query": "distributed systems",
  "filters": {"language": "en", "date_range": "2024-01-01.."},
  "page": 0,
  "size": 10,
  "sort": "relevance"        // relevance | date | popularity
}
Response: {
  "results": [
    {
      "doc_id": "d_abc123",
      "url": "https://example.com/...",
      "title": "...",
      "snippet": "...distributed <em>systems</em>...",
      "score": 12.45,
      "cached_url": "/cache/d_abc123"
    }
  ],
  "total_hits": 42390,
  "took_ms": 85,
  "spell_suggestion": "distributed system",
  "facets": {"language": {"en": 30000, "es": 5000}}
}

# Autocomplete API
GET /api/v1/suggest?q=distri&limit=5
Response: {
  "suggestions": [
    {"text": "distributed systems", "score": 0.95},
    {"text": "distributed computing", "score": 0.87}
  ]
}

# Index API (internal)
POST /api/v1/index
{
  "doc_id": "d_abc123",
  "url": "https://...",
  "title": "...",
  "body": "...",
  "language": "en",
  "crawl_timestamp": 1700000000
}
```

### Inverted Index Internal Structure

```
Inverted Index Segment (on-disk format):
┌──────────────────────────────────────────────────┐
│ Term Dictionary (FST - Finite State Transducer)  │
│  "cache"   → posting_offset=0x1A00, df=45230    │
│  "cached"  → posting_offset=0x1B80, df=12100    │
│  "caching" → posting_offset=0x1C40, df=8900     │
├──────────────────────────────────────────────────┤
│ Posting Lists (compressed with PForDelta)        │
│  "cache":                                        │
│    Doc IDs:  [12, 47, 89, 102, 156, ...]         │
│    Stored as delta: [12, 35, 42, 13, 54, ...]    │
│    TF per doc: [3, 1, 7, 2, 1, ...]              │
│    Positions: [[5,12,99],[3],[1,8,14,20,55,61,70]]│
├──────────────────────────────────────────────────┤
│ Skip List (for posting list intersection)        │
│    Every 128 docs: [doc_id, offset]              │
│    Enables O(log n) seek in posting traversal    │
├──────────────────────────────────────────────────┤
│ Stored Fields (compressed with LZ4)              │
│  doc_12: {title: "...", url: "...", length: 450} │
│  doc_47: {title: "...", url: "...", length: 230} │
├──────────────────────────────────────────────────┤
│ Norms (per-doc field lengths for BM25)           │
│  doc_12: fieldLen=450, avgFieldLen=320           │
└──────────────────────────────────────────────────┘
```

### Query Processing Pipeline

```
User Query: "best distributed cache systems"
         │
         ▼
┌─────────────────────────┐
│ 1. Query Parser         │
│   Parse into AST:       │
│   AND(best, distributed,│
│       cache, systems)   │
├─────────────────────────┤
│ 2. Query Rewriter       │
│   - Spell correction    │
│   - Synonym expansion   │
│     cache → (cache OR   │
│              caching)   │
│   - Stopword removal    │
│     remove "best"       │
├─────────────────────────┤
│ 3. Scatter to Shards    │
│   Broadcast to all N    │
│   index shards in       │
│   parallel              │
├─────────────────────────┤
│ 4. Per-Shard Execution  │
│   a. Posting list fetch │
│      for each term      │
│   b. Intersection using │
│      skip lists         │
│   c. BM25 scoring       │
│   d. Top-K via min-heap │
│      (return top 50)    │
├─────────────────────────┤
│ 5. Gather & Merge       │
│   Merge N sorted lists  │
│   → global top-K        │
│   Apply re-ranking:     │
│   final = 0.6×BM25 +    │
│     0.2×PageRank +      │
│     0.1×freshness +     │
│     0.1×click_score     │
├─────────────────────────┤
│ 6. Snippet Generation   │
│   Highlight matching    │
│   terms in stored text  │
│   Return best 2-3       │
│   sentence fragments    │
└─────────────────────────┘
```

## 10. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Index Size** | Document-partitioned shards (hash of doc_id) | 10Bn documents across 1000 shards |
| **Query Throughput** | Replica shards (3 replicas per shard) | 100K QPS |
| **Index Updates** | Near-real-time: buffer in memory segment, flush every 1s, merge in background | <1s indexing latency |
| **Autocomplete** | Separate trie service with prefix sharding | 500K suggest QPS |
| **Feature Ranking** | Offline ML model export → online scoring in C++ | <5ms per 1000 candidates |

**Horizontal Scaling:**
```
Scaling Strategy:
  Index grows  → Add more shards (re-index with higher shard count)
  QPS grows    → Add more replicas per shard
  Latency goal → Reduce shard size (fewer docs per shard = faster scan)

Elasticsearch approach:
  - Primary shards: set at index creation (immutable)
  - Replica shards: adjust dynamically
  - Index lifecycle: hot → warm → cold → frozen → delete
  - Rollover: create new index when size > 50GB or age > 7d
```

## 11. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Index Segments** | Translog (WAL) persisted before ACK; replayed on crash recovery |
| **Replication** | Synchronous write to primary + in-sync replicas before client ACK |
| **Source Documents** | Store original crawled content in object storage (S3); index is reproducible |
| **Cluster State** | Master-eligible nodes with quorum (min_master_nodes = N/2 + 1); prevents split-brain |
| **Index Updates** | Sequence numbers per operation; follower replicas detect gaps and request resync |

**Recovery Plan:**
```
Scenario: Shard lost (disk failure)
  1. Allocate new shard on another node
  2. Copy from replica (peer recovery)
  3. Replay translog for ops during copy
  4. Promote to in-sync replica

Scenario: Full index corruption
  1. Re-index from source documents in object storage
  2. Or restore from snapshot (hourly to S3)
  3. Replay crawler queue from last snapshot timestamp
```

## 12. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Simple keyword search | 20ms | 100ms | <200ms |
| Complex bool query | 50ms | 300ms | <500ms |
| Autocomplete | 5ms | 20ms | <50ms |
| Index single doc | 2ms | 10ms | <100ms |
| Snippet generation | 10ms | 50ms | <100ms |

**Optimization Techniques:**
- **Query cache**: Cache results for frequent queries (LRU, 10% of heap)
- **Request cache**: Cache shard-level aggregation results; invalidate on refresh
- **Filesystem cache**: OS page cache for index segments (allocate 50% RAM)
- **Warm-up**: Pre-load frequent terms' posting lists into memory on node start
- **Concurrent shard search**: Query all shards in parallel; total latency ≈ slowest shard
- **Early termination**: Stop scoring after top-K candidates reach quality threshold
- **Approximate nearest neighbor**: HNSW index for vector similarity (semantic search)

## 13. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Data node crash** | Shard unavailable | Replica promotion in <30s; reroute queries to replicas |
| **Master node crash** | Cluster state stale | Master election from eligible nodes (Raft); recovered in <10s |
| **Slow shard (hot spot)** | p99 latency spike | Adaptive replica selection: route to fastest replica |
| **Bad query (wildcard leading)** | CPU spike, affects others | Query circuit breaker: kill queries exceeding 30s or 70% heap |
| **Index corruption** | Wrong results | Checksum verification on segment open; automatic shard recovery |
| **Cascading failure** | Full cluster down | Coordinating node limits concurrent shard requests; backpressure |

## 14. Availability

**Target: 99.99% for search, 99.9% for indexing**

```
Availability Architecture:
┌─────────────────────────────────────────────────┐
│               Global Load Balancer              │
│         (DNS round-robin + health check)        │
├──────────────┬──────────────┬───────────────────┤
│  Cluster A   │  Cluster B   │  Cluster C        │
│  (us-east)   │  (eu-west)   │  (ap-south)       │
│  Full index  │  Full index  │  Full index       │
│  replica     │  replica     │  replica          │
└──────────────┴──────────────┴───────────────────┘

Within each cluster:
  - 3 master-eligible nodes (quorum = 2)
  - N data nodes with RF=2 per shard
  - Coordinating-only nodes for query routing

Graceful Degradation:
  1. Cluster unhealthy → DNS removes region
  2. Some shards unavail → Return partial results + warning
  3. Index overloaded → Serve from stale cache (search cache TTL 60s)
  4. Autocomplete down → Disable suggest, search still works
```

## 15. Security

| Layer | Mechanism |
|-------|-----------|
| **Transport** | TLS 1.3 between all nodes (internode encryption) |
| **Authentication** | API keys per application; SAML/OIDC for dashboard users |
| **Authorization** | Document-level security: filter results by user role (DLS queries) |
| **Field-level** | Mask sensitive fields (SSN, email) based on caller role |
| **Audit** | Log all admin operations and query patterns (compliance) |
| **Content Safety** | Filter offensive autocomplete suggestions; block query injection |
| **Rate Limiting** | Per-API-key: 1000 QPS for search, 100 QPS for index |
| **Data at Rest** | Encrypted index segments (AES-256); encrypted snapshots |

## 16. Cost Constraints

**Estimated Cost (10 Billion documents, 100K search QPS):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **Data Nodes** | 100× r6g.2xlarge (64GB RAM, 2TB NVMe) | $45,000 |
| **Master Nodes** | 3× m6g.xlarge | $500 |
| **Coordinating Nodes** | 10× c6g.2xlarge | $3,500 |
| **Storage (snapshots)** | S3: 50TB compressed | $1,200 |
| **Cross-AZ transfer** | 500TB/month | $5,000 |
| **Crawler Infrastructure** | 50 workers + queue | $4,000 |
| **Total** | | **~$59,200/month** |

**Cost Optimization:**
- **Hot/Warm/Cold architecture**: Move old indices to warm nodes (larger disk, less RAM) — saves 40%
- **Frozen tier**: Searchable snapshots from S3 — 90% cheaper than hot
- **Force merge**: Reduce segment count on read-only indices — less RAM overhead
- **Source-only snapshots**: Don't store `_source` for rebuilt-from-crawler indices

**Build vs Buy:**
- Self-hosted Elasticsearch: $59K/month (above)
- Elastic Cloud: ~$120K/month (managed, includes monitoring)
- Amazon OpenSearch Serverless: ~$80K/month (no capacity planning)

## Key Interview Discussion Points

1. **How to update index in real-time?** — Dual-buffer: serve from one index while building the other; swap atomically. Or: append to in-memory segment, merge periodically
2. **How to handle multi-language search?** — Language-specific tokenizers and stemmers; language detection per document
3. **How to prevent autocomplete abuse?** — Cache popular prefixes; rate limit per user; filter offensive completions
4. **Sharding by document vs term?** — Document sharding is simpler and more common (Elasticsearch uses it). Term sharding reduces fan-out but complicates index updates
5. **How to rank when combining text + signals?** — Linear combination of BM25 score + PageRank + freshness + click-through features; tune weights with A/B testing
