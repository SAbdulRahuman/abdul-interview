# Design a Storage Area Network (SAN) System

## Overview
Design an enterprise SAN system like NetApp ONTAP SAN or Pure Storage — providing block-level storage over iSCSI, Fibre Channel, and NVMe-oF protocols with sub-millisecond latency, multipathing, and enterprise data protection.

## 1. Requirements

**Functional:**
- Block storage via iSCSI, FC (Fibre Channel), NVMe-oF
- LUN management: create, map, resize, snapshot, clone
- Multipathing with automatic failover (ALUA)
- SCSI persistent reservations for cluster fencing
- Thin provisioning with space reclamation (UNMAP/TRIM)
- RAID-DP/TEC for data protection

**Non-Functional:**
- Latency: <200μs for 4KB reads (NVMe-oF), <500μs (iSCSI)
- IOPS: 1M+ per HA pair (all-flash)
- Availability: 99.999%
- Non-disruptive failover: <2s I/O pause

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     SAN Architecture                        │
│                                                             │
│  Application Hosts:                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ DB Server │  │ VMware   │  │ K8s Node │                 │
│  │ (Oracle)  │  │ ESXi     │  │          │                 │
│  │ 2× HBA   │  │ 4× HBA   │  │ iSCSI    │                 │
│  └────┬──┬──┘  └──┬──┬────┘  └──┬──┬────┘                 │
│       │  │        │  │          │  │                        │
│    ┌──┴──┴────────┴──┴──────────┴──┴──┐                    │
│    │        SAN Fabric (redundant)     │                    │
│    │  Fabric A ──── FC/IP Switch ────  │                    │
│    │  Fabric B ──── FC/IP Switch ────  │                    │
│    └──┬──┬────────┬──┬──────────┬──┬──┘                    │
│       │  │        │  │          │  │                        │
│  ┌────┴──┴────┐  ┌┴──┴─────────┴──┴──┐                    │
│  │Controller A │  │Controller B       │  (HA Pair)         │
│  │(Active for  │  │(Active for        │                    │
│  │ LUNs 1-50) │  │ LUNs 51-100)      │                    │
│  │ NVRAM      │◄─►│ NVRAM             │  (mirrored)        │
│  └─────┬──────┘  └──────┬────────────┘                    │
│        │                 │                                  │
│  ┌─────┴─────────────────┴──────────┐                      │
│  │     All-Flash Disk Shelves       │                      │
│  │   ┌────────┐  ┌────────┐        │                      │
│  │   │24×NVMe │  │24×NVMe │        │                      │
│  │   │3.84TB  │  │3.84TB  │        │                      │
│  │   └────────┘  └────────┘        │                      │
│  └──────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

## 3. LUN Management & I/O Path

```
LUN (Logical Unit Number) Structure:
┌──────────────────────────────────────────────────────┐
│ LUN /vol/san_vol1/lun0                               │
│   Size: 2TB (thin provisioned, used: 800GB)          │
│   OS Type: linux                                     │
│   Serial: xK4tP-unique-serial                        │
│   Mapped to: igroup "linux_cluster_1"                │
│   LUN ID: 0                                          │
│   Multipath: ALUA (active/optimized + active/non-opt)│
│                                                      │
│   Block Map (extent-based):                          │
│   Logical Block  →  Physical Block                   │
│   0-1023         →  aggr1:blk_50000                  │
│   1024-2047      →  aggr1:blk_72000                  │
│   2048-3071      →  (unallocated - thin provision)   │
│   ...                                                │
│                                                      │
│   Snapshot: COW preserves old blocks                 │
│   Clone: shares blocks with parent (FlexClone)       │
└──────────────────────────────────────────────────────┘

I/O Path (Write):
  Host → HBA → Fabric → Controller Target Port
    → SCSI command decode
    → LUN mapping lookup
    → NVRAM write (persist to battery-backed cache)
    → ACK to host (<200μs for NVMe-oF)
    → Background: coalesce → write to SSD (RAID protected)
```

## 4. Multipathing (ALUA)

```
ALUA (Asymmetric Logical Unit Access):

Host sees LUN via 4 paths:
  Path 1: HBA0 → Fabric A → Controller A (Active/Optimized) ★
  Path 2: HBA0 → Fabric A → Controller B (Active/Non-Optimized)
  Path 3: HBA1 → Fabric B → Controller A (Active/Optimized) ★
  Path 4: HBA1 → Fabric B → Controller B (Active/Non-Optimized)

Normal Operation:
  I/O uses Path 1 + Path 3 (optimized, direct to owning controller)
  Paths 2 + 4 available but traverse inter-controller link

Controller A Failure:
  1. Paths 1 + 3: fail (timeout 15s)
  2. Controller B: takeover (owns LUN now)
  3. Paths 2 + 4: become Active/Optimized
  4. Host multipath driver: reroutes I/O
  5. Total I/O pause: <2 seconds
  6. No data loss (NVRAM mirrored)

Multipath I/O Policies:
  round-robin:   Distribute I/Os across optimized paths
  failover:      Use primary path only, failover on error
  weighted:      Load balance by path bandwidth
```

## 5. Low-Level Design (LLD)

### API Contracts

```
# Create LUN
POST /api/v1/luns
{
  "path": "/vol/san_vol1/lun0",
  "size_gb": 2048,
  "os_type": "linux",
  "thin_provisioned": true,
  "qos_policy": "db_gold",       // max 100K IOPS, min 10K
  "space_reserve": false
}

# Map LUN to Initiator Group
POST /api/v1/luns/{lun_path}/map
{
  "igroup": "linux_cluster_1",
  "lun_id": 0                     // SCSI LUN ID
}

# Create Initiator Group
POST /api/v1/igroups
{
  "name": "linux_cluster_1",
  "protocol": "iscsi",            // iscsi | fcp | nvme_of
  "os_type": "linux",
  "initiators": [
    "iqn.1994-05.com.redhat:server1",
    "iqn.1994-05.com.redhat:server2"
  ]
}

# Create LUN Snapshot
POST /api/v1/volumes/{vol}/snapshots
{ "name": "before_patch" }

# Clone LUN (instant)
POST /api/v1/luns/{lun_path}/clone
{
  "clone_path": "/vol/san_vol1/lun0_test",
  "snapshot": "before_patch"
}
```

### SCSI Protocol Handling

```
SCSI Command Processing:
┌──────────────────────────────────────────────────┐
│ iSCSI/FC/NVMe-oF Target Port receives:         │
│                                                  │
│ SCSI WRITE(10):                                 │
│   LBA: 0x1000, Length: 8 blocks (32KB)          │
│   Data: [payload]                                │
│                                                  │
│ Processing:                                      │
│   1. Lookup LUN mapping → volume + offset       │
│   2. Check SCSI reservations (PR)                │
│      - If persistent reserve: validate I_T nexus │
│   3. Allocate blocks if thin-provisioned         │
│   4. Write to NVRAM (mirrored to partner)        │
│   5. Return GOOD status                          │
│   6. Background: dedup check → compress → flush  │
│                                                  │
│ SCSI Persistent Reservations:                    │
│   Used by cluster software (VMware, Oracle RAC)  │
│   PR types: Write Exclusive, Exclusive Access    │
│   Survive controller failover (state in NVRAM)   │
│   Key: registered_key per initiator              │
│   Preempt: fencing mechanism for split-brain     │
└──────────────────────────────────────────────────┘
```

## 6. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **IOPS** | All-NVMe drives; NVRAM write coalescing | 1M+ IOPS per HA pair |
| **Capacity** | Add disk shelves; scale-out cluster (12 HA pairs) | 100PB per cluster |
| **LUNs** | Thousands of LUNs per controller; FlexVol containers | 12K LUNs per cluster |
| **Bandwidth** | 32Gbps FC / 100GbE iSCSI / NVMe-oF links | 40 GB/s per HA pair |
| **Host Connectivity** | Thousands of initiator sessions per controller | 8K iSCSI sessions |

## 7. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Write Cache** | NVRAM (battery-backed) mirrored to HA partner; survives dual controller failure |
| **Disk Protection** | RAID-DP (double parity, survives 2 disk failures) or RAID-TEC (triple, survives 3) |
| **Replication** | SnapMirror sync (RPO=0) for metro DR; async for long-distance |
| **Snapshots** | Application-consistent snapshots via SnapCenter (DB quiesce + snap) |
| **Data Integrity** | End-to-end checksums (DIF/DIX T10-PI); detect silent data corruption |

## 8. Latency

| Operation | NVMe-oF | iSCSI | FC |
|-----------|---------|-------|-----|
| 4KB random read | 100μs | 300μs | 200μs |
| 4KB random write | 100μs | 300μs | 200μs |
| 64KB seq read | 100μs | 400μs | 250μs |
| 64KB seq write | 100μs | 400μs | 250μs |

**Optimization:** NVRAM write-back, read-ahead for sequential, SSD caching for hot data, NVMe-oF kernel bypass.

## 9. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Controller failure** | I/O pause <2s | HA takeover; NVRAM data preserved; ALUA path switch |
| **Disk failure** | RAID degraded | Auto-reconstruction from parity; hot spare activation |
| **Fabric failure** | Half paths lost | Multipath; dual fabric architecture; remaining paths handle I/O |
| **SCSI reservation loss** | Cluster fencing broken | Persistent reservations survive failover; stored in NVRAM |

## 10. Availability

**Target: 99.999% — non-disruptive operations critical**

```
Non-Disruptive Operations:
  - Controller upgrade: takeover → upgrade → giveback (<2s I/O pause)
  - LUN move: online migration between volumes
  - Volume move: between aggregates, transparent to host
  - Disk firmware update: one disk at a time, RAID protected
```

## 11. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | CHAP (iSCSI), zoning (FC), DH-HMAC-CHAP (NVMe-oF) |
| **Authorization** | LUN masking via igroups; only mapped initiators can access LUNs |
| **Encryption at Rest** | Self-Encrypting Drives (SED) + NSE; AES-256 |
| **Encryption in Transit** | IPsec for iSCSI; FC encryption |
| **Zoning** | FC zoning: isolate initiators from unauthorized targets |
| **Audit** | All LUN operations logged; management CLI/API audit trail |

## 12. Cost Constraints

**Estimated Cost (500TB usable, all-flash, HA pair):**

| Component | Specification | Cost |
|-----------|--------------|------|
| **Controllers** | 2× high-end (HA pair), 768GB RAM | $250,000 |
| **NVMe SSDs** | 48× 15.36TB NVMe | $300,000 |
| **FC HBAs + Switches** | 32Gbps FC dual fabric | $80,000 |
| **Software License** | SAN protocols, SnapMirror, encryption | $150,000/yr |
| **Total (CapEx + 3yr SW)** | | **~$1,080,000** |

**Effective $/IOPS: ~$1.08/IOPS** (at 1M IOPS capacity)

## Key Interview Discussion Points

1. **iSCSI vs FC vs NVMe-oF?** — iSCSI: cheapest (Ethernet), highest latency. FC: proven, low latency. NVMe-oF: lowest latency, newest. Choose based on workload sensitivity
2. **How does ALUA work?** — Active/Optimized path = direct to owning controller. Active/Non-Optimized = indirect via inter-controller link. Host multipath driver prefers optimized
3. **RAID-DP vs RAID-TEC?** — RAID-DP: 2 parity (98.6% efficiency at 16+2). RAID-TEC: 3 parity (better for large SSDs where rebuild times are long)
4. **How do SCSI reservations survive failover?** — Reservation state stored in NVRAM; mirrored to partner; restored during takeover
5. **Thin provisioning risks?** — Overcommitment: if all LUNs fill simultaneously, aggregate runs out of space. Mitigation: autogrow, space monitoring, guarantees
