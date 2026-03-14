# Design a Hybrid Cloud Storage System

## Overview
Design a hybrid cloud storage platform like NetApp Data Fabric / Cloud Volumes — enabling seamless data mobility between on-premises and multiple cloud providers with unified management, policy-driven tiering, and consistent data services.

## 1. Requirements

**Functional:**
- Unified namespace across on-prem and cloud (single pane of glass)
- Policy-driven data tiering: hot (on-prem SSD) → warm (on-prem HDD) → cold (cloud object storage)
- Cloud bursting: extend on-prem volumes to cloud during peak demand
- Multi-cloud support: AWS, Azure, GCP with cloud-native integration
- Data mobility: migrate/replicate workloads between on-prem and cloud
- Consistent snapshots, clones, and replication across hybrid boundary

**Non-Functional:**
- Tiering latency for cold recall: <5s for first byte
- Cloud upload throughput: saturate available bandwidth
- Availability: 99.99% for hybrid management plane
- Data sovereignty: policy enforcement for geo-restricted data

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **On-prem capacity** | 2PB across 10 HA pairs |
| **Cloud-tiered data** | 5PB (cold data in S3/Blob/GCS) |
| **Active workloads** | 500TB hot data on-prem |
| **Daily tier movement** | 2TB cold→cloud, 500GB recalled |
| **Multi-cloud targets** | 3 clouds + 2 on-prem sites |
| **Managed relationships** | 2,000 (replication + tiering) |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│              Hybrid Cloud Storage Architecture                    │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │         Unified Management / Control Plane               │     │
│  │  ┌────────────┐ ┌──────────┐ ┌─────────────────────┐    │     │
│  │  │ Cloud Mgr  │ │ Tiering  │ │ Replication Orch.   │    │     │
│  │  │ (SaaS)     │ │ Policy   │ │ (SnapMirror Cloud)  │    │     │
│  │  └────────────┘ └──────────┘ └─────────────────────┘    │     │
│  └───────────────────────┬──────────────────────────────────┘     │
│                          │ API                                    │
│    On-Premises           │              Cloud                     │
│  ┌───────────────────┐   │   ┌──────────────────────────────┐    │
│  │                   │   │   │  AWS                          │    │
│  │  ┌─────────────┐  │   │   │  ┌──────────────────────┐    │    │
│  │  │ ONTAP       │  │   │   │  │ Cloud Volumes ONTAP  │    │    │
│  │  │ Controller  │──┼───┼───┼─►│ (CVO on EC2)         │    │    │
│  │  │             │  │   │   │  │ + EBS/S3 backend     │    │    │
│  │  │ ┌─────────┐ │  │   │   │  └──────────────────────┘    │    │
│  │  │ │Hot: SSD │ │  │   │   │  ┌──────────────────────┐    │    │
│  │  │ │Warm: HDD│ │  │   │   │  │ S3 (cold tier)       │    │    │
│  │  │ │Cold ────┼─┼──┼───┼───┼─►│ Lifecycle policies   │    │    │
│  │  │ └─────────┘ │  │   │   │  └──────────────────────┘    │    │
│  │  └─────────────┘  │   │   └──────────────────────────────┘    │
│  │                   │   │                                        │
│  └───────────────────┘   │   ┌──────────────────────────────┐    │
│                          │   │  Azure                        │    │
│                          └──►│  ┌──────────────────────┐    │    │
│                              │  │ Azure NetApp Files   │    │    │
│                              │  │ (ANF)                │    │    │
│                              │  └──────────────────────┘    │    │
│                              │  ┌──────────────────────┐    │    │
│                              │  │ Blob Storage (cold)  │    │    │
│                              │  └──────────────────────┘    │    │
│                              └──────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

## 4. Data Tiering Engine

```
FabricPool Tiering Architecture:
  
  On-Prem Volume:
  ┌────────────────────────────────────────┐
  │ Performance Tier (local SSD/HDD)      │
  │ ┌──────┐ ┌──────┐ ┌──────┐           │
  │ │Blk A │ │Blk B │ │Blk D │  (hot)    │
  │ │(hot) │ │(warm)│ │(hot) │           │
  │ └──────┘ └──────┘ └──────┘           │
  │                                        │
  │ Cooling Policy:                        │
  │   auto:     Tier blocks cold >31 days  │
  │   snapshot: Tier only snapshot blocks   │
  │   all:      Tier everything possible    │
  │   none:     No tiering (keep local)     │
  └──────────┬─────────────────────────────┘
             │ Tier-out (background)
             ▼
  ┌────────────────────────────────────────┐
  │ Cloud Tier (Object Store)             │
  │ ┌──────┐ ┌──────┐ ┌──────┐           │
  │ │Blk C │ │Blk E │ │Blk F │  (cold)   │
  │ │ obj1 │ │ obj2 │ │ obj3 │           │
  │ └──────┘ └──────┘ └──────┘           │
  │                                        │
  │ Object Mapping:                        │
  │   4MB block → 1 cloud object           │
  │   Metadata stays on performance tier   │
  │   Inline compression before upload     │
  │                                        │
  │ Recall on Read:                        │
  │   App reads cold block → fetch from    │
  │   cloud → cache locally → serve        │
  │   Latency: 2-5s first byte            │
  └────────────────────────────────────────┘

Tiering Decision Algorithm:
  for each 4MB block in volume:
    if access_time < now() - cooling_period:
      if policy == 'auto' or policy == 'all':
        queue_for_tierout(block)
    if block.recently_accessed and block.location == CLOUD:
      queue_for_recall(block)  # bring back to performance tier
```

## 5. Data Mobility & Cloud Bursting

```
Cloud Bursting Workflow:
  
  Normal: All I/O served from on-prem
  
  Peak Detected:
  1. Cloud Manager detects capacity/performance threshold
  2. Spin up CVO instance in cloud (pre-configured AMI)
  3. SnapMirror replicate volume to CVO (incremental)
  4. Update DNS/mount points to cloud endpoint
  5. Serve I/O from cloud CVO
  6. Monitor: when peak subsides
  7. Reverse replicate changes back to on-prem
  8. Fail back to on-prem; terminate CVO instance
  
  Time to burst: ~15 minutes (pre-staged CVO)
  Cost: pay only for cloud compute during burst

Data Migration Flow:
  On-Prem → Cloud (permanent migration):
  1. Establish SnapMirror to CVO
  2. Incremental sync until cutover window
  3. Final sync (quiesce source, transfer last changes)
  4. Break mirror; cloud volume becomes primary
  5. Update application endpoints
  6. Decommission on-prem volume
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Register Cloud Target (Object Store)
POST /api/v1/cloud/targets
{
  "provider": "aws_s3",           // aws_s3 | azure_blob | gcp_gcs
  "bucket": "company-cold-tier",
  "region": "us-east-1",
  "access_key": "vault:///cloud/aws/access_key",
  "encryption": "SSE-S3",
  "proxy": null
}

# Attach Cloud Tier to Aggregate
POST /api/v1/aggregates/{aggr_name}/cloud-stores
{
  "target_id": "cloud-target-uuid",
  "tiering_fullness_threshold": 50   // % full before tiering starts
}

# Set Volume Tiering Policy
PATCH /api/v1/volumes/{vol_uuid}
{
  "tiering": {
    "policy": "auto",             // auto | snapshot-only | all | none
    "cooling_days": 31,
    "cloud_retrieval_policy": "default"  // default | on-read | never | promote
  }
}

# Deploy Cloud Volumes ONTAP
POST /api/v1/cloud-manager/deployments
{
  "provider": "aws",
  "region": "us-east-1",
  "instance_type": "m5.2xlarge",
  "ha_pair": true,
  "capacity_tier": "s3",
  "license": "byol",
  "vpc_id": "vpc-abc123",
  "security_group": "sg-ontap"
}

# Create Cross-Environment Replication
POST /api/v1/snapmirror/relationships
{
  "source": {"cluster": "onprem-cluster", "volume": "prod_data"},
  "destination": {"cluster": "cvo-aws", "volume": "prod_data_cloud"},
  "policy": "mirror_and_vault",
  "schedule": "5min"
}
```

### Tiering Metadata Structure

```
Block Location Map (per volume):
┌──────────────────────────────────────────────────────────┐
│ Volume: prod_data                                        │
│                                                          │
│ Block Index  │ Location │ Cloud Object Key │ Last Access  │
│──────────────┼──────────┼─────────────────┼─────────────│
│ 0-1023       │ LOCAL    │ -               │ 2024-01-15  │
│ 1024-2047    │ CLOUD    │ tier/v1/blk1024 │ 2023-06-01  │
│ 2048-3071    │ LOCAL    │ -               │ 2024-01-14  │
│ 3072-4095    │ CLOUD    │ tier/v1/blk3072 │ 2023-03-15  │
│ 4096-5119    │ TIERING  │ (in transit)    │ 2023-09-01  │
│ ...          │          │                 │             │
│                                                          │
│ Total blocks: 500M (2PB volume, 4MB blocks)              │
│ Local: 30% (hot/warm)                                    │
│ Cloud: 68% (cold)                                        │
│ In-transit: 2%                                           │
│                                                          │
│ Metadata size: ~2GB (stays on performance tier always)   │
└──────────────────────────────────────────────────────────┘
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Cloud capacity** | Virtually unlimited object storage per volume | 300TB per volume, petabytes per cluster |
| **Tiering throughput** | Parallel 4MB block uploads, multi-connection | 10 Gbps sustained |
| **Cloud instances** | Deploy CVO instances across regions on-demand | 30 CVO pairs per Cloud Manager |
| **Management scope** | Single Cloud Manager manages all environments | 100+ clusters (on-prem + cloud) |
| **Multi-cloud** | Same APIs, same data services across AWS/Azure/GCP | 3 clouds simultaneously |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Cloud object deletion** | Object lock / versioning on cloud bucket; immutable retention |
| **Tiering metadata corruption** | WAFL checksums on metadata; metadata replicated with SnapMirror |
| **Cloud provider outage** | Cold blocks temporarily unavailable; hot data served from local tier |
| **In-transit block loss** | Upload verified with checksum; retry on failure; block stays on local tier until confirmed |
| **CVO instance failure** | HA pair in cloud; NVRAM mirrored via cloud network; auto-failover |

## 9. Latency

| Operation | Latency |
|-----------|---------|
| **Read (hot, local SSD)** | 200μs |
| **Read (warm, local HDD)** | 2ms |
| **Read (cold, cloud recall)** | 2-5s first byte; subsequent 4MB blocks parallel |
| **Tier-out (background)** | Not user-facing; ~50ms per 4MB block upload |
| **CVO I/O (cloud-native)** | 1-2ms (EBS gp3 backend) |

**Optimization:** Prefetch adjacent cold blocks on sequential read; cache recalled blocks on local SSD; read-ahead policies for known sequential workloads.

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Cloud connectivity loss** | Cannot tier out or recall cold blocks | Serve hot data locally; queue tier operations; alert |
| **Cloud object corruption** | Data block unreadable | Checksums detect; restore from SnapMirror secondary or backup |
| **CVO crash in cloud** | Cloud workloads interrupted | HA pair with auto-failover (<60s); data on persistent EBS/ANF |
| **Cloud provider deprecation** | Vendor lock-in risk | Multi-cloud architecture; same ONTAP data services everywhere |

## 11. Availability

**Target: 99.99% for management plane; 99.999% for on-prem data**

```
Availability Architecture:
  On-Prem: HA pairs → 99.999% (< 5min downtime/year)
  Cloud CVO: HA in availability zones → 99.99%
  Cloud Manager: SaaS (NetApp managed) → 99.9%
  
  Degraded Mode:
    Cloud Manager down → on-prem continues independently
    Cloud connectivity lost → hot data served; cold reads fail gracefully
    CVO down → failover to partner; data on persistent storage
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Cloud credentials** | Stored in HashiCorp Vault / AWS KMS; never in config files |
| **Data in transit** | TLS 1.3 to cloud APIs; IPsec for intercluster replication |
| **Data at rest (cloud)** | SSE-S3/SSE-KMS (AWS), Azure SSE, or customer-managed keys |
| **Data at rest (on-prem)** | NVE (volume encryption) + NAE (aggregate encryption) |
| **Access control** | RBAC for Cloud Manager; IAM roles for cloud resources |
| **Data sovereignty** | Tiering policies enforce geo-restrictions; tag data with region constraints |

## 13. Cost Constraints

**Estimated Cost (2PB on-prem + 5PB cloud-tiered):**

| Component | Cost |
|-----------|------|
| **On-prem infrastructure** | 2PB all-flash: $2.4M CapEx (amortized $66K/month over 3yr) |
| **Cloud object storage** | 5PB S3 IA: $62,500/month |
| **Cloud egress** | 500GB/day recall: ~$1,200/month |
| **CVO instances** | 2 HA pairs (burst): $8,000/month (on-demand hours) |
| **Cloud Manager** | Included with NetApp subscription |
| **Total monthly** | **~$137,700/month** |

Cost savings vs all-on-prem: ~60% (5PB on cloud at $0.0125/GB vs $0.10/GB on-prem SSD).

## Key Interview Discussion Points

1. **Why hybrid vs all-cloud?** — Performance-sensitive workloads need on-prem latency; compliance may require data on-prem; cloud for elastic capacity and cold storage economics
2. **How does tiering maintain transparency?** — File system metadata stays local; cloud blocks recalled on-demand; application sees single namespace with no code changes
3. **What if cloud is unreachable?** — Hot data serves normally; cold reads fail with EIO; tier-out queued. Design for graceful degradation, not hard dependency
4. **Multi-cloud portability?** — Same ONTAP data services (snapshots, replication, encryption) run identically on any cloud. Data format portable
5. **Tiering performance impact?** — Cold data recall = seconds (not microseconds). Mitigate with cooling period tuning, prefetch, and monitoring access patterns
