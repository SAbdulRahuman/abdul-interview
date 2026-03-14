# Design a Vector Database

Examples: Pinecone, Weaviate, Milvus, Qdrant, pgvector

---

## 1. Requirements

### Functional
- Store high-dimensional vectors (768-4096 dimensions) with metadata
- Similarity search: find K nearest neighbors
- CRUD operations on vectors
- Filter by metadata during search (hybrid search)
- Namespace/collection management

### Non-Functional
- < 50ms search latency for 100M vectors
- Support 10K+ queries per second
- Recall > 95% (approximate nearest neighbor acceptable)
- Real-time index updates (not batch-only)
- Horizontal scalability

---

## 2. High-Level Architecture

```
  ┌──────────┐     ┌──────────────┐     ┌──────────────────┐
  │  Client   │────▶│  API Layer   │────▶│  Query Planner   │
  │  (SDK)    │     │  (REST/gRPC) │     │  (route to shards│
  └──────────┘     └──────────────┘     └────────┬─────────┘
                                                  │
                                   ┌──────────────┼──────────────┐
                                   ▼              ▼              ▼
                            ┌───────────┐  ┌───────────┐  ┌───────────┐
                            │  Shard 0  │  │  Shard 1  │  │  Shard N  │
                            │  ┌─────┐  │  │  ┌─────┐  │  │  ┌─────┐  │
                            │  │ ANN │  │  │  │ ANN │  │  │  │ ANN │  │
                            │  │Index│  │  │  │Index│  │  │  │Index│  │
                            │  └─────┘  │  │  └─────┘  │  │  └─────┘  │
                            │  ┌─────┐  │  │  ┌─────┐  │  │  ┌─────┐  │
                            │  │Meta │  │  │  │Meta │  │  │  │Meta │  │
                            │  │Store│  │  │  │Store│  │  │  │Store│  │
                            │  └─────┘  │  │  └─────┘  │  │  └─────┘  │
                            └───────────┘  └───────────┘  └───────────┘
                                   │              │              │
                                   └──────────────┼──────────────┘
                                                  │ merge top-K
                                           ┌──────▼──────┐
                                           │   Results   │
                                           └─────────────┘
```

---

## 3. ANN Index Algorithms

```
  Exact KNN: compare query against ALL vectors → O(N×D)
  Too slow for 100M vectors. Use approximate nearest neighbor (ANN).

  ALGORITHM 1: HNSW (Hierarchical Navigable Small World)
  ┌──────────────────────────────────────────────┐
  │  Multi-layer graph:                          │
  │                                              │
  │  Layer 3:  A ─────────────── D               │
  │  (sparse)  (long-range connections)          │
  │                                              │
  │  Layer 2:  A ──── B ──── D                   │
  │                                              │
  │  Layer 1:  A ─ B ─ C ─ D ─ E                │
  │                                              │
  │  Layer 0:  A─B─C─D─E─F─G─H─I─J             │
  │  (dense)   (all vectors connected locally)   │
  │                                              │
  │  Search: start at top layer, greedy descent  │
  │  O(log N) per query                          │
  │                                              │
  │  Pro: Fast queries, good recall (~99%)       │
  │  Con: High memory (graph links), slow build  │
  └──────────────────────────────────────────────┘

  ALGORITHM 2: IVF (Inverted File Index)
  ┌──────────────────────────────────────────────┐
  │  Cluster vectors into K centroids (K-means)  │
  │                                              │
  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐        │
  │  │ C1  │  │ C2  │  │ C3  │  │ C4  │        │
  │  │•••• │  │•••  │  │••••••│  │••   │        │
  │  │•••  │  │••••│  │•••   │  │•••• │        │
  │  └─────┘  └─────┘  └─────┘  └─────┘        │
  │                                              │
  │  Query: find nearest centroid(s)             │
  │  Search only vectors in nearest N clusters   │
  │  (nprobe parameter)                          │
  │                                              │
  │  Pro: Lower memory, fast build               │
  │  Con: Lower recall than HNSW, needs tuning   │
  └──────────────────────────────────────────────┘

  ALGORITHM 3: Product Quantization (PQ)
  ┌──────────────────────────────────────────────┐
  │  Compress 768-dim vector into ~32 bytes      │
  │  Split vector into sub-vectors (e.g., 8)     │
  │  Each sub-vector → codebook index (1 byte)   │
  │                                              │
  │  768 dims × 4 bytes = 3072 bytes/vector      │
  │  After PQ: 8 bytes/vector (~384× compression)│
  │                                              │
  │  Often combined: IVF-PQ (partition + compress)│
  │  Used by FAISS for billion-scale search       │
  └──────────────────────────────────────────────┘

  Comparison:
  ┌──────────┬─────────┬────────┬───────────┬──────────┐
  │ Algorithm│ Recall  │ Latency│ Memory    │ Build    │
  ├──────────┼─────────┼────────┼───────────┼──────────┤
  │ HNSW     │ ~99%    │ <1ms   │ High      │ Slow     │
  │ IVF      │ ~95%    │ <5ms   │ Medium    │ Fast     │
  │ IVF-PQ   │ ~90%    │ <10ms  │ Low       │ Medium   │
  │ Flat     │ 100%    │ slow   │ Baseline  │ None     │
  └──────────┴─────────┴────────┴───────────┴──────────┘
```

---

## 4. Distance Metrics

```
  ┌──────────────┬──────────────────────────────────┐
  │ Metric       │ Use case                         │
  ├──────────────┼──────────────────────────────────┤
  │ Cosine       │ Text embeddings (normalized)     │
  │              │ cos(a,b) = a·b / (|a| × |b|)    │
  ├──────────────┼──────────────────────────────────┤
  │ Euclidean    │ Image embeddings, spatial data   │
  │ (L2)         │ d = sqrt(Σ(ai-bi)²)             │
  ├──────────────┼──────────────────────────────────┤
  │ Dot product  │ Learned relevance scores         │
  │ (Inner prod) │ d = Σ(ai × bi)                  │
  └──────────────┴──────────────────────────────────┘
```

---

## 5. Hybrid Search (Vector + Metadata Filtering)

```
  Query: "Find similar documents WHERE category='finance' AND date > 2024"

  Two strategies:

  PRE-FILTER:
  ┌──────────────────────────────────────────────┐
  │  1. Filter by metadata → candidate set       │
  │  2. Vector search within candidate set       │
  │                                              │
  │  Pro: Accurate filtering                     │
  │  Con: Small candidate set → poor ANN recall  │
  └──────────────────────────────────────────────┘

  POST-FILTER:
  ┌──────────────────────────────────────────────┐
  │  1. Vector search → top 10×K candidates      │
  │  2. Filter by metadata → return top K        │
  │                                              │
  │  Pro: Good ANN recall                        │
  │  Con: May filter out too many → < K results  │
  └──────────────────────────────────────────────┘

  HYBRID (recommended):
  ┌──────────────────────────────────────────────┐
  │  Partition index by high-selectivity metadata│
  │  (e.g., category, tenant)                    │
  │  Within partition: standard ANN search       │
  │  Apply remaining filters as post-filter      │
  └──────────────────────────────────────────────┘
```

---

## 6. Sharding & Replication

```
  Sharding for 100M+ vectors:
  ┌──────────────────────────────────────────────┐
  │  Shard by vector ID (hash-based)             │
  │  Each shard: 10-20M vectors                  │
  │  Each shard fits in memory (HNSW graph)      │
  │                                              │
  │  Query: fan out to ALL shards                │
  │         each returns local top-K             │
  │         merge → global top-K                 │
  │                                              │
  │  Replication: each shard has 2 replicas      │
  │  Route reads to any replica                  │
  │  Write to primary → async replicate          │
  └──────────────────────────────────────────────┘
```

---

## 7. RAG (Retrieval-Augmented Generation) Integration

```
  Typical RAG pipeline:
  ┌──────────────────────────────────────────────┐
  │  1. User query: "What is WAFL?"              │
  │  2. Embed query → 768-dim vector             │
  │  3. Search vector DB → top 5 chunks          │
  │  4. Construct prompt:                        │
  │     "Context: [chunk1] [chunk2] [chunk3]     │
  │      Question: What is WAFL?"                │
  │  5. Send to LLM → grounded answer            │
  └──────────────────────────────────────────────┘

  Document ingestion:
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Document│───▶│ Chunker  │───▶│ Embedder │───▶│ Vector   │
  │ (PDF,   │    │ (512 tok │    │ (OpenAI  │    │ DB       │
  │  HTML)  │    │  overlap)│    │  ada-002) │    │ (upsert) │
  └─────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## 8. Fault Tolerance

- **Shard failure:** Replicas serve reads; rebuild shard from WAL
- **Index corruption:** Rebuild from stored raw vectors
- **Memory pressure:** Offload cold segments to disk (mmap)
- **Write surge:** Buffer writes; batch index updates

---

## 9. Low-Level Design (LLD)

### API Contracts

```
# Create Collection
POST /api/v1/collections
{
  "name": "product_embeddings",
  "dimension": 1536,
  "distance_metric": "cosine",      // cosine | euclidean | dot_product
  "index_type": "hnsw",
  "index_params": {
    "m": 16,                         // HNSW max connections per node
    "ef_construction": 200           // build-time search width
  },
  "shard_count": 4,
  "replication_factor": 3
}

# Upsert Vectors
POST /api/v1/collections/{name}/vectors/upsert
{
  "vectors": [
    {
      "id": "vec_abc123",
      "values": [0.1, -0.02, 0.87, ...],  // 1536 dims
      "metadata": {
        "category": "electronics",
        "price": 299.99,
        "brand": "Acme"
      }
    }
  ]
}

# Search (ANN query)
POST /api/v1/collections/{name}/search
{
  "query_vector": [0.15, -0.01, 0.92, ...],
  "top_k": 10,
  "ef_search": 100,                  // HNSW search width (higher = better recall, slower)
  "filter": {
    "must": [
      {"field": "category", "match": "electronics"},
      {"field": "price", "range": {"lte": 500}}
    ]
  },
  "include_metadata": true,
  "include_vectors": false
}
Response: {
  "results": [
    {"id": "vec_abc123", "score": 0.95, "metadata": {"category": "electronics", ...}},
    {"id": "vec_def456", "score": 0.87, "metadata": {...}}
  ],
  "search_time_ms": 3
}

# Batch Embedding + Search (convenience)
POST /api/v1/collections/{name}/query
{
  "text": "wireless noise-canceling headphones",
  "embedding_model": "text-embedding-3-small",
  "top_k": 10,
  "filter": {...}
}
```

### HNSW Index Internals

```
HNSW (Hierarchical Navigable Small World) Structure:

Layer 3 (sparse):    [A] ──────────────────── [M]
                      │                         │
Layer 2:         [A] ─── [D] ──── [H] ──── [M]
                  │       │        │         │
Layer 1:     [A]─[B]─[D]─[E]─[G]─[H]─[K]─[M]─[P]
              │   │   │   │   │   │   │   │   │
Layer 0:   [A][B][C][D][E][F][G][H][I][J][K][L][M][N][O][P]
(all nodes)

Search Algorithm (greedy beam search):
  1. Enter at top layer, start from entry point
  2. Greedily move to nearest neighbor
  3. When no closer neighbor found → descend to next layer
  4. At layer 0: expand to ef_search candidates
  5. Return top_k closest from candidate set

Parameters:
  M = 16         max edges per node per layer
  ef_construct = 200  build-time beam width
  ef_search = 100     query-time beam width
  
  Trade-offs:
    Higher M → better recall, more memory (16 bytes × M × nodes)
    Higher ef_search → better recall, higher latency
    Higher ef_construct → better graph quality, slower build

Memory Layout (per node, 1536-dim float32):
  Vector data:  1536 × 4 bytes = 6,144 bytes
  HNSW edges:   M × 4 bytes × layers ≈ 256 bytes
  Metadata idx: 8 bytes (pointer to metadata store)
  Total/vector: ~6.5 KB
  1M vectors:   ~6.2 GB
  1B vectors:   ~6.2 TB (requires sharding)
```

### Write Path & Index Update

```
Vector Upsert Pipeline:
┌──────────────────────────────────────────────────────┐
│ 1. Write to WAL (crash safety)                       │
│    Append: {op: "upsert", id, vector, metadata, ts}  │
├──────────────────────────────────────────────────────┤
│ 2. Insert into Memory Buffer (memtable)              │
│    In-memory HNSW graph for recent vectors           │
│    Capacity: 100K vectors per memtable               │
├──────────────────────────────────────────────────────┤
│ 3. When memtable full → Flush to Segment             │
│    Build optimized HNSW graph for segment             │
│    Write to disk: vector data + graph edges + metadata│
│    Create Bloom filter for ID lookups                │
├──────────────────────────────────────────────────────┤
│ 4. Background Segment Merge (compaction)             │
│    Merge N small segments → 1 large segment          │
│    Rebuild HNSW graph (better quality than incremental│
│    Remove deleted vectors                            │
├──────────────────────────────────────────────────────┤
│ 5. Search spans all segments                         │
│    Query memtable + all flushed segments              │
│    Merge results by distance score                   │
│    Like LSM-tree but for vector indices              │
└──────────────────────────────────────────────────────┘
```

## 10. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Vector Count** | Shard by vector ID hash across nodes | 1Bn vectors (100 shards × 10M each) |
| **Query Throughput** | Read replicas per shard; parallel search across shards | 100K QPS |
| **Dimensionality** | Product Quantization (PQ): compress 1536-dim → 192 bytes | 8× memory reduction |
| **Index Build** | Distributed index build: each shard builds independently | Build 1Bn index in <1 hour |
| **Multi-tenancy** | Collection-per-tenant with namespace isolation | 10K+ collections |

**Scaling Strategy:**
```
Memory vs Disk Trade-off:
  In-memory HNSW:  Best latency (1-5ms), $$$
  Disk-backed:     Good latency (10-50ms), $
  
  Hybrid approach:
    - Frequently accessed vectors: in-memory HNSW
    - Cold vectors: disk-backed with memory-mapped graph
    - Quantized vectors (PQ/SQ): 8× more vectors in same RAM
  
  Sharding:
    shard_key = hash(vector_id) % shard_count
    
    Search fanout: query ALL shards in parallel
    Each shard returns local top-K
    Coordinator merges → global top-K
    
    Adding shards: re-index (no live resharding for HNSW)
```

## 11. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Vectors** | WAL (Write-Ahead Log) before index insertion; replayed on crash |
| **Segments** | Flushed segments immutable on disk; checksummed; replicated RF=3 |
| **Metadata** | Stored alongside vectors in segments; metadata index separately checkpointed |
| **Replication** | Raft consensus for write ordering; synchronous replication to quorum |
| **Snapshots** | Periodic full snapshots to S3; enables point-in-time recovery |
| **Deletions** | Tombstone markers; physical deletion during compaction only |

**Recovery:**
```
Node Crash Recovery:
  1. Replay WAL from last checkpoint
  2. Rebuild memtable HNSW from WAL entries
  3. Flushed segments already on disk (no rebuild needed)
  4. Verify segment checksums on load
  Recovery time: <30s for 10M vectors
  
Full Shard Loss:
  1. Provision new node
  2. Restore latest snapshot from S3
  3. Stream WAL from replica since snapshot
  4. Join Raft group as follower
  5. Catch up and become available
```

## 12. Latency

| Operation | p50 | p99 | Notes |
|-----------|-----|-----|-------|
| ANN search (1M vectors, in-memory) | 1ms | 5ms | ef_search=100, top_k=10 |
| ANN search (100M vectors, sharded) | 5ms | 20ms | 10 shards, parallel |
| ANN search (1B vectors, PQ) | 15ms | 80ms | Disk-backed with PQ |
| Vector upsert (single) | 2ms | 10ms | WAL + memtable |
| Batch upsert (1000 vectors) | 50ms | 200ms | Batched WAL |
| Filtered search (1M, 10% pass filter) | 3ms | 15ms | Pre-filter + ANN |

**Optimization Techniques:**
- **SIMD distance computation**: AVX-512 instructions for cosine/euclidean (8× faster than scalar)
- **Prefetch hints**: CPU cache prefetch for HNSW neighbor traversal (reduce cache misses)
- **Quantization**: Scalar Quantization (SQ8) for 4× faster distance computation
- **Binary quantization**: Reduce to 1-bit → use popcount for Hamming distance → 32× faster first stage
- **Filtered search**: Pre-filter metadata → build candidate set → ANN on filtered subset
- **Graph pruning**: Remove long edges that are never traversed; reduces memory + speedup traversal

## 13. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Node crash** | Shard unavailable | Raft failover to replica in <10s; read replicas serve queries |
| **Index corruption** | Wrong search results | Checksum verification on segment load; rebuild from WAL |
| **OOM (too many vectors)** | Node crash | Memory limits per collection; evict cold segments to disk; PQ compression |
| **Slow query (high ef_search)** | Latency spike | Query timeout (100ms default); limit max ef_search per tenant |
| **Compaction storms** | Write latency spike | Rate-limit compaction I/O; schedule during low-traffic periods |
| **Embedding model change** | Incompatible vectors | Version collections; re-embed and rebuild index; dual-serve during migration |

## 14. Availability

**Target: 99.99% for search, 99.9% for writes**

```
Availability Architecture:
┌──────────────────────────────────────────────────┐
│              Query Router / LB                   │
│    (Route by collection → shard → replica)       │
├──────────────────────────────────────────────────┤
│  Shard 1          Shard 2          Shard 3       │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐   │
│  │Leader   │     │Leader   │     │Leader   │   │
│  │(writes) │     │(writes) │     │(writes) │   │
│  ├─────────┤     ├─────────┤     ├─────────┤   │
│  │Follower │     │Follower │     │Follower │   │
│  │(reads)  │     │(reads)  │     │(reads)  │   │
│  ├─────────┤     ├─────────┤     ├─────────┤   │
│  │Follower │     │Follower │     │Follower │   │
│  │(reads)  │     │(reads)  │     │(reads)  │   │
│  └─────────┘     └─────────┘     └─────────┘   │
└──────────────────────────────────────────────────┘

Graceful Degradation:
  1. Leader down → Follower promoted (Raft, <10s)
  2. One replica down → Serve from remaining replicas
  3. Whole shard down → Return partial results from other shards
  4. Write failures → Queue writes in WAL; replay when shard recovers
```

## 15. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | API keys per tenant; mTLS for inter-node communication |
| **Authorization** | Collection-level ACLs: read, write, admin per API key |
| **Multi-tenancy** | Namespace isolation; each tenant's data in separate collections |
| **Encryption** | TLS 1.3 in transit; AES-256 at rest for vector data + metadata |
| **Data Privacy** | Vectors are non-reversible embeddings (can't reconstruct original text easily) |
| **Access Control** | RBAC for admin operations (create/delete collection, manage replicas) |
| **Audit** | All write operations logged with tenant ID + timestamp |
| **Network** | Query nodes in private subnet; API gateway as only public endpoint |

## 16. Cost Constraints

**Estimated Cost (100M vectors, 1536-dim, 10K QPS search):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **Vector Storage (in-memory)** | 100M × 6.5KB = 620GB RAM; 20× r6g.4xlarge (128GB each) | $30,000 |
| **Vector Storage (PQ compressed)** | 100M × 0.8KB = 76GB RAM; 3× r6g.4xlarge | $4,500 |
| **Disk Storage (segments)** | 2TB NVMe per node | Included |
| **Object Storage (snapshots)** | S3: 1TB compressed | $23 |
| **Replication (RF=3)** | 3× above numbers | $13,500 (PQ) |
| **Query Router** | 3× c6g.xlarge | $1,000 |
| **Total (PQ, recommended)** | | **~$19,000/month** |

**Cost Optimization:**
- **Product Quantization (PQ)**: 8× memory reduction → $4.5K vs $30K for in-memory HNSW
- **Scalar Quantization (SQ8)**: 4× reduction, better recall than PQ → good middle ground
- **Tiered storage**: Hot vectors in RAM, warm on NVMe, cold in S3 (re-hydrate on demand)
- **Binary quantization**: 32× compression for first-stage retrieval; re-rank top-100 with full vectors
- **Managed vs self-hosted**: Pinecone $70/month for 1M vectors vs self-hosted at $190/month but 10× more capacity

**Build vs Buy:**
| Option | 100M Vectors | 1B Vectors |
|--------|-------------|------------|
| **Pinecone (managed)** | $7,000/month | $70,000/month |
| **Weaviate Cloud** | $5,000/month | $50,000/month |
| **Self-hosted (Qdrant/Milvus)** | $19,000/month | $190,000/month |
| **Self-hosted (PQ + disk)** | $5,000/month | $50,000/month |

## Key Interview Discussion Points

1. **HNSW vs IVF?** — HNSW: better recall, higher memory. IVF: lower memory, faster build, lower recall. Choose based on scale and latency requirements
2. **How to handle real-time updates?** — Write to in-memory buffer; periodically merge into main index (like LSM tree approach)
3. **How to scale to 1B vectors?** — IVF-PQ for memory efficiency; shard across nodes; GPU-accelerated search (FAISS)
4. **Cosine vs Euclidean?** — Cosine for text (direction matters, not magnitude). Euclidean for images/spatial
5. **How to evaluate retrieval quality?** — Recall@K, precision@K, MRR (Mean Reciprocal Rank); A/B test with LLM output quality
