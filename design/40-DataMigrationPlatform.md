# Design a Data Migration Platform

## Overview
Design an enterprise data migration platform like NetApp XCP / AWS DataSync — enabling petabyte-scale migrations between heterogeneous storage systems with minimal downtime, data integrity verification, and automated cutover orchestration.

## 1. Requirements

**Functional:**
- File-level migration (NFS, SMB/CIFS) between any storage vendors
- Block-level migration (SAN LUN) with online capability
- Incremental sync: initial baseline + continuous incremental updates
- Cutover orchestration: automated final sync + switchover
- Data verification: checksum comparison source vs destination
- Filtering: include/exclude by path, size, date, type

**Non-Functional:**
- Throughput: 10+ Gbps sustained (100TB/day)
- Cutover downtime: <30 minutes for 100TB dataset
- Zero data loss during migration
- Support for billions of files (small file optimization)
- Cross-protocol: NFS→SMB, CIFS→S3, etc.

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Migration volume** | 2PB total |
| **File count** | 5 billion files |
| **Avg file size** | 400KB (highly skewed: many small, few large) |
| **Available bandwidth** | 100Gbps network |
| **Sustained throughput** | 10 GB/s (80 Gbps effective) |
| **Migration duration** | ~55 hours for baseline |
| **Daily change rate** | 0.5% = 10TB |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│              Data Migration Platform                              │
│                                                                   │
│  Source Storage          Migration Engine         Dest Storage     │
│  ┌──────────────┐     ┌─────────────────┐     ┌──────────────┐  │
│  │ NetApp FAS   │     │                 │     │ Cloud (S3)   │  │
│  │ (NFS/CIFS)   │◄───►│  ┌───────────┐  │◄───►│ or           │  │
│  │              │     │  │ Scanner   │  │     │ ONTAP CVO    │  │
│  │ or           │     │  │ (catalog) │  │     │ or           │  │
│  │ EMC Isilon   │     │  └─────┬─────┘  │     │ Pure Storage │  │
│  │ or           │     │        │         │     │              │  │
│  │ Windows FS   │     │  ┌─────▼─────┐  │     │              │  │
│  │ or           │     │  │ Scheduler │  │     │              │  │
│  │ S3 bucket    │     │  │ (plan)    │  │     │              │  │
│  └──────────────┘     │  └─────┬─────┘  │     └──────────────┘  │
│                       │        │         │                       │
│                       │  ┌─────▼─────┐  │                       │
│                       │  │ Workers   │  │                       │
│                       │  │ (parallel │  │                       │
│                       │  │  copy)    │  │                       │
│                       │  │           │  │                       │
│                       │  │ W1 W2 ... │  │                       │
│                       │  │ W3 W4 Wn  │  │                       │
│                       │  └─────┬─────┘  │                       │
│                       │        │         │                       │
│                       │  ┌─────▼─────┐  │                       │
│                       │  │ Verifier  │  │                       │
│                       │  │ (checksum)│  │                       │
│                       │  └───────────┘  │                       │
│                       └─────────────────┘                       │
│                                                                   │
│  Migration Lifecycle:                                            │
│  ┌─────┐  ┌────────┐  ┌───────┐  ┌────────┐  ┌──────┐         │
│  │Scan │→ │Baseline│→ │Increm.│→ │Cutover │→ │Verify│         │
│  │     │  │Copy    │  │Sync   │  │(final) │  │      │         │
│  └─────┘  └────────┘  └───────┘  └────────┘  └──────┘         │
│  1 hour   55 hours     ongoing    30 min      2 hours           │
└───────────────────────────────────────────────────────────────────┘
```

## 4. Scanner & Catalog

```
File System Scanner:

Phase 1: Build Source Catalog
  ┌──────────────────────────────────────────────────────┐
  │ Parallel Directory Walker                            │
  │                                                      │
  │ Strategy: breadth-first, N parallel threads          │
  │   Thread 1: scan /data/dir_a/**                     │
  │   Thread 2: scan /data/dir_b/**                     │
  │   Thread N: scan /data/dir_z/**                     │
  │                                                      │
  │ Each entry recorded in catalog DB:                   │
  │   path: /data/dir_a/file1.dat                       │
  │   size: 45,234,567 bytes                            │
  │   mtime: 2024-01-15T08:30:00Z                       │
  │   checksum: xxhash64:0xABCDEF01                     │
  │   permissions: 0644, uid:1000, gid:1000             │
  │   acl: [user:bob:rwx, group:eng:r-x]               │
  │   xattrs: {security.selinux: ...}                   │
  │                                                      │
  │ Small File Optimization:                             │
  │   Files < 64KB → batch into single copy operation   │
  │   Reduce per-file overhead (open/close/metadata)    │
  │   Bundle 1000 small files per batch                 │
  │                                                      │
  │ Performance: 50,000 files/sec per scanner thread    │
  │ 5B files / 50K/s / 16 threads ≈ ~1.7 hours         │
  └──────────────────────────────────────────────────────┘

Change Detection (for incremental sync):
  Method 1: mtime comparison (fast, not 100% reliable)
  Method 2: Changelog/audit log from source (fastest, vendor-specific)
  Method 3: Full re-scan + diff (slow but universal)
  
  Preferred: vendor-specific changelog if available (e.g., ONTAP FPolicy)
```

## 5. Parallel Copy Engine

```
Worker Pool Design:

  ┌──────────────────────────────────────────────────────┐
  │ Copy Scheduler                                       │
  │                                                      │
  │ Work Queue (priority queue):                         │
  │   Priority 1: Large files (>1GB) → single worker    │
  │   Priority 2: Medium files → worker per batch        │
  │   Priority 3: Small files (<64KB) → bundled batch   │
  │                                                      │
  │ Large File Copy (parallel segments):                 │
  │   file_large.dat (100GB):                            │
  │     Segment 1: bytes 0-25GB      → Worker 1          │
  │     Segment 2: bytes 25-50GB     → Worker 2          │
  │     Segment 3: bytes 50-75GB     → Worker 3          │
  │     Segment 4: bytes 75-100GB    → Worker 4          │
  │     Reassemble at destination                        │
  │                                                      │
  │ Small File Batch Copy:                               │
  │   batch_001: [file1, file2, ..., file1000]          │
  │     → Single worker, single TCP connection           │
  │     → Tar-stream or pNFS parallel                    │
  │                                                      │
  │ Throughput Optimization:                             │
  │   - 64 workers × 160 MB/s each = 10 GB/s           │
  │   - TCP window tuning: 4MB send/recv buffers        │
  │   - Multi-connection per source/dest pair            │
  │   - Zero-copy sendfile() where supported             │
  └──────────────────────────────────────────────────────┘

Checkpoint & Resume:
  Every 10,000 files or 60 seconds:
    Save checkpoint: {
      last_completed_path: "/data/dir_m/file_5234",
      files_copied: 2,340,000,
      bytes_copied: 943,234,567,890,
      timestamp: "2024-01-15T14:30:00Z"
    }
  
  On failure: resume from last checkpoint
  On restart: skip already-copied files (verified by size+mtime)
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Create Migration Job
POST /api/v1/migrations
{
  "name": "dc-migration-2024",
  "source": {
    "type": "nfs",
    "server": "old-filer.corp.com",
    "export": "/data/prod",
    "credentials": "vault:///migration/source"
  },
  "destination": {
    "type": "s3",
    "bucket": "company-data-lake",
    "prefix": "migrated/prod/",
    "region": "us-east-1"
  },
  "options": {
    "parallelism": 64,
    "preserve_permissions": true,
    "preserve_timestamps": true,
    "verify_checksums": true,
    "exclude_patterns": ["*.tmp", "*.log", ".snapshot/**"],
    "bandwidth_limit_mbps": 8000
  }
}

# Start Baseline Copy
POST /api/v1/migrations/{job_id}/baseline
→ { "status": "running", "estimated_hours": 55 }

# Start Incremental Sync
POST /api/v1/migrations/{job_id}/sync
{
  "mode": "continuous",       // continuous | once
  "interval_minutes": 15
}

# Execute Cutover
POST /api/v1/migrations/{job_id}/cutover
{
  "pre_cutover_script": "scripts/quiesce_apps.sh",
  "post_cutover_script": "scripts/update_mount.sh",
  "max_downtime_minutes": 30,
  "rollback_on_failure": true
}

# Verify Migration
POST /api/v1/migrations/{job_id}/verify
{
  "mode": "full_checksum",     // full_checksum | metadata_only | sample
  "sample_percentage": 100     // 100 = full verify
}
→ {
  "total_files": 5000000000,
  "verified": 5000000000,
  "mismatches": 0,
  "missing_at_dest": 0,
  "extra_at_dest": 0,
  "status": "PASS"
}
```

### Cutover Orchestration

```
Cutover Sequence (automated):

  T=0:  Pre-cutover checks
        - Verify last sync completed < 15 min ago
        - Estimate final sync time
        - Confirm cutover window approved
        
  T+1:  Quiesce applications
        - Run pre_cutover_script (stop writes to source)
        - Verify I/O quiesced (no new writes)
        
  T+2:  Final incremental sync
        - Copy all changes since last sync
        - Duration: ~5 minutes for 10TB daily change rate
        
  T+7:  Verify final sync
        - Quick verification: file count + total size match
        - Spot-check: random 1% checksum verify
        
  T+10: Switch access
        - Update DNS/mount points to destination
        - For NFS: update /etc/fstab or automount
        - For S3: update application endpoints
        - For SAN: remap LUNs to new target
        
  T+15: Validate applications
        - Run post_cutover_script (start apps on new storage)
        - Smoke test: read/write verification
        
  T+20: Cutover complete
        - Mark migration as completed
        - Source enters read-only quarantine (30 days)
        
  Rollback (if T+15 validation fails):
        - Revert DNS/mounts to source
        - Source was read-only, data intact
        - Investigate and retry
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Data volume** | Parallel workers + segmented large files | 10 GB/s (100TB/day) |
| **File count** | Parallel scanners + small file batching | 5 billion+ files |
| **Sources/Destinations** | Multiple migration jobs in parallel | 50 concurrent jobs |
| **Workers** | Scale out worker nodes horizontally | 256 workers per job |
| **Cross-protocol** | Adapters: NFS↔SMB↔S3↔HDFS | Any-to-any migration |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Worker crash mid-copy** | Checkpoint/resume; re-copy from last checkpoint; no partial files at dest |
| **Network interruption** | Retry with exponential backoff; resume from byte offset for large files |
| **Source modified during copy** | Incremental sync catches changes; final sync during quiesce ensures consistency |
| **Destination corruption** | Post-migration checksum verification; full or sampled |
| **Permission loss** | Preserve ACLs, xattrs, timestamps; cross-protocol mapping (POSIX↔NTFS ACLs) |

## 9. Latency

| Phase | Duration (2PB, 5B files) |
|-------|--------------------------|
| **Catalog scan** | ~2 hours |
| **Baseline copy** | ~55 hours (10 GB/s) |
| **Incremental sync** | ~30 min per cycle (10TB changes) |
| **Final cutover sync** | 5-15 minutes |
| **Verification** | 2-4 hours (full checksum) |
| **Total cutover downtime** | **<30 minutes** |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Worker failure** | Batch of files not copied | Checkpoint recovery; reassign work to healthy worker |
| **Source unavailable** | Migration paused | Retry with backoff; alert; resume when source returns |
| **Destination full** | Writes fail | Pre-check destination capacity; alert at 80% of estimated need |
| **Metadata mismatch** | Files copied but permissions wrong | Cross-protocol ACL mapping tables; verified in post-migration check |

## 11. Availability

**Target: 99.9% for migration platform; zero downtime for source during baseline**

```
Availability Design:
  - Migration engine: stateless workers + persistent queue → survive worker loss
  - Catalog DB: replicated PostgreSQL → survive DB failure
  - Checkpoints: stored in durable storage → resume after any failure
  - Source impact: read-only access during migration; no impact on production
  - Rolling worker updates: drain and replace workers without job interruption
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Credentials** | HashiCorp Vault for source/dest credentials; never in config files |
| **Data in transit** | TLS 1.3 for cloud destinations; Kerberos for NFS/SMB |
| **Data at rest** | Workers don't persist data; only metadata in catalog (encrypted) |
| **Access control** | RBAC: migration admin, operator, viewer roles |
| **Audit trail** | Every file copy logged; migration job history with timestamps |
| **Compliance** | Chain of custody: source path → dest path mapping for legal holds |

## 13. Cost Constraints

**Estimated Cost (2PB migration):**

| Component | Cost |
|-----------|------|
| **Migration engine** | 16 worker nodes (32-core, 128GB RAM, 100GbE): $80,000 (lease 3 months) |
| **Network bandwidth** | 100Gbps dedicated link (if cross-DC): $15,000/month × 3 = $45,000 |
| **Cloud egress** | If source is cloud: 2PB × $0.05/GB = $100,000 (!!) |
| **Software license** | Migration tool: $20,000 |
| **Labor** | 2 engineers × 3 months: $90,000 |
| **Total (on-prem→on-prem)** | **~$235,000** |
| **Total (cloud→on-prem)** | **~$335,000** (cloud egress is expensive) |

**Alternative:** For cloud-to-cloud, use provider's internal transfer (free/cheap within same region).

## Key Interview Discussion Points

1. **How to handle billions of small files?** — Batch into groups; use tar-stream or multi-file parallel copy; reduce per-file overhead (open/close/stat). Scanner is the bottleneck, not bandwidth
2. **Cutover downtime minimization?** — Continuous incremental sync reduces final delta to minutes. Pre-stage everything; only final sync + mount switch during outage window
3. **Cross-protocol ACL mapping?** — POSIX mode bits → NTFS ACLs: lossy mapping. Maintain mapping table; verify with test users. Consider running both protocols during transition
4. **How to verify 5 billion files?** — Full checksum: 2-4 hours (hash at 10GB/s). Sampled: 1% random, verify size+mtime for rest. Trade-off between certainty and time
5. **Cloud egress cost trap?** — 2PB egress from AWS/Azure/GCP = $50-100K. Mitigation: use provider transfer services (AWS Snowball: $10K for petabytes), or schedule during promotional periods
