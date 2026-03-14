# Design a Network Attached Storage (NAS) System

## Overview
Design an enterprise NAS system like NetApp ONTAP NAS or Dell EMC Isilon — providing file-level storage over NFS and SMB/CIFS protocols with POSIX semantics, snapshots, deduplication, and high availability.

## 1. Requirements

**Functional:**
- File system with POSIX compliance (permissions, locking, timestamps)
- Multi-protocol: NFS v3/v4.1 (pNFS) and SMB 3.x simultaneously
- Snapshots and clones (space-efficient, near-instant)
- Inline deduplication and compression
- QoS per volume (IOPS, throughput, latency caps)
- HA controller pairs (non-disruptive failover)

**Non-Functional:**
- Latency: <1ms for metadata ops, <2ms for 4KB reads
- Throughput: 10+ GB/s per HA pair
- Capacity: petabyte-scale per cluster
- Availability: 99.999% (5.26 min downtime/year)

## 2. Scale Estimation

```
Storage capacity:         2 PB usable (4 PB raw with efficiency)
Files:                    10 billion
Volumes:                  5,000
IOPS:                     500K mixed (70% read, 30% write)
Throughput:               20 GB/s aggregate
Clients:                  10,000 concurrent NFS/SMB sessions
Snapshots per volume:     255
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      NAS Cluster                            │
│                                                             │
│  ┌─────────────────┐      ┌─────────────────┐             │
│  │  Controller A    │◄────►│  Controller B    │ (HA Pair 1) │
│  │  (Active)        │NVRAM │  (Standby)       │             │
│  │  CPU + 256GB RAM │mirror│  CPU + 256GB RAM  │             │
│  └────────┬────────┘      └────────┬─────────┘             │
│           │                        │                        │
│  ┌────────┴────────────────────────┴─────────┐             │
│  │           Disk Shelf Fabric                │             │
│  │  ┌────────┐ ┌────────┐ ┌────────┐         │             │
│  │  │SSD Tier│ │SSD Tier│ │HDD Tier│         │             │
│  │  │(hot)   │ │(hot)   │ │(cold)  │         │             │
│  │  │24×NVMe │ │24×NVMe │ │40×SAS  │         │             │
│  │  └────────┘ └────────┘ └────────┘         │             │
│  └───────────────────────────────────────────┘             │
│                                                             │
│  Client Access:                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐                │
│  │NFS v4.1 │  │SMB 3.1  │  │Management   │                │
│  │(pNFS)   │  │(CIFS)   │  │REST API     │                │
│  │Port 2049│  │Port 445 │  │Port 443     │                │
│  └─────────┘  └─────────┘  └─────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

## 4. File System Layout (WAFL-inspired)

```
Write Anywhere File Layout:
┌──────────────────────────────────────────────────────────┐
│ Aggregate (physical storage pool)                        │
│                                                          │
│  RAID Group 1      RAID Group 2      RAID Group 3       │
│  ┌──┬──┬──┬──┐   ┌──┬──┬──┬──┐   ┌──┬──┬──┬──┐       │
│  │D0│D1│D2│P │   │D3│D4│D5│P │   │D6│D7│D8│P │       │
│  └──┴──┴──┴──┘   └──┴──┴──┴──┘   └──┴──┴──┴──┘       │
│                                                          │
│  Volumes (thin-provisioned):                             │
│  ┌──────────────────────┐                                │
│  │ Volume: /vol/data1    │                                │
│  │ Logical size: 10TB    │                                │
│  │ Used: 3.2TB           │                                │
│  │ Snapshots: 12         │                                │
│  │ Dedup savings: 35%    │                                │
│  │ Export: NFS + SMB      │                                │
│  │ QoS: max 50K IOPS     │                                │
│  └──────────────────────┘                                │
│                                                          │
│  Write Path (copy-on-write):                              │
│  1. Write arrives → NVRAM journal (battery-backed)       │
│  2. Acknowledge to client (sub-ms)                       │
│  3. Coalesce writes in memory (CP interval: 10s)         │
│  4. Write full 4KB blocks to FREE space (never in-place)│
│  5. Update block pointers in inode tree                   │
│  6. Old blocks become snapshot data (no copy needed)     │
└──────────────────────────────────────────────────────────┘
```

## 5. Snapshot Implementation

```
Copy-on-Write Snapshots (WAFL-style):

Before Snapshot:
  Inode Tree → [Block A] [Block B] [Block C]

Create Snapshot (instant — just mark current tree):
  Snapshot_1 tree → [Block A] [Block B] [Block C]
  Active tree     → [Block A] [Block B] [Block C]  (shared)

After Modifying Block B:
  Snapshot_1 tree → [Block A] [Block B] [Block C]
  Active tree     → [Block A] [Block B'] [Block C]
  
  Block B' written to NEW location (write-anywhere)
  Block B preserved for snapshot (no extra copy!)
  Space overhead: only changed blocks

Snapshot Chain (up to 255 per volume):
  snap_daily_1 → snap_daily_2 → ... → snap_daily_7
  snap_hourly_1 → ... → snap_hourly_24
  
  .snapshot directory: visible to clients (NFS/SMB)
  Self-service restore: user browses .snapshot/daily_1/
```

## 6. Multi-Protocol Access

```
Mixed Protocol Support (NFS + SMB on same data):

┌──────────────────────────────────────────────────┐
│              Unified Permission Model            │
│                                                  │
│  NFS Client:                                     │
│    POSIX permissions: uid=1000, gid=100, mode=755│
│    NFSv4 ACLs (optional)                         │
│                                                  │
│  SMB Client:                                     │
│    NTFS ACLs: DOMAIN\user, Full Control          │
│    SID-based security descriptors                │
│                                                  │
│  Mapping Layer:                                  │
│    UNIX uid 1000 ↔ DOMAIN\john (name mapping)    │
│    POSIX mode 755 ↔ NTFS ACL translation         │
│    Security style per volume:                    │
│      "unix"  → POSIX permissions authoritative   │
│      "ntfs"  → NTFS ACLs authoritative           │
│      "mixed" → last-writer-wins (not recommended)│
│                                                  │
│  File Locking:                                   │
│    NFS: byte-range locks (NLM or NFSv4 state)    │
│    SMB: oplock/lease-based locking               │
│    Cross-protocol: lock manager coordinates both │
│    Prevents: NFS client reading while SMB writes │
└──────────────────────────────────────────────────┘
```

## 7. Low-Level Design (LLD)

### API Contracts (Management REST API)

```
# Create Volume
POST /api/v1/volumes
{
  "name": "data_vol_01",
  "aggregate": "aggr1_ssd",
  "size_gb": 10240,
  "thin_provisioned": true,
  "security_style": "unix",        // unix | ntfs | mixed
  "snapshot_policy": "default",     // hourly:6, daily:2, weekly:2
  "efficiency": {
    "dedup": "inline",
    "compression": "lz4"
  },
  "qos": {
    "max_iops": 50000,
    "max_throughput_mbps": 1000,
    "min_iops": 5000               // guaranteed floor
  },
  "export_policy": "allow_10.0.0.0/8"
}

# Create Snapshot
POST /api/v1/volumes/{vol_name}/snapshots
{
  "name": "before_upgrade",
  "comment": "Pre-upgrade backup"
}
Response: {"snapshot_id": "snap_123", "created_at": "...", "size_bytes": 0}

# Restore from Snapshot
POST /api/v1/volumes/{vol_name}/restore
{
  "snapshot": "before_upgrade",
  "preserve_snapshots": true
}

# Clone Volume (instant, space-efficient)
POST /api/v1/volumes/{vol_name}/clone
{
  "clone_name": "data_vol_01_dev",
  "snapshot": "before_upgrade",   // base snapshot for clone
  "thin": true
}
```

### Inode and Block Management

```
Inode Structure (on-disk):
┌──────────────────────────────────────────┐
│ Inode 12345                              │
│  type: regular_file                      │
│  size: 1,048,576 (1MB)                   │
│  uid: 1000, gid: 100                    │
│  permissions: 0644                       │
│  atime, mtime, ctime                    │
│  link_count: 1                          │
│  block_count: 256                       │
│  direct_blocks: [blk_100, blk_101, ...] │
│  indirect_block: blk_200               │
│  double_indirect: blk_300              │
│  dedup_fingerprint: 0xABCD...          │
│  compression_info: lz4, ratio=2.1       │
└──────────────────────────────────────────┘

Block Allocation (write-anywhere):
  Free Space Map (bitmap per aggregate)
  Allocation strategy: 
    1. Prefer contiguous blocks (sequential writes)
    2. Spread across RAID groups (parallel I/O)
    3. Keep hot data on SSD tier
    4. Cold data auto-tiered to HDD

Metadata B-Tree:
  Directory entries: B+ tree keyed by filename hash
  Large directories (>1M files): 
    Multi-level B+ tree with 4KB fan-out
    Lookup: O(log_256 N) ≈ 3 I/Os for 10M files
```

## 8. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Capacity** | Add disk shelves to aggregate; add HA pairs to cluster | 2PB per HA pair, 24 pairs per cluster |
| **IOPS** | SSD tier for metadata + hot data; NVRAM write coalescing | 500K IOPS per HA pair |
| **Throughput** | pNFS parallel access across multiple nodes; 100GbE links | 20 GB/s per HA pair |
| **File Count** | B+ tree directories; parallel directory walk | 10Bn files per cluster |
| **Clients** | NFS session multiplexing; TCP connection pooling | 10K concurrent sessions |

## 9. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Write Path** | NVRAM (battery-backed, mirrored to partner) → data safe even in power loss |
| **RAID** | RAID-DP (double parity) or RAID-TEC (triple parity); survive 2-3 disk failures |
| **Replication** | SnapMirror async/sync to DR site; RPO: 0 (sync) or <15min (async) |
| **Snapshots** | Scheduled snapshots every hour; .snapshot accessible to users for self-restore |
| **SSD Wear** | SSD monitoring; proactive drive replacement before failure; spare drives pre-provisioned |
| **Consistency** | WAFL ensures every write is atomic; crash recovery replays NVRAM journal |

## 10. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Metadata (stat, lookup) | 0.1ms | 0.5ms | <1ms |
| 4KB random read (SSD) | 0.2ms | 1ms | <2ms |
| 4KB random write (SSD) | 0.3ms | 1ms | <2ms |
| 64KB sequential read | 0.5ms | 2ms | <5ms |
| Large sequential write | <1ms per 64KB | 3ms | NVRAM-speed |
| NFS GETATTR | 0.05ms | 0.2ms | <0.5ms |

**Latency Optimization:**
- **NVRAM**: All writes ACKed from NVRAM (battery-backed DRAM), not disk
- **Read-ahead**: Sequential access detected → prefetch next blocks
- **Metadata cache**: 50%+ of RAM for inode/directory caches
- **Connection affinity**: NFS client pinned to controller for cache locality

## 11. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Controller failure** | Active controller down | Partner takes over in <60s (transparent to clients via IP takeover) |
| **Disk failure** | RAID degraded | Automatic reconstruction from parity; hot spare drive used immediately |
| **Dual disk failure** | Data at risk | RAID-DP/TEC survives 2-3 failures; alert ops immediately |
| **NVRAM failure** | Write cache lost | Mirrored NVRAM: partner has copy; fail-safe: disable write caching |
| **Network failure** | Clients disconnected | LIF failover to surviving ports; NFS grace period for lock recovery |

## 12. Availability

**Target: 99.999% (5.26 min downtime/year)**

```
HA Architecture:
  Controller A (active) ←──NVRAM mirror──→ Controller B (standby)
         │                                        │
    ┌────┴────┐                              ┌────┴────┐
    │ Port 1  │ ← LIF failover ─────────→  │ Port 1  │
    │ Port 2  │                              │ Port 2  │
    └─────────┘                              └─────────┘
         │                                        │
    Shared Disk Shelves (both controllers can access)

Non-Disruptive Operations:
  - Controller firmware upgrade: takeover → upgrade → giveback
  - Volume move: online migration between aggregates
  - LIF migration: move network interface between ports
  - Disk replacement: hot-swap with automatic reconstruction
```

## 13. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | Kerberos (NFS/SMB); LDAP/AD integration; local users |
| **Authorization** | POSIX permissions + NFSv4 ACLs; NTFS ACLs for SMB |
| **Encryption at Rest** | NSE (NetApp Storage Encryption) AES-256; self-encrypting drives |
| **Encryption in Transit** | NFS over TLS (RFC 9289); SMB 3.x encryption |
| **Access Control** | Export policies (NFS); share permissions (SMB); IP-based filtering |
| **Audit** | FPolicy: file access auditing; CIFS audit log; syslog integration |
| **Anti-Ransomware** | Detect anomalous file encryption patterns; auto-snapshot + alert |
| **Multi-tenancy** | SVM (Storage Virtual Machine): isolated namespace, network, auth per tenant |

## 14. Cost Constraints

**Estimated Cost (2PB usable, HA pair, enterprise-grade):**

| Component | Specification | Cost |
|-----------|--------------|------|
| **Controllers** | 2× mid-range (HA pair), 256GB RAM each | $150,000 |
| **SSD (hot tier)** | 48× 3.84TB NVMe SSDs | $120,000 |
| **HDD (capacity tier)** | 80× 18TB SAS HDDs | $80,000 |
| **Disk Shelves** | 4× 24-bay SAS shelves | $40,000 |
| **Networking** | 4× 100GbE ports, switches | $30,000 |
| **Software License** | NFS, SMB, SnapMirror, dedup, encryption | $100,000/yr |
| **Total (CapEx + 3yr SW)** | | **~$720,000** |

**Effective $/GB: ~$0.36/GB** (after dedup+compression: ~$0.18/GB effective)

**Cloud Alternative:**
- Amazon FSx for NetApp ONTAP: ~$0.08/GB/month = $163K/month for 2PB
- On-prem breaks even vs cloud at ~4.4 months (CapEx model)

## Key Interview Discussion Points

1. **How does WAFL handle snapshots efficiently?** — Write-anywhere: new data goes to free blocks; old blocks preserved as snapshot data automatically — zero copy overhead
2. **Mixed NFS + SMB on same data?** — Unified permission model with uid↔SID mapping; security style per volume decides authoritative permissions
3. **How to handle millions of files in one directory?** — B+ tree directory structure; hash-based lookup; O(log N) access
4. **What is NVRAM's role?** — Battery-backed write cache; acknowledges writes instantly; survives power failure; mirrored to HA partner
5. **Thin provisioning vs thick?** — Thin: allocate on write (overcommit possible); thick: pre-allocate (guaranteed space). Thin saves 60-70% storage typically
