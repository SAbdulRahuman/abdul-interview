# Design a Snapshot and Clone Management System

## Overview
Design an enterprise snapshot and clone management system like NetApp SnapCenter / FlexClone — providing application-consistent snapshots, instant space-efficient clones, and automated lifecycle management for databases, VMs, and file systems.

## 1. Requirements

**Functional:**
- File system snapshots: instant, point-in-time, read-only images
- Application-consistent snapshots: quiesce DB/VM before snap
- Instant clones: writable copies sharing parent data (zero copy)
- Snapshot scheduling: hourly, daily, weekly retention policies
- Snapshot restore: revert volume to any snapshot in seconds
- Single-file restore: recover individual files from snapshots

**Non-Functional:**
- Snapshot creation: <1 second
- Clone creation: <5 seconds regardless of data size
- Space overhead: zero until divergence (COW/ROW)
- Max snapshots per volume: 1,024
- Clone depth: unlimited (clones of clones)

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Volumes** | 5,000 across cluster |
| **Avg snapshots per volume** | 50 |
| **Total snapshots** | 250,000 |
| **Avg volume size** | 1TB |
| **Active clones** | 2,000 |
| **Snapshot creation rate** | 5,000/hour (scheduled) |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│           Snapshot & Clone Management                          │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  SnapCenter Server (Orchestrator)                     │    │
│  │                                                       │    │
│  │  ┌────────┐  ┌────────────┐  ┌────────────────────┐  │    │
│  │  │Schedule│  │App Plugins │  │Lifecycle Manager   │  │    │
│  │  │Engine  │  │Oracle/SQL/ │  │(retention, purge)  │  │    │
│  │  │        │  │VMware/SAP  │  │                    │  │    │
│  │  └───┬────┘  └─────┬──────┘  └────────┬───────────┘  │    │
│  │      │              │                  │              │    │
│  └──────┼──────────────┼──────────────────┼──────────────┘    │
│         │              │                  │                    │
│  ┌──────▼──────────────▼──────────────────▼──────────────┐   │
│  │  Storage Controller (ONTAP)                            │   │
│  │                                                        │   │
│  │  Volume: prod_db_data (1TB)                           │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │ Active File System                               │  │   │
│  │  │ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      │  │   │
│  │  │ │Blk A│ │Blk B│ │Blk C│ │Blk D│ │Blk E│      │  │   │
│  │  │ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘      │  │   │
│  │  │    │       │       │       │       │          │  │   │
│  │  │ ┌──▼───────▼───────▼───────▼───────▼──┐       │  │   │
│  │  │ │     Shared Block Pool (WAFL)        │       │  │   │
│  │  │ │  (blocks shared between active +    │       │  │   │
│  │  │ │   snapshots + clones)               │       │  │   │
│  │  │ └─────────────────────────────────────┘       │  │   │
│  │  │    ▲       ▲       ▲                          │  │   │
│  │  │ ┌──┴──┐ ┌──┴──┐ ┌──┴──┐                      │  │   │
│  │  │ │Snap │ │Snap │ │Snap │  Read-only snapshots  │  │   │
│  │  │ │hourly│ │daily│ │weekly│                      │  │   │
│  │  │ └─────┘ └─────┘ └─────┘                      │  │   │
│  │  │    ▲                                           │  │   │
│  │  │ ┌──┴────────┐                                  │  │   │
│  │  │ │FlexClone  │  Writable clone (test env)      │  │   │
│  │  │ │(shares all│  Only changed blocks new        │  │   │
│  │  │ │blocks)    │                                  │  │   │
│  │  │ └───────────┘                                  │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘
```

## 4. WAFL Snapshot Implementation

```
Write-Anywhere File Layout (WAFL) Snapshots:

Key Insight: Snapshots are FREE at creation time.
  WAFL never overwrites blocks in-place.
  Writes go to new locations → old blocks naturally preserved.

Snapshot Creation (~1 second):
  1. Freeze inode file (momentary pause <100ms)
  2. Record current root inode pointer as snapshot anchor
  3. Resume I/O
  
  Result: snapshot = pointer to root inode at that moment
  All blocks below that root are now referenced by snapshot

  Before snapshot:         After snapshot:
  ┌─────────┐              ┌─────────┐  ┌──────────┐
  │Root     │              │Root     │  │Snap Root │
  │Inode    │              │Inode    │  │(frozen)  │
  └────┬────┘              └────┬────┘  └────┬─────┘
       │                        │             │
  ┌────▼────┐              ┌────▼────┐  ┌────▼─────┐
  │Inode    │              │Inode    │  │Inode     │
  │File     │              │File    '│  │File      │
  └───┬──┬──┘              └───┬──┬──┘  └───┬──┬───┘
      │  │                     │  │          │  │
   BlkA BlkB                BlkA' BlkB    BlkA BlkB
   (shared)                (new) (shared) (preserved)

After write to file (modifies BlkA):
  - New BlkA' written to new location
  - Active inode points to BlkA' (new)
  - Snapshot inode still points to BlkA (original)
  - BlkB shared between active and snapshot
  - Space used: only BlkA' is additional (4KB)

Snapshot Deletion:
  1. Remove snapshot anchor (root pointer)
  2. Walk snapshot tree
  3. For each block: decrement reference count
  4. If ref_count == 0 → free block
  5. If ref_count > 0 → still used by other snapshot or active FS
  6. Duration: proportional to unique blocks (fast for recent snaps)
```

## 5. FlexClone (Instant Writable Clone)

```
FlexClone Architecture:

Clone creation (< 5 seconds, any volume size):
  1. Create snapshot of parent volume (if not from existing snap)
  2. Create new volume metadata pointing to snapshot blocks
  3. Mark clone as writable
  4. Done — zero data copied

  Parent Volume (1TB)          Clone Volume (1TB apparent)
  ┌─────────────────┐          ┌─────────────────┐
  │ Active FS       │          │ Clone FS        │
  │                 │          │                 │
  │ BlkA' (modified)│          │ BlkA (from snap)│ ← reads parent
  │ BlkB (shared)───┼──────────┼─BlkB (shared)  │
  │ BlkC (shared)───┼──────────┼─BlkC (shared)  │
  │ BlkD' (modified)│          │ BlkD (from snap)│ ← reads parent
  │                 │          │ BlkX (clone-own)│ ← new write
  └─────────────────┘          └─────────────────┘
  
  Physical space used by clone: only BlkX (new writes)
  
Clone Write Path:
  1. Read from clone: check clone block map → miss → read from parent snapshot
  2. Write to clone: allocate new block → write → update clone block map
  3. No COW needed: parent snapshot blocks are immutable

Clone Use Cases:
  - Dev/test databases: clone 10TB production → ready in 5 seconds
  - CI/CD pipelines: clone per branch for integration testing
  - Reporting: clone production for analytics without impacting prod
  - N clones of 1TB parent = 1TB + N × (delta per clone)
  - 100 clones × 5% divergence = 1TB + 5TB = 6TB (not 100TB!)
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Create Application-Consistent Snapshot
POST /api/v1/snapcenter/snapshots
{
  "resource_group": "oracle_prod",
  "resources": [
    {"type": "oracle_database", "host": "db-01", "sid": "PROD"},
    {"type": "volume", "cluster": "ontap-01", "volume": "db_data"},
    {"type": "volume", "cluster": "ontap-01", "volume": "db_log"}
  ],
  "pre_script": "oracle_hot_backup_begin.sh",
  "post_script": "oracle_hot_backup_end.sh",
  "label": "daily",
  "retention": {"count": 30}
}

# App-Consistent Snapshot Workflow:
#   1. Connect to DB host via plugin
#   2. Run pre_script: ALTER DATABASE BEGIN BACKUP
#   3. Create storage snapshots (all volumes atomically)
#   4. Run post_script: ALTER DATABASE END BACKUP
#   5. Catalog snapshot in SnapCenter

# Create FlexClone
POST /api/v1/volumes/{parent}/clone
{
  "clone_name": "db_data_test_env",
  "parent_snapshot": "daily.2024-01-15_0100",
  "junction_path": "/test/db_data",
  "space_guarantee": "none",            // thin
  "qos_policy": "test_workload"
}
Response: { "status": "created", "time_ms": 3200, "space_used": "0 bytes" }

# Restore Volume from Snapshot
POST /api/v1/volumes/{vol}/restore
{
  "snapshot": "daily.2024-01-14_0100",
  "type": "volume_revert"               // volume_revert | single_file
}

# Single File Restore
POST /api/v1/volumes/{vol}/restore
{
  "snapshot": "hourly.2024-01-15_1400",
  "type": "single_file",
  "source_path": "/data/important.docx",
  "dest_path": "/data/important_restored.docx"
}

# Set Snapshot Policy (automated schedule)
POST /api/v1/snapshot-policies
{
  "name": "standard",
  "schedules": [
    {"type": "hourly",  "count": 6, "minute": 5},
    {"type": "daily",   "count": 14, "hour": 1},
    {"type": "weekly",  "count": 4,  "day": "sunday", "hour": 2},
    {"type": "monthly", "count": 6,  "day": 1, "hour": 3}
  ]
}
```

### Block Reference Counting Internals

```
WAFL Block Reference Management:

struct BlockEntry {
    physical_addr: u64,          // physical disk address
    checksum: u32,               // CRC32C
    ref_count: u16,              // active FS + snapshots + clones
    flags: u16,                  // compressed, deduped, etc.
}

Reference Count Scenarios:
  ref_count=1: Only active FS uses this block
    → can be overwritten (but WAFL writes elsewhere anyway)
    → freed if deleted
    
  ref_count=3: Active FS + 2 snapshots reference this block
    → cannot be freed until all 3 references release
    → overwrite in active FS: write new block, old block ref_count stays 3→2
    → delete oldest snapshot: ref_count 3→2
    → delete newest snapshot: ref_count 3→2
    → all deleted: ref_count → 0 → free

  ref_count overflow protection:
    → u16 max = 65,535 references per block
    → if exceeded: block "pinned" (never freed until manually resolved)

Snapshot Space Accounting:
  vol.space.active = blocks_with_no_snapshot_refs
  vol.space.snapshot_used = blocks_referenced_only_by_snapshots
  vol.space.shared = blocks_referenced_by_both_active_and_snapshots
  
  Deleting a snapshot frees: blocks where ref_count drops to 0
  NOT: blocks still referenced by other snapshots or active FS
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Snapshots per volume** | WAFL anchor pointers; minimal metadata | 1,024 snapshots |
| **Clones per parent** | Shared block pool; only deltas stored | Unlimited (practically 500+) |
| **Clone depth** | Clones of clones; flattened block resolution | No limit (slight read overhead) |
| **Restore speed** | Volume revert: swap root pointer (instant) | Any size, <1 minute |
| **Concurrent snapshots** | Consistency group: atomic multi-volume snapshot | 40 volumes per CG |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Accidental deletion** | Snapshot retention policy preserves N copies; cannot delete locked snaps |
| **Ransomware/malware** | SnapLock on snapshot copies → immutable; restore from pre-infection snapshot |
| **Application corruption** | App-consistent snapshot: DB quiesced → crash-consistent restore possible |
| **Snapshot space exhaustion** | Auto-delete oldest snapshots; alert before snapshot reserve full |
| **Clone parent snapshot deleted** | Split clone from parent first; or error: "snapshot busy — clone references it" |

## 9. Latency

| Operation | Latency |
|-----------|---------|
| **Create snapshot** | <1 second (freeze + anchor pointer) |
| **Create clone (FlexClone)** | <5 seconds (metadata only) |
| **Volume restore (snap revert)** | <1 minute (swap root pointer + replay) |
| **Single file restore** | Seconds (copy file from .snapshot directory) |
| **Read from clone** | Same as parent (<200μs for SSD) |
| **Delete oldest snapshot** | Seconds-minutes (proportional to unique blocks) |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Snapshot during write storm** | Brief I/O pause (<100ms) | WAFL write-anywhere: minimal freeze window |
| **Snapshot space overflow** | Volume goes offline if reserve + data = 100% | Autodelete; fractional reserve; monitoring |
| **Clone divergence too high** | Clone becomes large; space pressure | Monitor clone space; split or collapse when appropriate |
| **Controller failure during snapshot** | Snapshot atomic: either created or not | NVRAM ensures metadata consistency; retry on recovery |

## 11. Availability

**Target: 99.999% — snapshots/clones are core data services**

```
Availability Architecture:
  - Snapshot creation survives controller failover (NVRAM metadata)
  - Clone volumes survive failover (same as regular volumes)
  - SnapCenter HA: active-passive with shared DB
  - Self-service restore: users access .snapshot directory directly
  - RPO with snapshots: as frequent as every 5 minutes
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Snapshot tampering** | SnapLock Compliance: snapshots immutable until retention expires |
| **Clone access control** | Clones inherit parent ACLs; can be customized post-creation |
| **Snapshot access** | .snapshot directory accessible to volume users; admin can hide |
| **Encryption** | Snapshots/clones encrypted if parent volume is encrypted (same key) |
| **Compliance** | SnapLock vaulted snapshots for SEC 17a-4 compliance; WORM protection |

## 13. Cost Constraints

**Cost Comparison: Snapshots + Clones vs Traditional Backup + Full Copies:**

| Approach | 10TB DB, 14-day retention, 5 test copies |
|----------|------------------------------------------|
| **Traditional** | 14 full backups (140TB) + 5 full copies (50TB) = 190TB |
| **Snapshots + Clones** | 10TB + 14 snaps (7TB delta) + 5 clones (2.5TB delta) = 19.5TB |
| **Space savings** | **~90%** |
| **Time savings** | Backup: hours → Snap: 1 second. Clone: hours → FlexClone: 5 seconds |

| Component | Cost |
|-----------|------|
| **SnapCenter Server** | Included with ONTAP license |
| **Snapshot storage overhead** | ~30-50% of volume for 30 snapshots (included in existing capacity) |
| **Clone storage overhead** | 5-10% per clone (only deltas) |
| **Software license** | SnapCenter included; SnapManager per-app: $5,000/yr |

## Key Interview Discussion Points

1. **COW vs ROW snapshots?** — Copy-On-Write: copy old block before overwrite (read penalty). Redirect-On-Write (WAFL): write to new location always → snapshot is free, no COW overhead. ROW is superior for write-heavy workloads
2. **How does FlexClone achieve instant cloning?** — Share all blocks with parent snapshot; only new writes use new blocks. No data copy = instant creation regardless of data size
3. **Snapshot space management?** — Snapshots consume space as active FS diverges. Monitor with `df` showing snapshot usage. Auto-delete oldest snapshots at threshold. Fractional reserve for write buffer
4. **Application consistency?** — File system snapshot = crash-consistent. App-consistent = quiesce app + snapshot + resume. For Oracle: BEGIN BACKUP. For VMware: quiesce VM tools + VADP snapshot
5. **Clone depth performance impact?** — Deep clone chains (A→B→C→D): reads traverse chain to find block. Mitigated by block map caching. Can "split" clone to become independent (background copy)
