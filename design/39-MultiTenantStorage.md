# Design a Multi-Tenant Storage Platform

## Overview
Design a multi-tenant storage platform with strict QoS isolation — preventing noisy neighbors, guaranteeing per-tenant IOPS/throughput minimums, and providing fair resource sharing across hundreds of tenants on shared infrastructure, similar to NetApp ONTAP QoS or cloud provider storage systems.

## 1. Requirements

**Functional:**
- Per-tenant/volume QoS policies: min/max IOPS, min/max throughput, max latency
- Adaptive QoS: limits scale with volume size
- Workload classification: latency-sensitive vs throughput-oriented
- Resource pools: allocate physical resources to tenant groups
- QoS ceiling enforcement: hard limit for IOPS bursting
- Fair-share scheduling when resources are contended

**Non-Functional:**
- QoS enforcement accuracy: ±5% of configured limits
- Noisy neighbor impact: <10% latency increase on co-tenants
- Latency overhead from QoS: <50μs per I/O
- Scale: 1,000 QoS policies per controller

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Tenants** | 200 per storage cluster |
| **Volumes per tenant** | 50 avg (10,000 total) |
| **Aggregate IOPS** | 2M per HA pair |
| **IOPS per tenant (avg)** | 10K (range: 1K-200K) |
| **Bandwidth per tenant** | 500 MB/s (range: 50MB/s-5GB/s) |
| **QoS policies** | 500 active |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│             Multi-Tenant Storage with QoS                         │
│                                                                   │
│  Tenant A (Gold)    Tenant B (Silver)   Tenant C (Bronze)        │
│  ┌────────────┐     ┌────────────┐      ┌────────────┐           │
│  │ App Servers │     │ App Servers │      │ App Servers │           │
│  │ NFS/iSCSI  │     │ NFS/iSCSI  │      │ S3/NFS     │           │
│  └─────┬──────┘     └─────┬──────┘      └─────┬──────┘           │
│        │                  │                    │                   │
│  ┌─────┴──────────────────┴────────────────────┴───────────┐     │
│  │                 Storage Controller                       │     │
│  │                                                          │     │
│  │  ┌──────────────────────────────────────────────────┐   │     │
│  │  │            QoS Scheduler                          │   │     │
│  │  │                                                    │   │     │
│  │  │  ┌───────────┐ ┌───────────┐ ┌───────────┐      │   │     │
│  │  │  │ Policy A  │ │ Policy B  │ │ Policy C  │      │   │     │
│  │  │  │ min:50K   │ │ min:10K   │ │ min:2K    │      │   │     │
│  │  │  │ max:200K  │ │ max:50K   │ │ max:10K   │      │   │     │
│  │  │  │ IOPS      │ │ IOPS      │ │ IOPS      │      │   │     │
│  │  │  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘      │   │     │
│  │  │        │              │              │            │   │     │
│  │  │  ┌─────┴──────────────┴──────────────┴─────┐     │   │     │
│  │  │  │       Token Bucket / Fair Queue          │     │   │     │
│  │  │  │   Dispatch rate-limited I/Os to disks    │     │   │     │
│  │  │  └──────────────────────────────────────────┘     │   │     │
│  │  └──────────────────────────────────────────────────┘   │     │
│  │                                                          │     │
│  │  ┌───────────────────────────────────────────────┐      │     │
│  │  │ SVM (Storage Virtual Machine) per Tenant      │      │     │
│  │  │                                                │      │     │
│  │  │ SVM-A: Volumes [v1,v2,...v20]                  │      │     │
│  │  │ SVM-B: Volumes [v1,v2,...v30]                  │      │     │
│  │  │ SVM-C: Volumes [v1,v2,...v15]                  │      │     │
│  │  │                                                │      │     │
│  │  │ Each SVM: own namespace, ACLs, network (LIFs) │      │     │
│  │  └───────────────────────────────────────────────┘      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Shared Physical Resources                                │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │ Aggregate 1  │  │ Aggregate 2  │  │ Aggregate 3  │   │    │
│  │  │ 50TB NVMe    │  │ 100TB SSD    │  │ 200TB HDD    │   │    │
│  │  │ Gold tenants │  │ Silver       │  │ Bronze        │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  └──────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

## 4. QoS Scheduler Design

```
Token Bucket Rate Limiter (per QoS policy):

┌──────────────────────────────────────────────────────┐
│  QoS Policy: "gold_db"                               │
│                                                      │
│  Parameters:                                         │
│    min_iops: 50,000    (guaranteed minimum)           │
│    max_iops: 200,000   (hard ceiling)                │
│    min_throughput: 500 MB/s                          │
│    max_throughput: 2 GB/s                            │
│    max_latency: 1ms    (not enforced, monitoring)    │
│                                                      │
│  Token Bucket:                                       │
│    bucket_size = max_iops / refill_rate_hz           │
│    tokens_per_second = max_iops                      │
│    burst_allowance = bucket_size × 1.2               │
│                                                      │
│  Each I/O:                                           │
│    if tokens > 0:                                    │
│      consume_token()                                 │
│      dispatch_io()                                   │
│    else:                                             │
│      queue_io()  → dispatch when tokens refill       │
│      → adds latency (queuing delay)                  │
└──────────────────────────────────────────────────────┘

Fair-Share Scheduling (when total demand > capacity):

class FairShareScheduler:
  def schedule(self):
    total_capacity = 2_000_000  # IOPS
    total_min_reserved = sum(p.min_iops for p in policies)
    spare = total_capacity - total_min_reserved
    
    for policy in policies:
      # Phase 1: guarantee minimums
      policy.allocated = policy.min_iops
      
      # Phase 2: distribute spare fairly
      if policy.demand > policy.min_iops:
        weight = policy.weight  # Gold=10, Silver=5, Bronze=1
        policy.allocated += spare * (weight / total_weight)
      
      # Phase 3: cap at maximum
      policy.allocated = min(policy.allocated, policy.max_iops)
    
    # Result: Gold gets more spare capacity; all get minimums

Noisy Neighbor Detection:
  Monitor per-policy:
    - actual_iops vs allocated_iops
    - queue_depth (I/Os waiting)
    - latency_contribution_us
    
  If tenant exceeds max:
    → throttled (token bucket empty → I/Os queued)
    → no impact on other tenants
    
  If total system at capacity:
    → minimums guaranteed via fair-share
    → spare distributed by weight
```

## 5. Storage Virtual Machine (SVM) Isolation

```
SVM-Based Multi-Tenancy:

┌──────────────────────────────────────────────────┐
│ SVM "tenant-a" (Storage Virtual Machine):        │
│                                                  │
│ Network Isolation:                               │
│   LIF: 10.1.1.100 (data), 10.1.1.101 (data)    │
│   VLAN: 100                                      │
│   Routing: isolated routing table                │
│                                                  │
│ Storage Isolation:                               │
│   Volumes: vol1 (500GB), vol2 (1TB), ...        │
│   Aggregates: assigned to aggr1 (shared phys)   │
│   Quota: max 10TB across all volumes             │
│                                                  │
│ Protocol Isolation:                              │
│   NFS exports: own export policies               │
│   CIFS shares: own AD domain                     │
│   iSCSI: own IQN, own igroups                   │
│                                                  │
│ Security Isolation:                              │
│   Own admin users (delegated admin)              │
│   Own audit log                                  │
│   Own encryption keys (per-SVM key)              │
│   Cannot see other SVMs' data or config          │
│                                                  │
│ QoS:                                             │
│   Policy group "gold_db" applied to volumes      │
│   SVM-level aggregate QoS ceiling                │
└──────────────────────────────────────────────────┘

Admin Hierarchy:
  Cluster Admin → can see all SVMs, manage physical
  SVM Admin    → can manage own SVM only (volumes, shares, users)
  Tenant User  → can access data via protocols (NFS, CIFS, S3)
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Create QoS Policy Group
POST /api/v1/qos/policies
{
  "name": "gold_database",
  "svm": "tenant-a",
  "fixed": {
    "min_iops": 50000,
    "max_iops": 200000,
    "min_throughput_mbps": 500,
    "max_throughput_mbps": 2000
  }
}

# Create Adaptive QoS Policy (scales with size)
POST /api/v1/qos/adaptive-policies
{
  "name": "standard_per_tb",
  "expected_iops_per_tb": 6000,
  "peak_iops_per_tb": 12000,
  "peak_allocation": "allocated_space",  // used_space | allocated_space
  "absolute_min_iops": 1000
}
# For 5TB volume: expected=30K, peak=60K IOPS

# Create SVM (Tenant)  
POST /api/v1/svms
{
  "name": "tenant-bravo",
  "aggregates": ["aggr1", "aggr2"],     // allowed aggregates
  "max_volumes": 100,
  "protocols": ["nfs", "cifs"],
  "qos_policy": "silver_default",
  "admin_password": "delegated-admin-pass"
}

# Get QoS Workload Statistics
GET /api/v1/qos/workloads/{workload_id}/stats
→ {
  "name": "vol_db1-read",
  "iops": 45000,
  "throughput_mbps": 350,
  "latency_us": 420,
  "service_time_us": 200,
  "wait_time_us": 220,         // QoS queuing delay
  "visits": 1.2,               // avg disk visits per I/O
  "concurrency": 32
}

# Set SVM Resource Limits
PATCH /api/v1/svms/{svm_uuid}/limits
{
  "max_storage_gb": 10240,
  "max_volumes": 100,
  "max_lifs": 8,
  "max_iops": 500000           // SVM-level ceiling
}
```

### I/O Scheduling Internals

```
Multi-Level QoS Enforcement:

Level 1: SVM ceiling
  Total I/O from all volumes in SVM ≤ SVM max

Level 2: Policy group ceiling  
  Total I/O from all volumes sharing policy ≤ policy max

Level 3: Volume workload
  Individual volume I/O governed by assigned policy

I/O Classification:
  class IOClassifier:
    def classify(self, io):
      if io.size <= 4096 and io.sequential == False:
        return "random_small"       # IOPS-bound
      elif io.size >= 65536:
        return "sequential_large"   # throughput-bound
      else:
        return "mixed"
    
    def cost(self, io):
      # Normalize different I/O types to "IOPS equivalents"
      if io.type == "random_small":
        return 1                    # 1 IOPS = 1 4KB random
      elif io.type == "sequential_large":
        return io.size / 4096       # 64KB = 16 IOPS equivalent
      else:
        return io.size / 4096

Scheduling Priority Queue:
  ┌─────────────────────────────────────────────────┐
  │  Priority Queue (min-heap by deadline)          │
  │                                                 │
  │  Each I/O has:                                  │
  │    virtual_time = tokens_consumed / weight      │
  │    deadline = submit_time + max_latency         │
  │                                                 │
  │  Dispatch order:                                │
  │    1. I/Os below minimum guarantee (starving)   │
  │    2. I/Os approaching deadline                 │
  │    3. I/Os from higher-weight policies          │
  │    4. I/Os within burst allowance               │
  │                                                 │
  │  Result: guaranteed mins, fair surplus, caps    │
  └─────────────────────────────────────────────────┘
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Tenants** | SVM per tenant; 200 SVMs per cluster | 200 tenants/cluster |
| **Volumes** | 10K volumes across all SVMs | With FlexGroup: PB-scale volumes |
| **QoS policies** | Hierarchical: SVM ceiling → policy → volume | 1,000 policies/controller |
| **IOPS** | All-flash aggregates; QoS ensures fair distribution | 2M IOPS per HA pair |
| **Clusters** | Scale-out: up to 12 HA pairs per cluster | 24M aggregate IOPS |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Tenant data isolation** | SVM-level volume separation; no cross-SVM data access possible |
| **QoS misconfiguration** | Validation: sum of min_iops ≤ system capacity; prevent overcommit of minimums |
| **Volume deletion** | Per-SVM recycle bin; admin approval for permanent delete |
| **Tenant offboarding** | Crypto-shredding: destroy SVM encryption key → data unreadable |

## 9. Latency

| Scenario | Tenant A (Gold) | Tenant B (Silver) | Tenant C (Bronze) |
|----------|-----------------|--------------------|--------------------|
| **Idle system** | 200μs | 200μs | 200μs |
| **System at 50%** | 200μs | 250μs | 300μs |
| **System at 90%** | 300μs | 500μs | 2ms |
| **System at 100%** | 500μs (min guaranteed) | 1ms (min guaranteed) | 5ms (throttled) |

**Key insight:** Gold tenants see minimal latency increase under contention due to higher minimum guarantees and priority scheduling.

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **QoS scheduler crash** | All tenants unthrottled briefly | Default to fair-share; restart scheduler in <1s |
| **SVM failure** | One tenant's volumes inaccessible | SVM failover to partner controller; volumes accessible on partner |
| **Noisy neighbor burst** | Temporary latency spike for co-tenants | Token bucket enforces ceiling within 1ms; spike contained |
| **Aggregate full** | All tenants on that aggregate affected | Per-SVM space guarantees; monitoring; autogrow aggregate |

## 11. Availability

**Target: 99.999% per tenant**

```
Tenant Isolation for Availability:
  - SVM failover: independent per tenant; one tenant's failover doesn't affect others
  - Controller upgrade: NDU (non-disruptive); SVM migrated transparently
  - Network isolation: per-SVM LIFs; one tenant's broadcast storm doesn't flood others
  - Blast radius: failure impacts single SVM, not all tenants
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Tenant isolation** | SVM: separate namespace, network, admin, audit, encryption keys |
| **Network isolation** | Per-SVM VLANs; IPspace separation; firewall policies per LIF |
| **Data encryption** | Per-SVM encryption keys (NVE); tenant-managed or platform-managed |
| **Admin delegation** | SVM admin cannot see or affect other SVMs; cluster admin for physical |
| **Audit** | Per-SVM audit logs; cannot be disabled by SVM admin |
| **Compliance** | Per-SVM SnapLock volumes for regulatory retention |

## 13. Cost Constraints

**Estimated Cost (200 tenants, 500TB usable):**

| Component | Cost |
|-----------|------|
| **Physical infrastructure** | 500TB all-flash HA pair: $1.2M CapEx |
| **Software (multi-tenant features)** | $80,000/year |
| **Management overhead** | 0.5 FTE for 200 tenants (self-service via SVM admin) |
| **Per-tenant cost** | $1.2M / 200 / 36 months = **~$167/tenant/month** (infra only) |

**Pricing model:**
| Tier | Min IOPS | Storage | Price/month |
|------|----------|---------|-------------|
| Bronze | 2K | 500GB | $200 |
| Silver | 10K | 2TB | $800 |
| Gold | 50K | 10TB | $3,000 |

## Key Interview Discussion Points

1. **How to prevent noisy neighbors?** — Token bucket rate limiting per policy group; minimum guarantees via fair-share scheduler; physical separation for ultra-sensitive tenants
2. **Adaptive QoS: why?** — Fixed IOPS limits don't scale with data growth. Adaptive: 6K IOPS/TB means a 10TB volume gets 60K IOPS automatically. Simpler to manage
3. **What if sum of minimums exceeds capacity?** — Overcommit minimums cautiously (statistical multiplexing — not all tenants peak simultaneously). Alert if sustained demand exceeds capacity
4. **SVM vs namespace isolation?** — SVM: complete isolation (network, admin, keys, protocols). Namespace: shared SVM, folder-level separation. SVM is stronger, more resource overhead
5. **How to handle tenant migration?** — Volume move between aggregates (non-disruptive); SVM migrate between clusters (planned downtime for IP change or DNS update)
