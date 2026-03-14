# Design a Data Replication System

## Overview
Design an enterprise data replication system like NetApp SnapMirror — providing synchronous and asynchronous replication across sites for disaster recovery, data migration, and workload distribution with configurable RPO/RTO guarantees.

## 1. Requirements

**Functional:**
- Synchronous replication (RPO=0) for metro-distance clusters
- Asynchronous replication with configurable schedules (RPO minutes-hours)
- Cascading replication (A → B → C) for multi-site DR
- Fan-out replication (A → B, A → C) for read distribution
- Initial baseline transfer + incremental forever
- Consistency groups for multi-volume application consistency

**Non-Functional:**
- Sync replication latency overhead: <1ms per write
- Network efficiency: block-level dedup + compression on wire
- Failover time (async): RTO < 30 minutes
- Failover time (sync): RTO < 2 minutes
- Support for 100Gbps replication links

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Source data** | 500TB across 200 volumes |
| **Daily change rate** | 2% = 10TB/day |
| **Replication relationships** | 500 (fan-out, cascading) |
| **Sync distance** | ≤100km (metro) |
| **Async distance** | Unlimited (cross-region) |
| **Network bandwidth** | 10Gbps dedicated replication |

## 3. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│              Data Replication Architecture                      │
│                                                                │
│  Site A (Primary)                Site B (DR)                   │
│  ┌──────────────┐                ┌──────────────┐              │
│  │ Controller A  │               │ Controller B  │              │
│  │              │               │              │              │
│  │ ┌──────────┐ │  Sync/Async   │ ┌──────────┐ │              │
│  │ │VolA      │─┼───────────────┼─│VolA_DR   │ │              │
│  │ │(source)  │ │ Intercluster  │ │(dest, RO) │ │              │
│  │ └──────────┘ │    LIFs       │ └──────────┘ │              │
│  │ ┌──────────┐ │               │ ┌──────────┐ │              │
│  │ │VolB      │─┼───────────────┼─│VolB_DR   │ │              │
│  │ │(source)  │ │               │ │(dest, RO) │ │              │
│  │ └──────────┘ │               │ └──────────┘ │              │
│  └──────┬───────┘               └──────┬───────┘              │
│         │                              │                       │
│  ┌──────┴───────┐               ┌──────┴───────┐              │
│  │  Disk Pool   │               │  Disk Pool   │              │
│  │  (aggr)      │               │  (aggr)      │              │
│  └──────────────┘               └──────────────┘              │
│                                                                │
│     ┌───────────────────────────────────┐                      │
│     │     Replication Engine            │                      │
│     │                                   │                      │
│     │  Sync:                            │                      │
│     │    Write → local NVRAM            │                      │
│     │         → remote NVRAM            │                      │
│     │         → ACK only after both     │                      │
│     │                                   │                      │
│     │  Async:                           │                      │
│     │    Snapshot (source)              │                      │
│     │    → Diff (snap_old → snap_new)   │                      │
│     │    → Transfer changed blocks      │                      │
│     │    → Apply at destination          │                      │
│     └───────────────────────────────────┘                      │
│                                                                │
│  Cascading:  A ──async──► B ──async──► C                      │
│  Fan-out:    A ──sync──► B                                     │
│              A ──async──► C                                     │
└────────────────────────────────────────────────────────────────┘
```

## 4. Synchronous Replication (RPO = 0)

```
Sync Write Path:
  Host → Write I/O → Controller A
    1. Write to local NVRAM (Site A)          ────┐
    2. Send to remote Controller B            ────┤ parallel
    3. Write to remote NVRAM (Site B)         ────┘
    4. Remote ACK received
    5. ACK to Host
    
  Total additional latency: ~0.5ms (metro 50km)
  RTT formula: distance_km × 2 / (speed_of_light × 0.7)
  50km → ~0.48ms RTT

Sync Modes:
  StrictSync: If partner unreachable → I/O STOPS (RPO=0 guarantee)
  Sync:       If partner unreachable → continue as async (temporary RPO>0)

Consistency Group (CG):
  ┌─────────────────────────────────────────┐
  │ CG: "oracle_db"                         │
  │   Vol: data_vol  → data_vol_dr          │
  │   Vol: log_vol   → log_vol_dr           │
  │   Vol: ctrl_vol  → ctrl_vol_dr          │
  │                                         │
  │   Write ordering preserved across all   │
  │   volumes → crash-consistent at dest    │
  │                                         │
  │   Fence mechanism: I/O serial number    │
  │   applied atomically to all CG members  │
  └─────────────────────────────────────────┘
```

## 5. Asynchronous Replication (Incremental Forever)

```
Async Transfer Mechanism:
  
  Time T1:                    Time T2:
  ┌──────────────┐            ┌──────────────┐
  │ Source Volume │            │ Source Volume │
  │              │            │              │
  │  snap_T1 ◄──┼── common   │  snap_T2 ◄──│── new snapshot
  │              │  snapshot  │  snap_T1 ◄──│── common (retained)
  └──────────────┘            └──────────────┘
  
  Transfer at T2:
    1. Create snap_T2 on source
    2. Compute diff: blocks_changed(snap_T1 → snap_T2)
    3. Read changed blocks from source
    4. Compress + encrypt on wire
    5. Apply to destination volume
    6. Delete old snap_T1 on source (keep snap_T2 as new common)
    
  Block Diff Algorithm:
    - Compare block fingerprints (wafl_check_generate)
    - Only transfer blocks where fingerprint differs
    - Typically 1-5% of volume changes between snapshots
    
  Schedule Examples:
    RPO 15-min:  Transfer every 15 minutes
    RPO 1-hour:  Transfer hourly
    RPO 1-day:   Transfer daily (often compressed to off-peak hours)
    
  Network Optimization:
    - Dedup: if block exists at destination, skip transfer
    - Compression: LZ4 on wire (2-3× reduction)
    - Throttling: bandwidth limits to avoid saturating links
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Create Replication Relationship
POST /api/v1/snapmirror/relationships
{
  "source": {
    "cluster": "cluster_a",
    "svm": "svm_prod",
    "volume": "data_vol"
  },
  "destination": {
    "cluster": "cluster_b",
    "svm": "svm_dr",
    "volume": "data_vol_dr"
  },
  "policy": "sync_mirror",        // sync_mirror | async_mirror | vault
  "schedule": null,                // null for sync; "15min" for async
  "throttle_kbps": 0              // 0 = unlimited
}

# Initialize (Baseline Transfer)
POST /api/v1/snapmirror/relationships/{uuid}/initialize
{
  "transfer_priority": "normal"    // normal | low
}

# Trigger Manual Update (Async)
POST /api/v1/snapmirror/relationships/{uuid}/update
{
  "transfer_priority": "urgent"
}

# Failover (Break Mirror)
POST /api/v1/snapmirror/relationships/{uuid}/break
# Makes destination read-write; source becomes stale

# Reverse Resync (After Failback)
POST /api/v1/snapmirror/relationships/{uuid}/resync
{
  "direction": "reverse"           // B→A instead of A→B
}
```

### Replication State Machine

```
Relationship States:
  
  uninitialized ──initialize──► transferring ──complete──► snapmirrored
                                    │                         │
                                    │ fail                    │ update
                                    ▼                         ▼
                               broken_off              transferring
                                    │                    │       │
                                    │ resync             │done   │fail
                                    ▼                    ▼       ▼
                               transferring        snapmirrored  idle
                                                                  │
                                                            auto-retry

Transfer Types:
  initialize:  Full baseline (can take hours for large volumes)
  update:      Incremental (snapshot diff, minutes)
  resync:      Re-establish after break (incremental if common snap exists)
  restore:     Reverse transfer: destination → source
```

### Consistency Group Implementation

```
CG Replication Coordinator:

class ConsistencyGroupReplicator:
    def sync_write(self, cg_id, writes: List[Write]):
        """Ensure all CG volumes replicated atomically"""
        cg = self.get_consistency_group(cg_id)
        
        # Assign CG-wide sequence number
        seq = self.cg_sequence_counter.increment()
        
        for write in writes:
            write.cg_sequence = seq
        
        # Sync to all remote volumes atomically
        # Remote applies writes in sequence order
        # If any volume replication fails in StrictSync:
        #   → fence all I/O to CG until resolved
        
    def failover(self, cg_id):
        """Activate all CG destination volumes"""
        cg = self.get_consistency_group(cg_id)
        
        # Find latest common CG snapshot
        latest_cg_snap = self.find_latest_cg_consistent_snap(cg)
        
        # Revert all dest volumes to that snapshot
        for vol in cg.destination_volumes:
            vol.revert_to(latest_cg_snap)
            vol.set_writable()
```

## 7. Scalability

| Dimension | Strategy | Limit |
|-----------|----------|-------|
| **Relationships per cluster** | Parallel replication engines per node | 2,500 relationships |
| **Transfer throughput** | Multi-stream parallel transfer per relationship | 100 Gbps per node |
| **Fan-out degree** | Mirror from single source to multiple destinations | 8 destinations per source |
| **Cascade depth** | Chain replication through intermediate sites | 3 hops (A→B→C→D) |
| **CG size** | Volumes in single consistency group | 40 volumes |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Site A failure (sync)** | RPO=0: all writes ACKed were persisted at Site B; CG ensures app consistency |
| **Site A failure (async)** | RPO = last successful transfer time. Worst case: schedule interval of data loss |
| **Network partition (sync)** | StrictSync halts I/O (no data loss); Sync mode continues locally (potential RPO>0) | 
| **Transfer corruption** | Checksums on every transferred block; retry on mismatch |
| **Common snapshot deleted** | Full re-baseline required; alert before allowing deletion |

## 9. Latency

| Operation | Latency |
|-----------|---------|
| **Sync write overhead** | +0.3-1.0ms (depends on site distance) |
| **Async transfer lag** | Schedule interval (5min - 24hr) |
| **Failover (sync)** | <2 minutes (break mirror + mount) |
| **Failover (async)** | <30 minutes (apply last transfer + break + mount) |
| **Resync after failback** | Minutes (incremental) to hours (full re-baseline) |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Replication link down** | Sync: I/O stops or degrades. Async: transfers queue | Redundant links; auto-fallback sync→async |
| **Destination controller crash** | Pending transfer lost | Resume from last common snapshot |
| **Source snapshot deleted** | Cannot compute incremental diff | Re-baseline; prevent with snapshot locks |
| **Split-brain (sync)** | Both sites think they're primary | Mediator/tiebreaker node at 3rd site; automated fencing |

## 11. Availability

**Target: 99.99% for replication service**

```
HA Architecture:
  - Source + Dest: each behind HA pair → tolerate single controller failure
  - Network: dual replication links (active-active)
  - Mediator: 3rd-site lightweight VM for sync replication tiebreaker
  
Non-Disruptive Ops:
  - Controller upgrade: takeover/giveback, replication auto-resumes
  - Volume migration: move volume, replication relationship follows
  - Network failover: LIF migration to surviving port
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Encryption in transit** | TLS 1.3 for all replication traffic; IPsec for intercluster |
| **Encryption at rest** | Source + destination both encrypted (NAE/NVE) |
| **Authentication** | Cluster peering with mutual certificate authentication |
| **Authorization** | RBAC: replication admin role separate from data access |
| **Compliance** | SnapLock on vault destination: immutable copies for retention |

## 13. Cost Constraints

**Estimated Cost (500TB replicated, two sites):**

| Component | Cost |
|-----------|------|
| **Replication bandwidth** | 10Gbps dedicated WAN link: $15,000/month |
| **Destination storage** | 500TB all-flash (mirrors source): $800,000 CapEx |
| **Destination controllers** | HA pair: $250,000 |
| **Software licenses** | SnapMirror per-TB: $50,000/yr |
| **3rd-site mediator** | Single VM: $200/month |
| **Total Year 1** | **~$1,238,400** |

Cost optimization: thin provisioning on destination; dedup/compress reduces effective capacity.

## Key Interview Discussion Points

1. **Sync vs Async tradeoffs?** — Sync: RPO=0 but latency penalty and distance limit. Async: flexible distance but potential data loss = schedule interval
2. **How does incremental replication work?** — Snapshot-based: diff between common snapshot and new snapshot, transfer only changed blocks
3. **What if common snapshot is lost?** — Must re-baseline (full transfer). Prevent by locking replication snapshots; alert on unexpected deletion
4. **How to handle split-brain in sync replication?** — Tiebreaker/Mediator at 3rd site; automated fencing; STONITH equivalent for storage
5. **Consistency groups why?** — Multi-volume apps (DB data + logs): without CG, volumes could be at different points in time after failover, causing corruption
