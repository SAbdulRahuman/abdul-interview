# Design a Distributed Block Storage System

## Overview
Design a distributed block storage system like Ceph RBD or AWS EBS — providing reliable, scalable, thin-provisioned block devices over a commodity server cluster with automatic data placement, self-healing, and consistent performance.

## 1. Requirements

**Functional:**
- Create/attach/detach/resize virtual block devices (volumes)
- Thin provisioning with on-demand allocation
- Snapshots and clones (instant, space-efficient)
- Data placement: replicated (3×) or erasure-coded (4+2)
- Automatic data rebalancing on node add/remove
- IOPS/throughput QoS per volume

**Non-Functional:**
- Latency: <1ms for 4KB random read (SSD tier)
- IOPS: 100K per volume (max), millions aggregate
- Availability: 99.999% per volume
- Durability: 99.999999999% (11 nines)

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Cluster size** | 100 storage nodes |
| **Total raw capacity** | 10PB (100 nodes × 100TB each) |
| **Usable capacity (3× repl)** | 3.3PB |
| **Volumes** | 50,000 active |
| **Avg volume size** | 100GB |
| **Aggregate IOPS** | 10M (4KB random) |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│           Distributed Block Storage Architecture              │
│                                                               │
│  Compute Nodes (clients):                                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │ VM/Container│  │ VM/Container│  │ Bare Metal │             │
│  │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │             │
│  │ │ Block  │ │  │ │ Block  │ │  │ │ Block  │ │             │
│  │ │ Client │ │  │ │ Client │ │  │ │ Client │ │             │
│  │ │(kernel)│ │  │ │(librbd)│ │  │ │(iSCSI) │ │             │
│  │ └───┬────┘ │  │ └───┬────┘ │  │ └───┬────┘ │             │
│  └─────┼──────┘  └─────┼──────┘  └─────┼──────┘             │
│        │               │               │                     │
│  ──────┴───────────────┴───────────────┴──── Network ────    │
│                        │                                     │
│  ┌─────────────────────┴────────────────────────────────┐    │
│  │              Metadata Service (Monitors)              │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  (Paxos/Raft)        │    │
│  │  │Mon 1 │  │Mon 2 │  │Mon 3 │  Cluster map          │    │
│  │  └──────┘  └──────┘  └──────┘  CRUSH rules           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Storage Nodes (OSDs)                     │    │
│  │  ┌───────┐  ┌───────┐  ┌───────┐      ┌───────┐    │    │
│  │  │OSD 1  │  │OSD 2  │  │OSD 3  │ ...  │OSD N  │    │    │
│  │  │NVMe×8 │  │NVMe×8 │  │NVMe×8 │      │NVMe×8 │    │    │
│  │  │       │  │       │  │       │      │       │    │    │
│  │  │PG:    │  │PG:    │  │PG:    │      │PG:    │    │    │
│  │  │1,5,12 │  │2,5,8  │  │1,3,9  │      │4,7,12 │    │    │
│  │  └───────┘  └───────┘  └───────┘      └───────┘    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  Data Placement: CRUSH Algorithm (no central lookup)         │
│    volume_id + block_offset → hash → PG → OSD set            │
└───────────────────────────────────────────────────────────────┘
```

## 4. CRUSH Data Placement

```
CRUSH (Controlled Replication Under Scalable Hashing):

Input: object_id (volume_id + offset)
Output: set of OSDs to store replicas

Algorithm:
  1. Hash object_id → PG (Placement Group)
     pg_id = hash(object_id) % num_placement_groups
     
  2. CRUSH map PG → OSD set
     crush(pg_id, crush_map, rule) → [OSD_primary, OSD_repl1, OSD_repl2]

CRUSH Map (hierarchy):
  root default
    ├── rack1
    │   ├── host1 → [osd.0, osd.1, osd.2]
    │   ├── host2 → [osd.3, osd.4, osd.5]
    │   └── host3 → [osd.6, osd.7, osd.8]
    ├── rack2
    │   ├── host4 → [osd.9, osd.10, osd.11]
    │   └── ...
    └── rack3
        └── ...

CRUSH Rule (3× replication across racks):
  rule replicated_rack {
    step take default
    step chooseleaf firstn 3 type rack    // 1 replica per rack
    step emit
  }

Why CRUSH vs Consistent Hashing:
  - Failure-domain awareness (rack, host)
  - Deterministic: client computes placement locally
  - Weighted nodes: larger OSDs get proportionally more PGs
  - Minimal data movement on node add/remove
```

## 5. Write & Read Path

```
Write Path (3× Replication):

  Client writes block at offset X:
  1. Client: CRUSH(volume_id, offset_X) → PG → [OSD.5, OSD.12, OSD.27]
  2. Client → OSD.5 (primary): WRITE(data)
  3. OSD.5 → OSD.12: REPLICATE(data)     ┐
  4. OSD.5 → OSD.27: REPLICATE(data)     ┤ parallel
  5. OSD.12 → OSD.5: ACK                 │
  6. OSD.27 → OSD.5: ACK                 ┘
  7. OSD.5 writes to local WAL + BlueStore
  8. OSD.5 → Client: ACK (write committed)
  
  Latency: ~500μs (SSD) = journal write + 2 network hops

Read Path:
  1. Client: CRUSH(volume_id, offset_X) → PG → OSD.5 (primary)
  2. Client → OSD.5: READ(offset_X)
  3. OSD.5: read from BlueStore → return data
  4. Latency: ~200μs (single network hop + SSD read)

BlueStore (OSD Storage Engine):
  ┌──────────────────────────────────────┐
  │ BlueStore per OSD                    │
  │                                      │
  │  WAL (Write-Ahead Log): fast NVMe    │
  │  DB: RocksDB for metadata            │
  │  Data: raw block device (no FS)      │
  │                                      │
  │  Write: WAL → RocksDB → data blocks  │
  │  Read: RocksDB lookup → data blocks  │
  │  Checksum: per-block CRC32C          │
  └──────────────────────────────────────┘
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Create Volume (RBD Image)
POST /api/v1/volumes
{
  "name": "vol-abc123",
  "size_gb": 500,
  "pool": "ssd-replicated",        // pool defines CRUSH rule + replication
  "features": ["layering", "exclusive-lock", "fast-diff"],
  "stripe_unit_kb": 4096,          // object size for distribution
  "qos": {
    "max_iops": 50000,
    "max_bw_mbps": 500
  }
}

# Create Snapshot (instant)
POST /api/v1/volumes/{vol_id}/snapshots
{
  "name": "snap-before-upgrade"
}

# Clone from Snapshot (instant, COW)
POST /api/v1/volumes
{
  "name": "vol-clone-test",
  "parent_snapshot": "vol-abc123/snap-before-upgrade",
  "pool": "ssd-replicated"
}

# Resize Volume
PATCH /api/v1/volumes/{vol_id}
{
  "size_gb": 1000               // online grow, no data movement
}

# Attach Volume to Host
POST /api/v1/volumes/{vol_id}/attach
{
  "host": "compute-node-42",
  "access_mode": "rw",           // rw | ro | multi-attach-ro
  "device_path": "/dev/rbd0"
}
```

### Volume Striping & Object Layout

```
Volume Object Layout:
  
  Volume "vol-abc123" (500GB, 4MB stripe unit):
  ┌─────────────────────────────────────────────────────┐
  │ Offset    │ Object Name              │ PG  │ OSDs   │
  │───────────┼──────────────────────────┼─────┼────────│
  │ 0-4MB     │ rbd_data.abc123.00000000 │ 2.5 │ 5,12,27│
  │ 4-8MB     │ rbd_data.abc123.00000001 │ 2.9 │ 3,14,22│
  │ 8-12MB    │ rbd_data.abc123.00000002 │ 2.1 │ 7,19,31│
  │ ...       │                          │     │        │
  │ Thin: only objects with written data exist           │
  │ 500GB thin = maybe 50GB physical (10% allocated)     │
  └─────────────────────────────────────────────────────┘

Snapshot (COW):
  Parent: vol-abc123        Snapshot: snap-before-upgrade
  ┌────────────────────┐    ┌────────────────────┐
  │ obj.00000000 ──────┼────┤ (shared)           │
  │ obj.00000001 (new) │    │ obj.00000001 (old) │  ← COW copy
  │ obj.00000002 ──────┼────┤ (shared)           │
  └────────────────────┘    └────────────────────┘
  
  After write to obj.00000001:
    1. COW: copy old data to snapshot overlay
    2. Write new data to parent object
    3. Space used: only changed objects
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Capacity** | Add nodes → CRUSH rebalances PGs automatically | 10PB+ per cluster |
| **IOPS** | More OSDs = more parallel I/O; client talks directly to OSD | 10M+ aggregate |
| **Volumes** | Each volume = set of RADOS objects distributed across cluster | 100K+ volumes |
| **PG count** | ~100 PGs per OSD; increase total PGs as cluster grows | Millions of PGs |
| **Network** | Client-to-OSD direct (no central bottleneck) | 100Gbps per node |

```
Adding a Node:
  1. New OSDs registered in CRUSH map
  2. PGs gradually migrated to new OSDs (backfill)
  3. Backfill rate-limited to avoid client impact
  4. Result: ~uniform data distribution
  
  Movement: only 1/N of data moves (N = total OSDs)
  Example: 100→101 OSDs = ~1% data movement
```

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Single disk failure** | 3× replication: 2 copies remain; auto-recovery to another OSD |
| **Node failure** | Replicas on different nodes/racks; PG peers elect new primary |
| **Rack failure** | CRUSH rule ensures replicas span racks; no data loss |
| **Silent corruption** | Per-block checksums (CRC32C); deep scrub weekly compares replicas |
| **Partial write** | WAL (write-ahead log) ensures atomicity; replay on recovery |
| **Split-brain** | Paxos-based monitors; require majority for cluster map updates |

**Durability calculation (3× replication, 100 OSDs, 1% annual failure rate):**
P(lose all 3 copies before rebuild) ≈ 10^-11 per year = 11 nines durability

## 9. Latency

| Operation | SSD Tier | HDD Tier |
|-----------|----------|----------|
| 4KB random read | 200-500μs | 5-10ms |
| 4KB random write (3× repl) | 500μs-1ms | 10-20ms |
| 4KB sequential read | 200μs | 2-5ms |
| 128KB sequential write | 1-2ms | 15-25ms |

**Optimization:**
- BlueStore: bypass filesystem (raw block device) — eliminates double-write
- WAL on fast NVMe; data on larger NVMe/SSD
- Client-side caching (read cache in host memory)
- Primary-affinity: route reads to closest OSD (rack-local)

## 10. Reliability

| Failure Mode | Detection | Recovery |
|--------------|-----------|----------|
| **OSD failure** | Heartbeat timeout (20s) | Mark OSD down; PG peers re-replicate; new replica backfilled |
| **Network partition** | Monitor quorum lost | Minority partition: read-only; majority partition: continues R/W |
| **Slow OSD** | Latency tracking + auto-marking | Mark OSD "out" after timeout; migrate PGs to healthy OSDs |
| **Data corruption** | Checksums on read; scrubbing | Auto-repair from healthy replica; alert if all replicas corrupt |

## 11. Availability

**Target: 99.999% per volume**

```
Availability Architecture:
  - 3× replication across failure domains → survive any 2 failures
  - Monitor quorum (3 or 5 nodes) → cluster available with majority
  - PG Peering: if primary OSD dies, secondary promotes in seconds
  - Client retries: transparent failover to new primary OSD
  
Volume Availability:
  P(available) = 1 - P(all 3 replicas down simultaneously)
  With per-OSD availability of 99.9%:
  P(volume available) = 1 - (0.001)^3 = 99.9999999%
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | CephX (shared-secret Kerberos-like); per-client keys |
| **Authorization** | Pool-level ACLs; namespace isolation; capability bits per client |
| **Encryption at rest** | dm-crypt on OSD devices; key management via KMIP/Vault |
| **Encryption in transit** | Messenger v2 protocol with AES-GCM encryption |
| **Multi-tenancy** | Namespace isolation within pools; separate CRUSH rules per tenant |
| **Audit** | All admin operations logged; monitor election history |

## 13. Cost Constraints

**Estimated Cost (3.3PB usable, 100 nodes, 3× replication):**

| Component | Specification | Cost |
|-----------|--------------|------|
| **Storage nodes** | 100× (2×Xeon, 128GB RAM, 8×3.84TB NVMe) | $1,500,000 |
| **Network** | 100Gbps RDMA fabric (leaf-spine) | $300,000 |
| **Monitor nodes** | 5× (general purpose servers) | $25,000 |
| **Software** | Open-source Ceph + support contract | $200,000/yr |
| **Power + cooling** | 100 nodes × $300/month | $30,000/month |
| **Total Year 1** | | **~$2,185,000** |

**Effective $/GB: $0.66/GB usable** (or $0.22/GB raw)

## Key Interview Discussion Points

1. **CRUSH vs consistent hashing?** — CRUSH: deterministic, topology-aware (rack/host), weighted. Consistent hashing: simpler but no failure-domain awareness. CRUSH = better for storage
2. **Why Placement Groups?** — Direct object→OSD mapping requires massive tables. PGs group objects; CRUSH maps PGs to OSDs. ~100 PGs/OSD keeps mapping manageable
3. **Replication vs erasure coding?** — 3× repl: better latency, simpler recovery, more space used. EC (4+2): 50% overhead vs 200%, but higher latency for small I/O. Use repl for hot, EC for cold
4. **How does self-healing work?** — OSD failure → monitors update CRUSH map → PG peers detect missing replica → backfill from surviving copies → cluster returns to full replication
5. **How to handle slow OSDs?** — Monitor I/O latency per OSD; auto-mark slow OSDs as out; migrate PGs. Prevents "thundering herd" where one slow OSD degrades entire pool
