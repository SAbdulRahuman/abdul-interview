# Design a Storage Operating System

## Overview
Design a storage operating system like NetApp ONTAP (Data ONTAP) — a purpose-built OS for enterprise storage that provides a unified data management platform with WAFL (Write Anywhere File Layout), HA pairs, non-disruptive operations, multi-protocol support, and storage virtualization (SVMs). ONTAP powers NetApp FAS/AFF systems and Cloud Volumes ONTAP across AWS/Azure/GCP.

## 1. Requirements

**Functional:**
- Purpose-built OS for storage: optimized I/O path, not general-purpose Linux
- WAFL filesystem: copy-on-write, snapshots, write-anywhere layout
- HA pair architecture: active-active with storage failover (SFO)
- Storage Virtual Machines (SVMs): multi-tenant isolation
- Non-disruptive operations: upgrade, expand, migrate without downtime
- Clustered scale-out: up to 24 nodes in a single cluster
- Data management: snapshots, clones, replication, tiering, dedup, compression

**Non-Functional:**
- I/O latency: <200μs (all-flash)
- IOPs: 1M+ per HA pair (8K random read)
- Availability: 99.999% with HA
- Non-disruptive upgrade: zero downtime for major version upgrades
- Data integrity: end-to-end checksums from application to disk

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Cluster size** | Up to 12 HA pairs (24 nodes) |
| **Per-node capacity** | 100TB-2PB (flash) |
| **Cluster capacity** | Up to 48PB |
| **SVMs per cluster** | Up to 400 |
| **Volumes per cluster** | Up to 15,000 |
| **IOPs per cluster** | 10M+ (aggregate across all nodes) |

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│             Storage Operating System (ONTAP-like)                 │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐      │
│  │                 Management Plane                       │      │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │      │
│  │  │ CLI      │  │ REST API │  │ System Manager   │    │      │
│  │  │ (SSH)    │  │ (HTTPS)  │  │ (Web GUI)        │    │      │
│  │  └──────────┘  └──────────┘  └──────────────────┘    │      │
│  └────────────────────────────────────────────────────────┘      │
│                           │                                      │
│  ┌────────────────────────────────────────────────────────┐      │
│  │              Cluster Layer (D-blade + N-blade)         │      │
│  │                                                        │      │
│  │  Node 1 (HA Pair A)        Node 2 (HA Pair A)        │      │
│  │  ┌───────────────────┐    ┌───────────────────┐      │      │
│  │  │ N-blade            │    │ N-blade            │      │      │
│  │  │ (Network Protocol) │    │ (Network Protocol) │      │      │
│  │  │ NFS/SMB/iSCSI/S3   │    │ NFS/SMB/iSCSI/S3   │      │      │
│  │  │ LIF Management     │    │ LIF Management     │      │      │
│  │  ├───────────────────┤    ├───────────────────┤      │      │
│  │  │ D-blade            │    │ D-blade            │      │      │
│  │  │ (Data Path)        │    │ (Data Path)        │      │      │
│  │  │ WAFL               │    │ WAFL               │      │      │
│  │  │ RAID-DP/TEC        │    │ RAID-DP/TEC        │      │      │
│  │  │ Volume Management  │    │ Volume Management  │      │      │
│  │  ├───────────────────┤    ├───────────────────┤      │      │
│  │  │ NVRAM (battery-    │◄──►│ NVRAM (battery-    │      │      │
│  │  │  backed, mirrored) │HA  │  backed, mirrored) │      │      │
│  │  ├───────────────────┤Link├───────────────────┤      │      │
│  │  │ Flash Pool / HDD   │    │ Flash Pool / HDD   │      │      │
│  │  │ Aggregates         │    │ Aggregates         │      │      │
│  │  └───────────────────┘    └───────────────────┘      │      │
│  │                                                        │      │
│  │  Cluster Interconnect (10/25/100 GbE)                 │      │
│  │  ════════════════════════════════════                  │      │
│  │  Node 3 (HA Pair B)        Node 4 (HA Pair B)        │      │
│  │  ┌───────────────────┐    ┌───────────────────┐      │      │
│  │  │        ...         │    │        ...         │      │      │
│  │  └───────────────────┘    └───────────────────┘      │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐      │
│  │     Storage Virtual Machine (SVM) Layer                │      │
│  │                                                        │      │
│  │  SVM-prod       SVM-dev        SVM-dr                 │      │
│  │  ┌────────┐    ┌────────┐    ┌────────┐              │      │
│  │  │ vols   │    │ vols   │    │ vols   │              │      │
│  │  │ LIFs   │    │ LIFs   │    │ LIFs   │              │      │
│  │  │ exports│    │ exports│    │ exports│              │      │
│  │  │ users  │    │ users  │    │ users  │              │      │
│  │  │ DNS    │    │ DNS    │    │ DNS    │              │      │
│  │  └────────┘    └────────┘    └────────┘              │      │
│  │                                                        │      │
│  │  Each SVM: isolated namespace, protocols, identity     │      │
│  │  Can span multiple nodes in the cluster                │      │
│  └────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

## 4. WAFL (Write Anywhere File Layout)

```
WAFL Architecture:
══════════════════

  Core Principle: Never overwrite data in place
  Every write goes to a NEW location on disk

  Traditional Filesystem (ext4, NTFS):
    Write → overwrites existing block location
    Problem: crash during overwrite = corrupted data
    Solution: journal (write twice: journal + data)

  WAFL:
    Write → allocate new block → write data → update metadata pointer
    Old block still valid until pointer updated atomically
    Result: crash-safe without journal overhead

  WAFL Block Layout:
  ┌────────────────────────────────────────────────────┐
  │                                                     │
  │  Root Inode (superblock)                            │
  │    └─► Inode File                                   │
  │          └─► Inode 42 (file "report.pdf")          │
  │                └─► Block Map:                       │
  │                     [0] → Block 1001 (data)        │
  │                     [1] → Block 1002 (data)        │
  │                     [2] → Block 1003 (data)        │
  │                                                     │
  │  Write to block [1] of file "report.pdf":          │
  │    1. Allocate NEW block 5007                       │
  │    2. Write new data to block 5007                  │
  │    3. Update inode: [1] → Block 5007               │
  │    4. Old block 1002 still valid (snapshot points   │
  │       to it)                                        │
  │    5. If no snapshot references → block 1002 freed  │
  │                                                     │
  │  Consistency Point (CP):                            │
  │    Every 10 seconds or when NVRAM buffer ≥ threshold│
  │    1. Write all dirty data to new locations         │
  │    2. Write updated metadata (inodes, block maps)   │
  │    3. Atomically update root pointer                │
  │    4. Old root = instant snapshot                    │
  │                                                     │
  │  Benefits:                                          │
  │    ✓ Crash recovery: instant (no fsck, no journal)  │
  │    ✓ Snapshots: zero cost (just keep old root)      │
  │    ✓ Clones: instant (share all blocks)             │
  │    ✓ No write amplification from journaling         │
  │    ✓ Optimized for sequential writes (write-anywhere)│
  └────────────────────────────────────────────────────┘

  NVRAM (Non-Volatile RAM):
  ┌────────────────────────────────────────────────────┐
  │                                                     │
  │  Battery-backed (or flash-backed) DRAM              │
  │  Size: 16-64 GB per controller                      │
  │  Purpose: write cache (ACK to client before disk)   │
  │                                                     │
  │  Write Path:                                        │
  │    Client WRITE → NVRAM (log entry) → ACK to client │
  │    Background: NVRAM → WAFL CP → disk               │
  │                                                     │
  │  Crash Recovery:                                    │
  │    Controller crash → reboot → replay NVRAM log     │
  │    → reconstruct pending writes → consistent state  │
  │    Recovery time: <60 seconds                       │
  │                                                     │
  │  HA Mirroring:                                      │
  │    Every NVRAM entry mirrored to partner's NVRAM    │
  │    via HA interconnect (<100μs)                     │
  │    If primary fails → partner replays partner's     │
  │    NVRAM → zero data loss                           │
  └────────────────────────────────────────────────────┘
```

## 5. HA Pair & Storage Failover (SFO)

```
HA Pair Architecture:
═════════════════════

  ┌─────────────────┐     HA Interconnect     ┌─────────────────┐
  │ Controller A     │◄═══════════════════════►│ Controller B     │
  │                  │  (NVRAM mirror +        │                  │
  │  NVRAM A         │   heartbeat)            │  NVRAM B         │
  │  ┌─────────┐    │                          │  ┌─────────┐    │
  │  │ A's ops │    │                          │  │ B's ops │    │
  │  │ B mirror│    │                          │  │ A mirror│    │
  │  └─────────┘    │                          │  └─────────┘    │
  │                  │                          │                  │
  │  Owns:           │                          │  Owns:           │
  │  Aggr A1, A2     │                          │  Aggr B1, B2     │
  │  SVM-prod (LIFs) │                          │  SVM-dev (LIFs)  │
  └────────┬─────────┘                          └────────┬─────────┘
           │                                              │
     ┌─────┴──────────────────────────────────────────────┴─────┐
     │                  Shared Disk Shelf                        │
     │  (Both controllers can access all disks via dual-path)   │
     │  ┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐     │
     │  │ A1 ││ A1 ││ A2 ││ A2 ││ B1 ││ B1 ││ B2 ││ B2 │     │
     │  └────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘     │
     └──────────────────────────────────────────────────────────┘

  Storage Failover (SFO):
    Normal: Controller A serves A's aggregates + SVMs
            Controller B serves B's aggregates + SVMs
    
    Controller A Fails:
    T=0s:   A stops heartbeat
    T=3s:   B detects failure (missed heartbeats)
    T=5s:   B takes over A's aggregates
    T=8s:   B replays A's NVRAM (committed but unflushed writes)
    T=10s:  B brings up A's SVMs (LIFs, protocols)
    T=15s:  B serves both A and B workloads
    
    Giveback (A recovered):
    Admin: storage failover giveback -ofnode A
    1. Quiesce A's workloads on B
    2. Transfer A's aggregates back to A
    3. A resumes serving its own SVMs
    4. Total disruption: <30 seconds

  Non-Disruptive Upgrade (NDU):
    1. Upgrade Controller B first
       - Migrate B's LIFs to A
       - SFO: A takes over B's aggregates
       - Upgrade B's ONTAP
       - Reboot B (new version)
       - SFO giveback: B reclaims its aggregates
    2. Upgrade Controller A
       - Same process in reverse
       - Both controllers now on new version
    3. Total client disruption: ~0 (LIF migration is transparent)
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Cluster Management
GET /api/v1/cluster
→ {
  "name": "prod-cluster-01",
  "version": "9.14.1",
  "nodes": [
    {"name": "node-01", "model": "AFF-A900", "uptime": "365d", "state": "up"},
    {"name": "node-02", "model": "AFF-A900", "uptime": "365d", "state": "up"}
  ],
  "ha_pairs": [
    {
      "nodes": ["node-01", "node-02"],
      "interconnect_state": "connected",
      "takeover_possible": true
    }
  ]
}

# Aggregate Management
POST /api/v1/storage/aggregates
{
  "name": "aggr1_node01",
  "node": "node-01",
  "raid_type": "raid_tec",
  "disk_count": 29,
  "raid_size": 29,
  "encryption": true,
  "block_type": "64_bit"
}

# Volume Management
POST /api/v1/storage/volumes
{
  "name": "vol_prod_db",
  "svm": "svm-prod",
  "aggregate": "aggr1_node01",
  "size": "10TB",
  "type": "rw",
  "guarantee": "none",            // thin provisioning
  "snapshot_policy": "default",
  "efficiency": {
    "dedupe": true,
    "compression": "auto",        // auto-selects LZ4 or ZSTD
    "compaction": true
  },
  "tiering": {
    "policy": "auto",             // hot data on SSD, cold on HDD/cloud
    "minimum_cooling_days": 31
  },
  "qos": {
    "policy": "prod-gold",
    "max_throughput_iops": 50000,
    "max_throughput_mbps": 2000
  }
}

# SVM Management
POST /api/v1/svm/svms
{
  "name": "svm-prod",
  "ipspace": "Default",
  "allowed_protocols": ["nfs", "cifs", "iscsi", "s3"],
  "dns": {
    "domains": ["corp.example.com"],
    "servers": ["10.0.0.53", "10.0.0.54"]
  },
  "nis": {"domain": "corp", "servers": ["10.0.0.55"]},
  "language": "en_US.UTF-8",
  "max_volumes": 1000
}

# Storage Failover
GET /api/v1/cluster/nodes/{node}/ha
→ {
  "partner": "node-02",
  "state": "connected",
  "takeover_possible": true,
  "giveback_state": "nothing_to_give_back",
  "auto_giveback": true,
  "auto_giveback_delay_secs": 600
}

POST /api/v1/cluster/nodes/{node}/ha/takeover
{
  "type": "normal",              // normal | forced | bypass_optimization
  "reason": "planned_maintenance"
}

POST /api/v1/cluster/nodes/{node}/ha/giveback
{
  "override_vetoes": false
}

# Non-Disruptive Upgrade
POST /api/v1/cluster/software
{
  "version": "9.15.0",
  "action": "upgrade",
  "validation_only": false,
  "stabilize_minutes": 8,
  "update_type": "automated_ndu"  // automated non-disruptive upgrade
}
→ {
  "job_id": "upgrade-001",
  "plan": [
    {"step": 1, "action": "upgrade_node_02", "estimated": "45min"},
    {"step": 2, "action": "upgrade_node_01", "estimated": "45min"}
  ],
  "total_estimated": "90min",
  "client_impact": "none"
}
```

### Internal Architecture

```
ONTAP Kernel Architecture:

  ┌──────────────────────────────────────────────┐
  │  User Space                                   │
  │  ┌────────────────────────────────────┐      │
  │  │  Management Daemons                │      │
  │  │  mgwd (management gateway)         │      │
  │  │  httpd (REST API + System Manager) │      │
  │  │  sshd (CLI access)                 │      │
  │  │  vifmgr (LIF manager)             │      │
  │  │  bcomd (cluster communication)     │      │
  │  └────────────────────────────────────┘      │
  ├──────────────────────────────────────────────┤
  │  Kernel Space (ONTAP Kernel / Data ONTAP)    │
  │  ┌────────────────────────────────────┐      │
  │  │  N-blade (Network Blade)           │      │
  │  │  - Protocol engines                │      │
  │  │  - NFS/CIFS/iSCSI/NVMe/S3         │      │
  │  │  - Session management              │      │
  │  │  - Name services (DNS, LDAP, NIS)  │      │
  │  ├────────────────────────────────────┤      │
  │  │  SpinNP (Cluster Messaging)        │      │
  │  │  - Inter-node data forwarding      │      │
  │  │  - When client connects to node A  │      │
  │  │    but data is on node B           │      │
  │  │  - Redirect or proxy the I/O       │      │
  │  ├────────────────────────────────────┤      │
  │  │  D-blade (Data Blade)              │      │
  │  │  - WAFL filesystem                 │      │
  │  │  - Volume management               │      │
  │  │  - Snapshot engine                  │      │
  │  │  - Dedup / compression engine      │      │
  │  │  - RAID manager (DP / TEC)         │      │
  │  │  - NVRAM management                │      │
  │  ├────────────────────────────────────┤      │
  │  │  Storage Layer                     │      │
  │  │  - Disk driver (SAS, NVMe)        │      │
  │  │  - Flash Cache (read caching)      │      │
  │  │  - FabricPool (cloud tiering)      │      │
  │  └────────────────────────────────────┘      │
  └──────────────────────────────────────────────┘

I/O Path (Read):
  Client READ → NIC → N-blade (parse NFS/iSCSI)
    → Is data on this node? 
      Yes → D-blade → WAFL → Flash Cache hit? 
        Yes → return from cache
        No → RAID → Disk → return data
      No → SpinNP → forward to owning node's D-blade
        → return result via SpinNP

I/O Path (Write):
  Client WRITE → NIC → N-blade
    → D-blade → WAFL → NVRAM (log entry)
    → Mirror NVRAM entry to HA partner
    → ACK to client (data is safe in 2x NVRAM)
    → Background: Consistency Point → write to disk
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Nodes** | Clustered scale-out; SpinNP interconnect | 24 nodes (12 HA pairs) |
| **Capacity** | Aggregates per node; FlexVol/FlexGroup | 48PB per cluster |
| **IOPs** | Linear with nodes; local I/O preferred | 10M+ per cluster |
| **Volumes** | FlexVol (per node) + FlexGroup (distributed) | 15,000 per cluster |
| **SVMs** | 400 per cluster; each spans nodes as needed | 400 tenants |
| **Protocols** | All protocols available on every node | NFS+SMB+iSCSI+S3 |

## 8. No Data Loss

| Scenario | Protection |
|----------|------------|
| **Controller crash** | NVRAM replay: all acknowledged writes recovered |
| **Both controllers crash** | Battery-backed NVRAM survives power loss for 72+ hours |
| **Disk failure** | RAID-DP/TEC: survive 2-3 simultaneous disk failures |
| **Corruption** | End-to-end block checksums; WAFL never overwrites (COW) |
| **Site failure** | MetroCluster: sync mirror to remote site (RPO=0) |
| **Human error** | Snapshots: instant point-in-time recovery |

## 9. Latency

| Operation | AFF (All-Flash) | FAS (Hybrid) |
|-----------|-----------------|--------------|
| **4K random read** | 100-200μs | 1-5ms (cache miss to HDD) |
| **4K random write** | 200-300μs | 200-300μs (NVRAM-buffered) |
| **Sequential read** | ~500μs first byte | ~5ms first byte |
| **Sequential write** | ~300μs | ~300μs (NVRAM) |
| **HA takeover** | 15-30s (SFO) | 15-30s |
| **LIF migration** | <10s | <10s |
| **Snapshot create** | <1s | <1s |
| **NDU total** | 90 min (2 nodes) | 90 min |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Controller failure** | HA partner takes over in 15-30s | Automatic SFO; NVRAM mirror |
| **Disk failure** | No impact; RAID rebuilds | RAID-TEC (triple parity); hot spare |
| **NVRAM battery** | Write caching disabled | Alert; battery replaced online |
| **Interconnect failure** | Cross-node I/O unavailable | Dual interconnect; redirect to local |
| **Firmware bug** | Potential data path issue | Automated NDU with validation checks |

## 11. Availability

**Target: 99.999% (< 5.26 minutes downtime per year)**

- HA pairs: automatic failover (<30s)
- MetroCluster: site-level HA (<120s)
- Non-disruptive operations: upgrade, expand, migrate = 0 downtime
- Aggregate relocation (ARL): move aggregates between nodes non-disruptively
- LIF migration: transparent to clients

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | RBAC with predefined + custom roles; SAML; MFA |
| **Data encryption** | NVE (per-volume), NAE (per-aggregate), NSE (per-disk SED) |
| **Key management** | Onboard Key Manager or KMIP (Thales, Gemalto, etc.) |
| **Protocol security** | Kerberos (NFS), AES-GCM (SMB), CHAP (iSCSI), TLS 1.3 |
| **Audit** | FPolicy (file screening), native CIFS/NFS audit logging |
| **Ransomware** | Autonomous Ransomware Protection (ARP): ML-based anomaly detection |
| **Multi-admin** | Multi-admin verification for destructive operations |
| **Zero trust** | Certificate-based cluster peering; verified boot chain |

## 13. Cost Constraints

**Estimated Cost (HA Pair, AFF A900, 200TB usable):**

| Component | Cost |
|-----------|------|
| **2× AFF A900 controllers** | $250,000 |
| **200TB NVMe flash (raw ~300TB)** | $500,000 |
| **ONTAP licenses (base + data services)** | $150,000 |
| **HA interconnect + cabling** | $10,000 |
| **3-year support/maintenance** | $120,000 |
| **Total** | **~$1,030,000** |
| **Cost per usable TB** | **~$5,150** |

Cloud alternative: CVO (Cloud Volumes ONTAP) on AWS — ~$2/GB/month capacity tier, ~$0.05/GB/month PAYGO. For 200TB: $24,000/month ($288,000/year). Break-even vs on-prem: ~3.5 years.

## Key Interview Discussion Points

1. **Why a purpose-built OS vs Linux + filesystem?** — General-purpose OS has scheduler overhead, context switching, memory management not optimized for I/O. ONTAP kernel: direct-to-disk I/O path, zero-copy where possible, NVRAM as write cache, filesystem and RAID tightly integrated. Result: 100μs latency vs 500μs+ on Linux-based storage
2. **WAFL vs traditional journaling FS?** — WAFL: write-anywhere eliminates journal overhead; crash recovery is instant (last CP is consistent); snapshots are free (keep old CP). ext4/XFS: journal doubles write overhead; fsck on crash; snapshots require LVM (slow, limited). WAFL trades sequential disk layout for these benefits
3. **N-blade vs D-blade separation?** — N-blade handles protocol processing (NFS/SMB parsing); D-blade handles data layout (WAFL/RAID). Separation allows: client connects to any node (N-blade) but data served from owning node (D-blade) via cluster interconnect. This enables non-disruptive LIF migration
4. **How does NDU work without downtime?** — Rolling upgrade: upgrade one node at a time. Before upgrade: migrate LIFs off target node → HA takeover → upgrade + reboot → HA giveback → migrate LIFs back. Client sees brief pause (ONTAP handles NFS/SMB/iSCSI reconnection transparently)
5. **SVM as multi-tenancy primitive?** — SVM = virtual storage server: own namespace, protocols, identity (DNS, LDAP), network (LIFs, routing). Isolation: SVM admin cannot see other SVMs. Delegated administration: tenant manages their SVM. Can span multiple nodes. Migration: entire SVM can be migrated between clusters
