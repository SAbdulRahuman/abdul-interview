# Design an Object Storage System (like Amazon S3)

Examples: Amazon S3, MinIO, NetApp StorageGRID

---

## 1. Requirements

### Functional
- `PUT /bucket/object` — upload object (up to 5 TB)
- `GET /bucket/object` — download object
- `DELETE /bucket/object` — delete object
- `LIST /bucket?prefix=` — list objects with prefix
- Support multipart upload for large objects
- Versioning, lifecycle policies, access control

### Non-Functional
- 99.999999999% durability (11 nines)
- 99.99% availability
- Support billions of objects and exabytes of data
- Low cost per GB for cold storage
- Strong read-after-write consistency

---

## 2. High-Level Architecture

```
  ┌────────────┐
  │   Client   │
  └─────┬──────┘
        │  HTTPS (REST API)
  ┌─────▼──────────────────────────────────────────────┐
  │                  API Gateway                        │
  │  (auth, routing, rate limiting)                     │
  └─────┬──────────────────────────────────────────────┘
        │
  ┌─────▼──────────────────────────────────────────────┐
  │                 Metadata Service                    │
  │  ┌──────────────────────────────────────────┐      │
  │  │  Object metadata (key → chunk locations)  │      │
  │  │  Bucket metadata (ACLs, versioning)       │      │
  │  │  Stored in: distributed DB (DynamoDB,     │      │
  │  │  CockroachDB, or custom B-tree store)     │      │
  │  └──────────────────────────────────────────┘      │
  └─────┬──────────────────────────────────────────────┘
        │
  ┌─────▼──────────────────────────────────────────────┐
  │               Data Storage Layer                    │
  │                                                    │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
  │  │ Storage  │  │ Storage  │  │ Storage  │        │
  │  │ Node 1   │  │ Node 2   │  │ Node 3   │  ...   │
  │  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │        │
  │  │ │Chunks│ │  │ │Chunks│ │  │ │Chunks│ │        │
  │  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │        │
  │  └──────────┘  └──────────┘  └──────────┘        │
  └────────────────────────────────────────────────────┘
```

---

## 3. Write Path (PUT Object)

```
  Client                    API Gateway          Metadata Svc        Storage Nodes
    │                           │                     │                    │
    │──PUT /photos/cat.jpg────▶│                     │                    │
    │  (5 MB file)             │                     │                    │
    │                          │──auth + validate───▶│                    │
    │                          │                     │                    │
    │                          │──select placement──▶│                    │
    │                          │  (which nodes for   │                    │
    │                          │   data + parity)    │                    │
    │                          │◀──node list─────────│                    │
    │                          │                     │                    │
    │                          │──stream chunks to storage nodes──────────▶│
    │                          │                     │                    │
    │                          │  Chunk 1 ──▶ Node A, Node D (replica)   │
    │                          │  Chunk 2 ──▶ Node B, Node E (replica)   │
    │                          │  Parity  ──▶ Node C, Node F (replica)   │
    │                          │                     │                    │
    │                          │◀──all chunks ACK'd──│                    │
    │                          │                     │                    │
    │                          │──write metadata────▶│                    │
    │                          │  key: photos/cat.jpg│                    │
    │                          │  chunks: [A,B,C]    │                    │
    │                          │  size: 5MB          │                    │
    │                          │  checksum: sha256   │                    │
    │                          │                     │                    │
    │◀─200 OK (ETag)───────────│                     │                    │
```

---

## 4. Data Durability — Erasure Coding

```
  Instead of 3× replication (3× storage cost):

  Erasure Coding (8,4) scheme:
  ┌─────────────────────────────────────────────┐
  │  Original object split into 8 data chunks   │
  │  4 parity chunks computed (Reed-Solomon)     │
  │  Total: 12 chunks across 12 nodes            │
  │                                             │
  │  Can lose ANY 4 chunks and still recover!    │
  │  Storage overhead: 12/8 = 1.5× (vs 3×)     │
  │                                             │
  │  Node 1:  [D1]    Node 7:  [D7]            │
  │  Node 2:  [D2]    Node 8:  [D8]            │
  │  Node 3:  [D3]    Node 9:  [P1]            │
  │  Node 4:  [D4]    Node 10: [P2]            │
  │  Node 5:  [D5]    Node 11: [P3]            │
  │  Node 6:  [D6]    Node 12: [P4]            │
  │                                             │
  │  Durability: >99.999999999% (11 nines)      │
  └─────────────────────────────────────────────┘

  Placement: chunks spread across racks/AZs
  to survive rack power failure or AZ outage
```

---

## 5. Metadata Design

```
  Object metadata table:
  ┌───────────────────┬──────────────────────────────┐
  │ Field             │ Example                      │
  ├───────────────────┼──────────────────────────────┤
  │ bucket_id         │ "my-bucket"                  │
  │ object_key        │ "photos/cat.jpg"             │
  │ version_id        │ "v3"                         │
  │ size              │ 5242880                      │
  │ content_type      │ "image/jpeg"                 │
  │ checksum          │ "sha256:abc123..."           │
  │ chunk_map         │ [{node:A, offset:0, len:1MB},│
  │                   │  {node:B, offset:0, len:1MB},│
  │                   │  ...]                        │
  │ created_at        │ "2026-03-12T10:00:00Z"       │
  │ storage_class     │ "STANDARD"                   │
  │ acl               │ {owner: rw, public: r}       │
  └───────────────────┴──────────────────────────────┘

  Index: (bucket_id, object_key) → primary key
  LIST operation: range scan on prefix
  Versioning: (bucket_id, object_key, version_id)
```

---

## 6. Multipart Upload

```
  For large objects (> 100 MB):

  Client                              API
    │                                  │
    │──InitiateMultipartUpload────────▶│ → returns upload_id
    │                                  │
    │──UploadPart(upload_id, part=1)──▶│ → stores part, returns ETag
    │──UploadPart(upload_id, part=2)──▶│   (parallel uploads OK)
    │──UploadPart(upload_id, part=3)──▶│
    │     ...                          │
    │──CompleteMultipartUpload────────▶│ → assembles metadata
    │  (list of part ETags)            │   → returns final ETag
    │                                  │

  Benefits:
  • Parallel part uploads (faster)
  • Resume on failure (only re-upload failed parts)
  • Parts can be different sizes (5MB–5GB)
```

---

## 7. Storage Classes & Lifecycle

```
  ┌─────────────┐    30 days    ┌──────────────┐    90 days   ┌─────────┐
  │  STANDARD   │──────────────▶│ INFREQUENT   │─────────────▶│ GLACIER │
  │  (SSD/HDD)  │               │ ACCESS (IA)  │              │ (tape/  │
  │  $0.023/GB  │               │ $0.0125/GB   │              │ cold)   │
  └─────────────┘               └──────────────┘              │$0.004/GB│
                                                              └─────────┘

  Lifecycle rules:
  • Transition: move to cheaper class after N days
  • Expiration: delete after N days
  • Abort incomplete multipart uploads after N days
```

---

## 8. Consistency Model

```
  Strong read-after-write consistency (S3 since Dec 2020):

  Timeline ──────────────────────────────────▶

  PUT object ──▶ │ committed to all replicas │
                 │                           │
  GET object ──▶ │ always returns latest     │

  Implementation:
  • Metadata write is synchronous (quorum write)
  • Data chunks are durably written before metadata commit
  • Read always goes through metadata service (no stale cache)
```

---

## 9. Scaling Strategy

- **Metadata:** Partition by bucket + key prefix hash; use distributed DB
- **Storage nodes:** Add nodes → background rebalancing redistributes chunks
- **Namespace scaling:** Flat key namespace with prefix-based partitioning
- **Regional:** Multi-region replication for disaster recovery
- **CDN integration:** Cache frequently accessed objects at edge

---

## 10. Observability

- **Metrics:** Request latency, throughput, storage utilization, error rates by bucket
- **Data integrity:** Background scrubbing—verify checksums periodically
- **Alerts:** Disk failures, replication lag, storage capacity thresholds
- **Audit:** Access logs per bucket (who accessed what, when)

---

---

## 11. Low-Level Design (LLD)

### API Contract (S3-Compatible)
```
  PUT    /{bucket}/{key}
         Headers: Content-Type, Content-MD5, x-amz-storage-class
         Body: <object data>
         Response: { "ETag": "md5hash", "VersionId": "v1" }

  GET    /{bucket}/{key}?versionId=v1
         Headers: Range (for partial reads)
         Response: object bytes + metadata headers

  DELETE /{bucket}/{key}?versionId=v1
         Response: 204 No Content (or insert delete marker if versioned)

  HEAD   /{bucket}/{key}   (metadata only, no body)

  LIST   /{bucket}?prefix=logs/2026/&delimiter=/&max-keys=1000
         Response: { "Contents": [...], "CommonPrefixes": [...],
                     "NextContinuationToken": str }

  Multipart:
  POST   /{bucket}/{key}?uploads         → initiate (returns UploadId)
  PUT    /{bucket}/{key}?partNumber=1&uploadId=X  → upload part
  POST   /{bucket}/{key}?uploadId=X       → complete (merge parts)
```

### Internal Storage Layout
```
  Object → split into data chunks + parity (erasure coded):
  ┌───────────────────────────────────────────────────────┐
  │  Object: "logs/2026/app.log" (100 MB)                 │
  │                                                        │
  │  Split into 8 data chunks (12.5 MB each)               │
  │  Generate 4 parity chunks (Reed-Solomon EC 8+4)        │
  │  12 chunks placed across 12 different storage nodes     │
  │                                                        │
  │  Node-1: chunk_0    Node-7:  chunk_6                   │
  │  Node-2: chunk_1    Node-8:  chunk_7                   │
  │  Node-3: chunk_2    Node-9:  parity_0                  │
  │  Node-4: chunk_3    Node-10: parity_1                  │
  │  Node-5: chunk_4    Node-11: parity_2                  │
  │  Node-6: chunk_5    Node-12: parity_3                  │
  │                                                        │
  │  Tolerate any 4 node failures → still reconstruct      │
  │  Storage overhead: 12/8 = 1.5× (vs 3× for replication)│
  └───────────────────────────────────────────────────────┘

  Metadata Store (per object):
  ┌──────────────────────────────────────────┐
  │  bucket_id + key (PK)                    │
  │  version_id, size, content_type, etag    │
  │  chunk_locations: [(node, disk, offset)] │
  │  created_at, storage_class, acl          │
  │  custom_metadata: { user-defined k-v }   │
  └──────────────────────────────────────────┘
```

---

## 12. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Data Plane:                                            │
  │  • Add storage nodes → linear capacity increase         │
  │  • Each node: 12-36 HDDs (100-500 TB raw per node)      │
  │  • Exabyte scale: 10,000+ storage nodes                │
  │                                                         │
  │  Metadata Plane:                                        │
  │  • Metadata DB sharded by hash(bucket + key prefix)    │
  │  • Hot buckets: sub-shard by key prefix automatically   │
  │  • Billions of objects: B-tree or LSM-tree at metadata  │
  │                                                         │
  │  Request Routing:                                       │
  │  • Stateless API gateway: auto-scale horizontally       │
  │  • CDN for frequently accessed public objects           │
  │  • Transfer acceleration: edge PoPs for upload          │
  │                                                         │
  │  Throughput:                                             │
  │  • Single object: 5 Gbps with multipart + parallel     │
  │  • Aggregate: 100+ Gbps per cluster                     │
  │  • Rate limits: 5,500 GET/s, 3,500 PUT/s per prefix   │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. No Data Loss

```
  Durability Target: 99.999999999% (11 nines)

  ┌─────────────────────────────────────────────────────────┐
  │  Write Durability:                                      │
  │  • Data written to W storage nodes before ACK           │
  │  • Each node: fsync to disk before ACK                  │
  │  • Content-MD5 verified on write (end-to-end checksum) │
  │                                                         │
  │  Erasure Coding:                                        │
  │  • (8,4) Reed-Solomon: survive 4 simultaneous failures │
  │  • Placement across failure domains (rack, AZ)          │
  │  • Automatic re-encoding when node fails                │
  │                                                         │
  │  Background Integrity:                                  │
  │  • Bit-rot detection: periodic checksum verification    │
  │  • Scrubbing cycle: verify all chunks every 30 days    │
  │  • Corrupt chunk → auto-repair from remaining chunks   │
  │                                                         │
  │  Versioning:                                             │
  │  • Object versioning preserves all previous versions    │
  │  • Delete creates "delete marker" — previous version   │
  │    still recoverable                                    │
  │  • MFA Delete: require MFA for permanent deletion      │
  │                                                         │
  │  Cross-Region Replication:                               │
  │  • Async replication to secondary region                 │
  │  • RPO: minutes (replication lag)                        │
  │  • Protects against entire datacenter loss              │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Latency

```
  Latency Targets:
  ┌────────────────────────┬──────────┬──────────┐
  │ Operation              │ p50      │ p99      │
  ├────────────────────────┼──────────┼──────────┤
  │ GET (< 1 MB)           │ 10 ms    │ 50 ms   │
  │ GET (1 GB, parallel)   │ 2 s      │ 5 s     │
  │ PUT (< 1 MB)           │ 20 ms    │ 100 ms  │
  │ PUT (multipart 1 GB)   │ 5 s      │ 15 s    │
  │ HEAD (metadata only)   │ 5 ms     │ 20 ms   │
  │ LIST (1000 keys)       │ 50 ms    │ 200 ms  │
  └────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Range reads: parallel fetch of chunks → faster GETs │
  │  • Metadata caching: LRU cache of hot object metadata  │
  │  • CDN integration: serve popular objects from edge     │
  │  • Transfer Acceleration: global edge ingestion points │
  │  • Presigned URLs: direct client ↔ storage (skip proxy)│
  │  • Byte-range fetches: resume interrupted downloads    │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Storage node failure   │ Erasure coding: reconstruct from│
  │                        │ remaining chunks; auto-repair   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Disk failure           │ Detect via SMART; rebuild chunks│
  │                        │ on healthy disks proactively    │
  ├────────────────────────┼─────────────────────────────────┤
  │ Metadata DB failure    │ Synchronous standby failover;   │
  │                        │ multi-AZ metadata replication   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Bit rot / silent       │ Checksums per chunk; scrubbing  │
  │ corruption             │ cycle every 30 days             │
  ├────────────────────────┼─────────────────────────────────┤
  │ Network partition      │ Quorum writes; reject if cannot │
  │                        │ reach min nodes for durability  │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 16. Availability

```
  Target: 99.99% (Standard), 99.9% (Infrequent Access)

  ┌─────────────────────────────────────────────────────────┐
  │  Multi-AZ by Default:                                   │
  │  • Chunks distributed across 3+ AZs                     │
  │  • Any single AZ outage → reads/writes continue         │
  │  • API gateway: stateless, multi-AZ auto-scaling        │
  │                                                         │
  │  Cross-Region (optional):                                │
  │  • Active-passive replication for DR                     │
  │  • Active-active with conflict resolution (version-based)│
  │                                                         │
  │  Maintenance:                                            │
  │  • Rolling node upgrades without downtime                │
  │  • Background re-encoding doesn't affect serving         │
  │  • Capacity expansion: add nodes, redistribute chunks   │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Access Control:                                        │
  │  • Bucket policies (resource-based JSON policies)       │
  │  • ACLs (per-object: private, public-read, etc.)        │
  │  • IAM policies (identity-based, attached to users/roles)│
  │  • Presigned URLs with expiry for temporary access      │
  │                                                         │
  │  Encryption:                                            │
  │  • In-transit: TLS 1.3 mandatory                        │
  │  • At-rest: SSE-S3 (managed keys), SSE-KMS (customer   │
  │    managed), SSE-C (customer-provided)                  │
  │  • Client-side encryption for end-to-end security       │
  │                                                         │
  │  Data Protection:                                       │
  │  • Object Lock: WORM compliance (Write Once Read Many)  │
  │  • Versioning + MFA Delete                              │
  │  • Cross-account access control                         │
  │  • VPC endpoints (private network access, no internet)  │
  │                                                         │
  │  Audit & Compliance:                                    │
  │  • Server access logging (every request logged)         │
  │  • CloudTrail integration for API call auditing         │
  │  • GDPR: lifecycle policies for data retention/deletion │
  └─────────────────────────────────────────────────────────┘
```

---

## 18. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Storage Classes (cost/GB/month):                       │
  │  • Standard:         $0.023/GB (frequent access)        │
  │  • Infrequent Access:$0.0125/GB (retrieval fee applies) │
  │  • Glacier:          $0.004/GB (min 90-day retention)   │
  │  • Deep Archive:     $0.00099/GB (12h retrieval)        │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Lifecycle policies: auto-transition to cheaper tiers │
  │  • Intelligent tiering: auto-move based on access       │
  │  • Erasure coding vs replication: 50% storage savings  │
  │  • Multipart upload: avoid re-uploading on failure      │
  │  • Compression at application layer before PUT          │
  │                                                         │
  │  Cost Estimate (1 PB storage):                          │
  │  • Standard: ~$23K/month                                │
  │  • 70% cold (Glacier): ~$6K/month                      │
  │  • Requests: ~$5K/month (at 100M GET/month)            │
  │  • Egress: ~$10K/month (100 TB out)                    │
  │  • With lifecycle optimization: 60-70% savings         │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Why erasure coding over replication?** — 1.5× overhead vs 3×; saves massive cost at exabyte scale
2. **How do you handle billions of objects in LIST?** — Prefix-based partitioning in metadata DB; pagination with continuation tokens
3. **How do you guarantee 11 nines durability?** — Erasure coding + cross-AZ placement + background integrity checks + auto-repair
4. **How does S3 achieve strong consistency?** — Synchronous metadata writes with quorum; data committed before metadata
5. **Hot bucket problem?** — Partition bucket metadata by key prefix; distribute request load across nodes
