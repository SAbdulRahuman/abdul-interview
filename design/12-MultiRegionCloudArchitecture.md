# Design a Multi-Region Cloud Architecture

---

## 1. Requirements

### Functional
- Serve users globally with low latency
- Replicate data across regions for disaster recovery
- Automatic failover when a region goes down
- Consistent user experience regardless of region

### Non-Functional
- < 100ms latency for users worldwide
- RPO ≤ 1 minute, RTO ≤ 5 minutes
- 99.999% availability (< 5 min downtime/year)
- Cost-efficient (avoid unnecessary cross-region traffic)

---

## 2. High-Level Architecture

```
                        ┌──────────────────┐
                        │  Global DNS      │
                        │  (Route 53 /     │
                        │   Cloudflare)    │
                        │  Geo-routing     │
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
     │  US-East Region  │ │  EU-West Region  │ │  AP-South Region │
     │                  │ │                  │ │                  │
     │ ┌──────────────┐ │ │ ┌──────────────┐ │ │ ┌──────────────┐ │
     │ │  CDN Edge    │ │ │ │  CDN Edge    │ │ │ │  CDN Edge    │ │
     │ └──────┬───────┘ │ │ └──────┬───────┘ │ │ └──────┬───────┘ │
     │ ┌──────▼───────┐ │ │ ┌──────▼───────┐ │ │ ┌──────▼───────┐ │
     │ │  Load        │ │ │ │  Load        │ │ │ │  Load        │ │
     │ │  Balancer    │ │ │ │  Balancer    │ │ │ │  Balancer    │ │
     │ └──────┬───────┘ │ │ └──────┬───────┘ │ │ └──────┬───────┘ │
     │ ┌──────▼───────┐ │ │ ┌──────▼───────┐ │ │ ┌──────▼───────┐ │
     │ │  App Servers │ │ │ │  App Servers │ │ │ │  App Servers │ │
     │ └──────┬───────┘ │ │ └──────┬───────┘ │ │ └──────┬───────┘ │
     │ ┌──────▼───────┐ │ │ ┌──────▼───────┐ │ │ ┌──────▼───────┐ │
     │ │  Database    │ │ │ │  Database    │ │ │ │  Database    │ │
     │ │  (primary)   │◀┼─┼─▶  (replica)  │◀┼─┼─▶  (replica)  │ │
     │ └──────────────┘ │ │ └──────────────┘ │ │ └──────────────┘ │
     └─────────────────┘ └─────────────────┘ └─────────────────┘
                   ▲              ▲              ▲
                   └──── async replication ───────┘
```

---

## 3. Traffic Routing

```
  ┌─────────────────────────────────────────────────────┐
  │  DNS-Based Geo Routing:                              │
  │                                                     │
  │  User in London → DNS returns EU-West LB IP         │
  │  User in Tokyo  → DNS returns AP-South LB IP        │
  │  User in NYC    → DNS returns US-East LB IP         │
  │                                                     │
  │  Health-based failover:                             │
  │  EU-West health check fails →                       │
  │  DNS routes EU users to US-East (next closest)      │
  │                                                     │
  │  Latency-based routing:                             │
  │  When regions overlap, route by measured latency     │
  ├─────────────────────────────────────────────────────┤
  │  Global Load Balancer (Anycast):                    │
  │                                                     │
  │  Single IP announced from all regions               │
  │  BGP routes to nearest region automatically          │
  │  Faster failover than DNS (no TTL delay)            │
  └─────────────────────────────────────────────────────┘
```

---

## 4. Data Replication Patterns

```
  Pattern 1: Single Leader (Write to one region)
  ┌──────────┐    async     ┌──────────┐    async     ┌──────────┐
  │ US-East  │─────────────▶│ EU-West  │─────────────▶│ AP-South │
  │ (Leader) │              │ (Follow) │              │ (Follow) │
  │  R + W   │              │  R only  │              │  R only  │
  └──────────┘              └──────────┘              └──────────┘
  Pro: Simple, strong consistency for writes
  Con: Write latency for non-US users, single point of failure for writes

  Pattern 2: Multi-Leader (Write to any region)
  ┌──────────┐              ┌──────────┐              ┌──────────┐
  │ US-East  │◀────────────▶│ EU-West  │◀────────────▶│ AP-South │
  │ (Leader) │   conflict   │ (Leader) │   conflict   │ (Leader) │
  │  R + W   │  resolution  │  R + W   │  resolution  │  R + W   │
  └──────────┘              └──────────┘              └──────────┘
  Pro: Low write latency everywhere
  Con: Write conflicts need resolution (LWW, CRDTs, app-level merge)

  Pattern 3: Partitioned by Region
  ┌──────────┐              ┌──────────┐              ┌──────────┐
  │ US-East  │              │ EU-West  │              │ AP-South │
  │ US users │              │ EU users │              │ AP users │
  │  R + W   │              │  R + W   │              │  R + W   │
  └──────────┘              └──────────┘              └──────────┘
  Pro: No conflicts, data sovereignty compliance
  Con: Cross-region queries are expensive
```

---

## 5. Failover Process

```
  Region failure detected:

  Time 0s:    Health checks fail (3 consecutive failures)
  Time 10s:   Declare region unhealthy
  Time 15s:   DNS update → remove failed region from routing
  Time 60s:   DNS propagation (TTL-dependent)
  Time 60s:   Promote replica to primary (if single-leader)
  Time 90s:   Traffic fully redirected to surviving regions

  Automated failover checklist:
  ┌──────────────────────────────────────────────┐
  │  1. Detect failure (health checks, monitors) │
  │  2. Promote read replica to primary          │
  │  3. Update DNS / load balancer rules         │
  │  4. Verify data consistency                  │
  │  5. Scale up surviving regions (handle load)│
  │  6. Alert operations team                    │
  │  7. After recovery: resync and failback      │
  └──────────────────────────────────────────────┘
```

---

## 6. Consistency vs Latency Trade-offs

```
  ┌──────────────────────────────────────────────┐
  │  Synchronous replication across regions:     │
  │  Write latency = inter-region RTT (~100ms+)  │
  │  Consistency: strong                         │
  │  Use for: financial transactions, auth       │
  │                                              │
  │  Asynchronous replication:                   │
  │  Write latency = local (~5ms)                │
  │  Consistency: eventual (seconds of lag)      │
  │  Use for: social feeds, analytics, logs      │
  │                                              │
  │  Hybrid approach:                            │
  │  Critical data (payments) → sync replication │
  │  Non-critical (user prefs) → async           │
  └──────────────────────────────────────────────┘
```

---

## 7. Cost Optimization

- **Data egress:** Cross-region transfer costs $0.02–0.09/GB → minimize by caching, compression
- **CDN caching:** Cache static assets at edge → reduce origin requests
- **Active-passive vs active-active:** Active-passive DR region costs less (reduced compute)
- **Data gravity:** Store data where it's processed most → reduce egress
- **Reserved capacity:** Use reserved instances in primary region, spot/on-demand for DR

---

---

## 8. Low-Level Design (LLD)

### API Contract (Region-Aware)
```
  Global Request Flow:
  GET    /api/v1/resource/{id}
         Header: X-Region-Preference: eu-west-1
         Response: { "data": {...}, "region": "eu-west-1", "consistency": "eventual" }

  Strong Consistency Mode:
  POST   /api/v1/resource
         Header: X-Consistency: strong
         → Write to leader region, replicate synchronously
         → Response only after quorum regions ACK

  Conflict Resolution:
  PUT    /api/v1/resource/{id}
         Header: If-Match: etag_value
         Body: { "data": {...}, "region_origin": "us-east-1" }
         409 Conflict → client re-reads and merges
```

### Data Replication Internals
```
  Async Replication (Default Path):
  ┌──────────────────────────────────────────────────────────┐
  │  Write → Primary DB (us-east-1)                         │
  │    → Binlog/WAL stream                                  │
  │    → Change Data Capture (Debezium / DMS)               │
  │    → Kafka (cross-region MirrorMaker 2)                 │
  │    → Consumer in eu-west-1 applies to local DB           │
  │  Lag: 200ms - 2s typical                                │
  └──────────────────────────────────────────────────────────┘

  Conflict Resolution (Multi-Master):
  ┌──────────────────────────────────────────────────────────┐
  │  CockroachDB / Spanner Pattern:                         │
  │  • MVCC + Hybrid Logical Clock (HLC)                    │
  │  • Each write gets globally ordered timestamp            │
  │  • Conflict: Last-Writer-Wins (LWW) or app-level CRDT  │
  │                                                          │
  │  DynamoDB Global Tables:                                 │
  │  • LWW with wall-clock timestamp                         │
  │  • Last region to write wins on conflict                 │
  │  • Custom conflict handler via Lambda                    │
  └──────────────────────────────────────────────────────────┘

  DNS Failover Internals:
  ┌──────────────────────────────────────────────────────────┐
  │  Route53 Health Check:                                   │
  │    Probe: HTTPS GET /health every 10s from 8 locations  │
  │    Threshold: 3 consecutive failures → unhealthy        │
  │    Failover: update DNS record (TTL 60s)                │
  │    Total detection + switch: ~90s                        │
  │                                                          │
  │  Anycast (CloudFlare / Global Accelerator):             │
  │    BGP announces same IP from multiple regions           │
  │    Failover: BGP withdrawal, ~10-30s                     │
  └──────────────────────────────────────────────────────────┘
```

---

## 9. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Per-Region Scaling:                                    │
  │  • Each region independently auto-scales               │
  │  • Stateless services: HPA / auto-scale groups          │
  │  • Database: read replicas within region                │
  │                                                         │
  │  Adding a New Region:                                   │
  │  1. Deploy infrastructure via IaC (Terraform)           │
  │  2. Seed database from snapshot of existing region      │
  │  3. Enable CDC replication from primary                 │
  │  4. Gradual traffic shift via weighted DNS              │
  │  5. Full active in ~2-4 hours                           │
  │                                                         │
  │  Data Partitioning:                                     │
  │  • Geo-partition: EU data in EU region, US in US        │
  │  • Global data (config, catalog): replicated everywhere │
  │  • User data: hashed to home region, cached in others   │
  │                                                         │
  │  CDN:                                                   │
  │  • Static assets: CloudFront / Akamai (200+ PoPs)      │
  │  • API responses: cache with Vary headers               │
  │  • Edge compute: Lambda@Edge for region routing         │
  └─────────────────────────────────────────────────────────┘
```

---

## 10. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Database Durability:                                   │
  │  • Synchronous replication to at least 1 standby AZ    │
  │  • Async cross-region replication (RPO: 1-5s)          │
  │  • Point-in-time recovery: continuous WAL archiving     │
  │  • Automated daily snapshots + cross-region copy        │
  │                                                         │
  │  Object Store:                                          │
  │  • S3 cross-region replication (CRR)                    │
  │  • Versioning enabled: protection from accidental delete│
  │  • MFA Delete for critical buckets                      │
  │                                                         │
  │  Event Bus:                                             │
  │  • Kafka MirrorMaker 2: cross-region topic replication  │
  │  • Consumer lag monitoring: alert if lag > threshold    │
  │  • Dead letter queue for failed event processing        │
  │                                                         │
  │  RPO by Tier:                                           │
  │  • Tier-1 (payments): 0 (sync replication)              │
  │  • Tier-2 (user data): < 5s (async + CDC)               │
  │  • Tier-3 (analytics): < 1 hour (batch replication)     │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. Latency

```
  Latency Targets:
  ┌──────────────────────────────┬──────────┬──────────┐
  │ Operation                   │ p50      │ p99      │
  ├──────────────────────────────┼──────────┼──────────┤
  │ Read (local region)         │ 5 ms     │ 20 ms   │
  │ Read (cross-region, sync)   │ 80 ms    │ 200 ms  │
  │ Write (local, async repl)   │ 10 ms    │ 50 ms   │
  │ Write (global, sync repl)   │ 100 ms   │ 300 ms  │
  │ Failover (DNS-based)        │ —        │ 90 s    │
  │ Failover (anycast/BGP)      │ —        │ 30 s    │
  └──────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Route to nearest region (GeoDNS / latency routing)  │
  │  • Local read replicas: avoid cross-region reads       │
  │  • Session affinity: pin user to closest region        │
  │  • Edge caching: cache popular content at PoP           │
  │  • Pre-warm connections across regions (connection pool)│
  │  • Async replication for writes that tolerate delay     │
  │  • Compression for cross-region data transfer          │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Region outage          │ DNS failover to secondary;      │
  │                        │ <90s RTO with health checks     │
  ├────────────────────────┼─────────────────────────────────┤
  │ Replication lag spike  │ Read-your-own-writes: route     │
  │                        │ recent writes to primary        │
  ├────────────────────────┼─────────────────────────────────┤
  │ Split-brain (network)  │ Fencing: only one region can    │
  │                        │ accept writes (lease-based)     │
  ├────────────────────────┼─────────────────────────────────┤
  │ Data corruption        │ Checksums, cross-region compare │
  │                        │ Restore from backup / PITR      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Cascading failure      │ Circuit breakers per region;    │
  │                        │ bulkhead isolation               │
  └────────────────────────┴─────────────────────────────────┘

  DR Testing:
  • Monthly GameDay: simulate region failure
  • Chaos Monkey / Gremlin: inject faults
  • Runbook automation: all failover steps scripted
```

---

## 13. Availability

```
  Target: 99.999% (five nines) — ~5 min downtime/year

  ┌─────────────────────────────────────────────────────────┐
  │  Architecture:                                          │
  │  • Active-Active: 2+ regions serving traffic            │
  │  • Each region independently functional                 │
  │  • No single point of failure across regions            │
  │                                                         │
  │  Availability Math:                                     │
  │  • Single region: 99.99% (52 min/year downtime)         │
  │  • Two active regions: 1 - (0.0001)² = 99.999999%      │
  │  • Limited by failover time, not region availability    │
  │                                                         │
  │  Maintenance Without Downtime:                          │
  │  • Drain region → shift traffic → upgrade → return      │
  │  • Database schema changes: online DDL (pt-osc, gh-ost) │
  │  • Feature flags: enable/disable per region             │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Data Sovereignty (GDPR / Data Residency):              │
  │  • Geo-partition user data by region of origin          │
  │  • EU data never leaves EU regions                       │
  │  • Data residency tags on every record                  │
  │  • Audit trail for cross-region data access             │
  │                                                         │
  │  Network Security (Cross-Region):                       │
  │  • VPC peering or Transit Gateway (private links)       │
  │  • All cross-region traffic: TLS 1.3 encrypted          │
  │  • No data crosses public internet                      │
  │  • AWS PrivateLink / Azure Private Endpoint              │
  │                                                         │
  │  Authentication:                                        │
  │  • Global IAM with region-local token validation        │
  │  • JWT with region claim: restrict API to user's region │
  │  • Cross-region service-to-service: mTLS + SPIFFE       │
  │                                                         │
  │  Key Management:                                        │
  │  • Per-region KMS (AWS KMS, GCP CMEK)                    │
  │  • Key material never exported cross-region             │
  │  • Envelope encryption: data key per record             │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Multi-Region Premium:                                  │
  │  • 2× compute (duplicate stack per region)              │
  │  • Data transfer: $0.02/GB inter-region                 │
  │  • Database: 2× for replicas, +30% for sync replication │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Active-passive secondary: minimal compute until DR   │
  │  • Asymmetric capacity: primary 100%, secondary 50%     │
  │    (auto-scale to 100% during failover)                 │
  │  • Reserved instances for primary, on-demand for DR      │
  │  • Cross-region transfer: compress, batch, use S3 CRR   │
  │                                                         │
  │  Cost Estimate (medium-scale web platform):             │
  │  • Single region: ~$30K/month                           │
  │  • Two active regions: ~$55K/month (+83%)               │
  │  • Active-passive: ~$42K/month (+40%)                   │
  │  • Data transfer between regions: ~$2K/month            │
  │  • CDN: ~$3K/month (but saves origin compute)           │
  │                                                         │
  │  ROI Justification:                                     │
  │  • 1 hour outage cost for e-commerce: $100K-$1M         │
  │  • Multi-region insurance: $12K/month premium           │
  │  • Break-even: prevents ~2 hours of outage per year     │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How do you handle data sovereignty (GDPR)?** — Region-partitioned data, ensure EU data stays in EU
2. **How do you handle split-brain?** — Quorum across regions, or accept temporary inconsistency with conflict resolution
3. **DNS TTL and failover speed trade-off?** — Lower TTL = faster failover but more DNS queries (cost + performance)
4. **Active-active vs active-passive?** — Active-active: lower latency, harder consistency. Active-passive: simpler, higher RTO
5. **How to test DR without disrupting production?** — Chaos engineering (simulate region failure), read-only DR tests, GameDay exercises
