# Design a Backup and Snapshot System

Examples: NetApp Snapshot, ZFS snapshots, Veeam, AWS Backup

---

## 1. Requirements

### Functional
- Create point-in-time snapshots of volumes/datasets
- Schedule automatic snapshots (hourly, daily, weekly)
- Restore files or entire volumes from snapshots
- Incremental backups (only changed blocks)
- Retention policies (keep last N snapshots, expire old ones)

### Non-Functional
- Near-zero overhead for snapshot creation
- Minimal impact on production I/O
- Space-efficient (share unchanged data)
- Fast restore (minutes, not hours)
- Consistent snapshots for databases

---

## 2. High-Level Architecture

```
  ┌────────────────────────────────────────────────────┐
  │                 Snapshot Manager                     │
  │                                                    │
  │  ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │
  │  │  Scheduler   │  │  Retention   │  │  Restore  │ │
  │  │  (cron-like) │  │  Policy Mgr  │  │  Engine   │ │
  │  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
  │         │                │               │         │
  └─────────┼────────────────┼───────────────┼─────────┘
            │                │               │
  ┌─────────▼────────────────▼───────────────▼─────────┐
  │              Storage Engine (WAFL / ZFS / COW)      │
  │                                                    │
  │  ┌─────────────────────────────────────────┐       │
  │  │  Active File System                      │       │
  │  │  ┌─────┬─────┬─────┬─────┬─────┐       │       │
  │  │  │ B1  │ B2  │ B3  │ B4  │ B5  │       │       │
  │  │  └─────┴─────┴─────┴─────┴─────┘       │       │
  │  │                                         │       │
  │  │  Snapshot Chain                          │       │
  │  │  snap1 → snap2 → snap3 (active)        │       │
  │  └─────────────────────────────────────────┘       │
  │                                                    │
  │  ┌──────────────────────┐                          │
  │  │  Backup Target       │                          │
  │  │  (remote, cloud, tape)│                          │
  │  └──────────────────────┘                          │
  └────────────────────────────────────────────────────┘
```

---

## 3. Copy-on-Write (COW) Snapshots

```
  Before snapshot:
  ┌─────────────────────────────────┐
  │  Active FS                      │
  │  Block ptrs: [B1][B2][B3][B4]  │
  └─────────────────────────────────┘

  Create snapshot (instant!):
  ┌─────────────────────────────────┐
  │  Snapshot S1 (frozen pointer)   │
  │  Block ptrs: [B1][B2][B3][B4]  │ ← shared with active
  └─────────────────────────────────┘
  ┌─────────────────────────────────┐
  │  Active FS                      │
  │  Block ptrs: [B1][B2][B3][B4]  │ ← same blocks
  └─────────────────────────────────┘

  After write to B2:
  ┌─────────────────────────────────┐
  │  Snapshot S1 (still points to   │
  │  original block B2)             │
  │  Block ptrs: [B1][B2][B3][B4]  │
  └─────────────────────────────────┘
  ┌─────────────────────────────────┐
  │  Active FS (B2 modified)        │
  │  Block ptrs: [B1][B2'][B3][B4] │ ← B2' written to NEW location
  └─────────────────────────────────┘

  Key insight: Snapshot creation is O(1)
  - Just freeze the current block pointers
  - No data is copied at snapshot time
  - Old blocks preserved as long as any snapshot references them
```

---

## 4. COW vs Redirect-on-Write (ROW)

```
  Copy-on-Write (COW):
  ┌──────────────────────────────────────────────┐
  │  1. Read original block                      │
  │  2. Copy original to snapshot reserve area   │
  │  3. Write new data to original location      │
  │                                              │
  │  Write penalty: 1 extra read + 1 extra write │
  │  Read: no penalty (data stays in place)      │
  │  Used by: VMware, traditional COW systems    │
  └──────────────────────────────────────────────┘

  Redirect-on-Write (ROW):
  ┌──────────────────────────────────────────────┐
  │  1. Write new data to NEW location           │
  │  2. Update pointer to new location           │
  │  3. Old location preserved for snapshot      │
  │                                              │
  │  Write: no extra copy (write anywhere)       │
  │  Read: may need to follow pointer chain      │
  │  Used by: NetApp WAFL, ZFS                   │
  └──────────────────────────────────────────────┘

  ROW is superior for write-heavy workloads
  because it avoids the extra read+write penalty.
```

---

## 5. Snapshot Chain Management

```
  Snapshot chain over time:

  Time ───────────────────────────────────▶

  S1 (Mon)    S2 (Tue)    S3 (Wed)    Active
    │            │            │          │
    ▼            ▼            ▼          ▼
  [B1,B2,B3]  [B1,B2',B3] [B1,B2',B3'] [B1,B2'',B3']
                  ▲                        ▲
              B2 changed              B2 changed again

  Block reference counting:
  ┌─────────────────────────────────┐
  │  Block B1: ref_count = 4       │  ← shared by all
  │  Block B2: ref_count = 1       │  ← only S1
  │  Block B2': ref_count = 2      │  ← S2 and S3
  │  Block B2'': ref_count = 1     │  ← Active only
  │  Block B3: ref_count = 2       │  ← S1 and S2
  │  Block B3': ref_count = 2      │  ← S3 and Active
  └─────────────────────────────────┘

  Deleting S1:
  • Decrement ref_count on B2 and B3
  • B2 ref_count → 0 → free block
  • B3 ref_count → 1 → keep (still used by S2)
```

---

## 6. Incremental Backup

```
  Full backup + Incremental chain:

  ┌──────────────┐
  │ Full Backup  │  ← all blocks (Sunday)
  │ 100 GB       │
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │ Incr Mon     │  ← only changed blocks (2 GB)
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │ Incr Tue     │  ← only changed blocks (3 GB)
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │ Incr Wed     │  ← only changed blocks (1 GB)
  └──────────────┘

  Restore Wed = Full + Mon + Tue + Wed (apply in order)

  Incremental Forever (snapshot-based):
  ┌──────────────────────────────────────────────┐
  │  Snap S1 → Snap S2: find changed blocks     │
  │  Use block-level differencing:               │
  │  • Compare block maps of S1 and S2           │
  │  • Only transfer changed/new blocks          │
  │  • No need for periodic full backup!         │
  │  • Restore: mount snapshot directly           │
  └──────────────────────────────────────────────┘
```

---

## 7. Application-Consistent Snapshots

```
  Problem: Database in middle of transaction during snapshot
           → restored snapshot has inconsistent state

  Solution: Quiesce application before snapshot

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
  │ 1. Flush │───▶│ 2. Freeze│───▶│ 3. Create│───▶│ 4. Thaw │
  │ buffers  │    │ I/O      │    │ snapshot │    │ resume  │
  │          │    │ (< 1s)   │    │ (instant)│    │ I/O     │
  └──────────┘    └──────────┘    └──────────┘    └─────────┘

  For databases:
  • MySQL: FLUSH TABLES WITH READ LOCK → snapshot → UNLOCK
  • PostgreSQL: pg_start_backup() → snapshot → pg_stop_backup()
  • VSS (Windows): Volume Shadow Copy Service coordinates with apps
  • NetApp SnapManager: plugin for Oracle, SQL Server, SAP HANA
```

---

## 8. Retention Policies

```
  ┌──────────────────────────────────────────────┐
  │  GFS (Grandfather-Father-Son) Policy:        │
  │                                              │
  │  Hourly:   keep last 24   (1 day)            │
  │  Daily:    keep last 30   (1 month)          │
  │  Weekly:   keep last 12   (3 months)         │
  │  Monthly:  keep last 12   (1 year)           │
  │                                              │
  │  Total snapshots: 24 + 30 + 12 + 12 = 78    │
  │  Covers 1 year of history                    │
  │                                              │
  │  Cleanup process:                            │
  │  • Every hour: create hourly, delete oldest  │
  │  • Promote one hourly to daily at midnight    │
  │  • Promote one daily to weekly on Sunday      │
  │  • Promote one weekly to monthly on 1st       │
  └──────────────────────────────────────────────┘
```

---

## 9. Space Estimation

```
  Volume size: 1 TB
  Daily change rate: 5%
  Snapshot retention: 30 days

  Worst case space:
  30 snapshots × 5% change × 1 TB = 1.5 TB for snapshots
  Total: 1 TB (active) + 1.5 TB (snapshots) = 2.5 TB

  With dedup across snapshots:
  Many changed blocks are duplicates → actual ~1.8 TB total

  COW/ROW benefit:
  Unchanged blocks are shared → space = only delta
```

---

---

## 10. Low-Level Design (LLD)

### API Contract
```
  Snapshot Management:
  POST   /api/v1/volumes/{vol_id}/snapshots
         Body: { "name": str, "type": "COW|ROW", "app_consistent": bool }
         Response: { "snapshot_id": uuid, "created_at": ts, "size_bytes": 0 }

  GET    /api/v1/volumes/{vol_id}/snapshots
         Response: [{ "id": uuid, "name": str, "created_at": ts, "used_bytes": int }]

  POST   /api/v1/volumes/{vol_id}/snapshots/{snap_id}/restore
         Body: { "target_volume": str | null }  (null = in-place restore)

  DELETE /api/v1/volumes/{vol_id}/snapshots/{snap_id}

  Backup Management:
  POST   /api/v1/backups
         Body: { "source_volume": str, "destination": "s3://bucket/path",
                 "type": "full|incremental", "schedule": "0 2 * * *" }

  GET    /api/v1/backups/{backup_id}/status
         Response: { "state": "running|completed|failed", "bytes_transferred": long,
                     "duration_seconds": int }

  POST   /api/v1/backups/{backup_id}/restore
         Body: { "target_volume": str, "point_in_time": timestamp | null }
```

### Snapshot Internal Structures
```
  COW Snapshot Block Map:
  ┌──────────────────────────────────────────────┐
  │  Volume block table:                         │
  │  Block 0 → Disk offset 0x1000               │
  │  Block 1 → Disk offset 0x2000               │
  │  Block 2 → Disk offset 0x3000               │
  │                                              │
  │  Snapshot S1 (freeze):                       │
  │  S1_map = copy of block table at time T      │
  │                                              │
  │  On write to Block 1 after snapshot:         │
  │  1. Read old Block 1 → write to COW area     │
  │  2. Write new data to Block 1 offset         │
  │  3. S1_map[1] → COW area (preserves old data)│
  └──────────────────────────────────────────────┘

  Incremental Backup Change Tracking:
  ┌──────────────────────────────────────────────┐
  │  Changed Block Bitmap (CBT):                 │
  │  Bit per 4 KB block: 0 = unchanged, 1 = dirty│
  │  1 TB volume = 256M blocks = 32 MB bitmap    │
  │                                              │
  │  Incremental backup:                         │
  │  1. Read CBT since last backup               │
  │  2. Transfer only dirty blocks to target     │
  │  3. Reset CBT after successful transfer     │
  │  4. Store backup manifest with block mapping │
  └──────────────────────────────────────────────┘
```

---

## 11. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Snapshot Scaling:                                      │
  │  • Up to 1023 snapshots per volume (NetApp WAFL)        │
  │  • Minimal overhead per snapshot (metadata only)        │
  │  • ROW: O(1) snapshot creation regardless of vol size   │
  │                                                         │
  │  Backup Scaling:                                        │
  │  • Parallel backup streams: 1 per volume, 8 per node   │
  │  • Deduplicated backup: 10:1 reduction typical          │
  │  • Incremental forever: only first backup is full       │
  │  • Network throttling: limit to 50% link capacity      │
  │                                                         │
  │  Multi-Volume:                                          │
  │  • Consistency group snapshots across volumes            │
  │  • Orchestrator schedules backups across 1000s of vols  │
  │  • Priority queue: critical volumes first                │
  │                                                         │
  │  Target Storage:                                        │
  │  • Object store (S3): unlimited capacity for backups   │
  │  • Dedup + compression at backup target: 20:1 effective │
  │  • Cross-region backup replication for DR               │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Snapshot Integrity:                                    │
  │  • Snapshots are crash-consistent by default            │
  │  • App-consistent: quiesce apps before snapshot         │
  │  • Atomic pointer update: snapshot either exists or not │
  │  • No partial snapshots                                 │
  │                                                         │
  │  Backup Verification:                                   │
  │  • Checksum per block: verify on write + periodic scrub │
  │  • Backup validation: post-backup integrity check       │
  │  • Test restore: monthly fire drill to verify recoverability│
  │  • Manifest integrity: cryptographic hash of backup set │
  │                                                         │
  │  RPO Scenarios:                                         │
  │  • Continuous snapshots (every 15 min): RPO = 15 min    │
  │  • Hourly backups to remote: RPO = 1 hour               │
  │  • Synchronous replication + snapshots: RPO ≈ 0         │
  │                                                         │
  │  Protection Against:                                    │
  │  • Ransomware: immutable snapshots (SnapLock/WORM)      │
  │  • Accidental deletion: soft delete with retention      │
  │  • Corruption: multiple snapshot points to roll back to │
  │  • Site disaster: off-site backup + cross-region repl   │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Latency

```
  Operation Latency:
  ┌──────────────────────────┬──────────┬──────────┐
  │ Operation                │ Typical  │ Notes    │
  ├──────────────────────────┼──────────┼──────────┤
  │ Create snapshot (ROW)    │ < 1 ms   │ O(1)     │
  │ Create snapshot (COW)    │ < 10 ms  │ metadata │
  │ Delete snapshot          │ 100 ms   │ space    │
  │                          │          │ reclaim  │
  │ Full backup (1 TB)       │ 30-60 min│ network  │
  │ Incremental backup (50GB)│ 5-10 min │ network  │
  │ Restore from snapshot    │ < 1 s    │ metadata │
  │ Restore from backup      │ 30-60 min│ transfer │
  └──────────────────────────┴──────────┴──────────┘

  Snapshot Impact on Normal I/O:
  ┌─────────────────────────────────────────────────────────┐
  │  COW: first write after snapshot incurs extra I/O       │
  │    (read old → write to COW area → write new data)      │
  │    Impact: ~30% write latency increase temporarily      │
  │                                                         │
  │  ROW: no impact on writes (new data goes to new block)  │
  │    Reads may need to follow pointers (slight overhead)  │
  │    Periodic defragmentation addresses this              │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Crash during snapshot  │ Atomic pointer: snapshot either │
  │                        │ fully created or not at all     │
  ├────────────────────────┼─────────────────────────────────┤
  │ Backup target failure  │ Retry with exponential backoff; │
  │                        │ resume from checkpoint          │
  ├────────────────────────┼─────────────────────────────────┤
  │ Corrupted backup       │ Block-level checksums; detect   │
  │                        │ during restore; keep multiple   │
  │                        │ backup generations              │
  ├────────────────────────┼─────────────────────────────────┤
  │ COW area exhaustion    │ Reserve 20% space for COW;      │
  │                        │ alert at 80%; auto-delete oldest│
  │                        │ snapshot if critical            │
  ├────────────────────────┼─────────────────────────────────┤
  │ Snapshot chain too deep│ Consolidate old snapshots;      │
  │                        │ merge metadata periodically     │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 15. Availability

```
  Target: 99.99% for snapshot operations

  ┌─────────────────────────────────────────────────────────┐
  │  Snapshot Availability:                                 │
  │  • Snapshots are local → always available with volume   │
  │  • No dependency on external services for snapshot      │
  │  • Restore is instant (pointer swap for ROW)            │
  │                                                         │
  │  Backup Availability:                                   │
  │  • Backup target: multi-AZ object store (99.99%)        │
  │  • Backup service: 2+ orchestrator instances (HA)      │
  │  • Failed backup → retry; alert after 3 failures       │
  │                                                         │
  │  Restore Scenarios:                                     │
  │  • Fastest: revert to local snapshot (seconds)          │
  │  • Medium: restore from local backup (minutes)          │
  │  • Slowest: restore from remote backup (hours)          │
  │  • RTO target: < 1h for most workloads                 │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Snapshot Security:                                     │
  │  • Snapshots inherit source volume permissions          │
  │  • SnapLock (WORM): prevent snapshot deletion           │
  │  • Compliance mode: even admin cannot delete before     │
  │    retention expiry                                     │
  │                                                         │
  │  Backup Encryption:                                     │
  │  • At-rest: AES-256 encryption of backup data          │
  │  • In-transit: TLS 1.3 for backup transfer             │
  │  • Key management: per-backup-policy keys via KMS      │
  │  • Customer-managed keys supported                      │
  │                                                         │
  │  Access Control:                                        │
  │  • RBAC: separate roles for snapshot-create, restore,   │
  │    delete, and backup-admin                             │
  │  • Audit: log all snapshot/backup operations            │
  │  • MFA required for snapshot deletion (optional)        │
  │                                                         │
  │  Anti-Ransomware:                                       │
  │  • Immutable snapshots with configurable lock period    │
  │  • Air-gapped backup copies (offline/vault)             │
  │  • Anomaly detection: unusual delete/encrypt patterns   │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Snapshot Costs (minimal):                              │
  │  • Only store changed blocks (delta from live)          │
  │  • Typical: 5-10% of volume size per snapshot           │
  │  • 10 snapshots on 1 TB volume ≈ 500 GB-1 TB extra    │
  │                                                         │
  │  Backup Costs:                                          │
  │  • Full backup: ~1 TB per 1 TB volume                   │
  │  • Incremental: 5-10% change rate → 50-100 GB/day      │
  │  • Deduplication: 5-10× reduction                       │
  │  • Compression: additional 2-3× reduction               │
  │  • Effective cost: $0.01/GB/month on cold storage      │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • GFS retention: fewer old backups (grandfather-father-│
  │    son) → reduce storage growth                          │
  │  • Expire old snapshots automatically                    │
  │  • Backup to cold storage (Glacier): 80% cheaper       │
  │  • Deduplicated backup target: 10:1 savings             │
  │                                                         │
  │  Cost Estimate (100 × 1 TB volumes):                    │
  │  • Snapshots (local): ~10 TB (~$500/month SSD)          │
  │  • Daily incremental backup: ~5 TB/month to S3          │
  │  • S3 storage: ~$500/month                              │
  │  • Total protection cost: ~$1K/month for 100 TB         │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How does WAFL create zero-cost snapshots?** — ROW: never overwrites; old blocks are the snapshot. Snapshot creation = freeze root pointer
2. **COW vs ROW trade-offs?** — COW penalises writes (extra read+write); ROW penalises reads (pointer chasing). ROW wins for write-heavy storage
3. **How deep can a snapshot chain go?** — NetApp supports 1023 snapshots per volume; deeper chains increase metadata overhead
4. **How to snapshot a running database consistently?** — Quiesce I/O, flush buffers, create snapshot, thaw. Use VSS or app-specific plugins
5. **How do you estimate snapshot space?** — Track change rate (dirty blocks per interval), multiply by retention depth
