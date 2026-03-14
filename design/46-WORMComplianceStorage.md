# Design a WORM Compliance Storage System

## Overview
Design an immutable, tamper-proof storage system like NetApp SnapLock / AWS S3 Object Lock — enforcing Write Once, Read Many (WORM) semantics for regulatory compliance (SEC 17a-4, FINRA, HIPAA, GDPR). Once committed, data cannot be modified, deleted, or shortened in retention — not even by administrators.

## 1. Requirements

**Functional:**
- WORM mode: files become immutable after commit; no modification or deletion
- Retention periods: per-file or per-volume retention locks (clock-based)
- Compliance clock: tamper-proof clock for retention (not dependent on system clock)
- Litigation hold: suspend retention expiry during legal hold
- Event-based retention: start retention clock on an external event (e.g., employee departure)
- Privileged delete (Enterprise mode): admin can delete before expiry (audit logged)
- Compliance mode: NO ONE can delete — not admin, not vendor, not support

**Non-Functional:**
- Tamper-proof: even root/admin cannot circumvent in compliance mode
- Audit trail: immutable log of all operations
- Clock accuracy: compliance clock drift < 1 second/day
- Standards: SEC 17a-4(f), CFTC 1.31, FINRA 4511, MiFID II, HIPAA

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **WORM data** | 200TB (financial records, healthcare, legal) |
| **Object count** | 500 million files / objects |
| **New WORM data/day** | 500GB |
| **Retention range** | 1 year to 99 years |
| **Active legal holds** | 50 litigation holds |
| **Audit log size** | 1TB/year |

## 3. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│              WORM Compliance Storage System                     │
│                                                                │
│  Client Applications                                           │
│  ┌──────────┐ ┌────────────┐ ┌─────────────┐                 │
│  │ Email    │ │ Trading    │ │ Medical     │                 │
│  │ Archive  │ │ Records    │ │ Records     │                 │
│  └────┬─────┘ └─────┬──────┘ └──────┬──────┘                 │
│       │              │               │                         │
│       └──────────────┼───────────────┘                         │
│                      ▼                                         │
│  ┌──────────────────────────────────────┐                      │
│  │     WORM Enforcement Layer           │                      │
│  │                                      │                      │
│  │  ┌────────────┐  ┌───────────────┐  │                      │
│  │  │ Compliance │  │ Retention     │  │                      │
│  │  │ Clock      │  │ Manager       │  │                      │
│  │  │ (tamper-   │  │ (per-file     │  │                      │
│  │  │  proof)    │  │  expiry)      │  │                      │
│  │  └────────────┘  └───────────────┘  │                      │
│  │                                      │                      │
│  │  ┌────────────┐  ┌───────────────┐  │                      │
│  │  │ Legal Hold │  │ Privileged    │  │                      │
│  │  │ Manager    │  │ Delete (Ent.) │  │                      │
│  │  └────────────┘  └───────────────┘  │                      │
│  │                                      │                      │
│  │  ┌────────────────────────────────┐ │                      │
│  │  │ Audit Log (append-only, WORM) │ │                      │
│  │  └────────────────────────────────┘ │                      │
│  └──────────────────────────────────────┘                      │
│                      │                                         │
│                      ▼                                         │
│  ┌──────────────────────────────────────┐                      │
│  │     Storage Layer                    │                      │
│  │                                      │                      │
│  │  SnapLock Volume / S3-WORM Bucket    │                      │
│  │  ┌──────┐ ┌──────┐ ┌──────┐        │                      │
│  │  │ File │ │ File │ │ File │ ...    │                      │
│  │  │ WORM │ │ WORM │ │ WORM │        │                      │
│  │  │ attr:│ │ attr:│ │ attr:│        │                      │
│  │  │ ret- │ │ ret- │ │ ret- │        │                      │
│  │  │ until│ │ until│ │ until│        │                      │
│  │  │ 2030 │ │ 2035 │ │ 2050 │        │                      │
│  │  └──────┘ └──────┘ └──────┘        │                      │
│  │                                      │                      │
│  │  Immutable: no modify/delete/shorten │                      │
│  │  until retention-time expires        │                      │
│  └──────────────────────────────────────┘                      │
└────────────────────────────────────────────────────────────────┘
```

## 4. WORM Commit & Retention Model

```
File Lifecycle in WORM Volume:

  Created ──► Appendable ──► Committed (WORM) ──► Expired ──► Deletable
    │              │               │                  │            │
    │  Can write   │  File exists  │  Immutable:      │  Retention │
    │  content     │  but not yet  │  - No modify     │  expired;  │
    │              │  committed    │  - No delete     │  can be    │
    │              │               │  - No shorten    │  deleted   │
    │              │               │  - No rename     │            │
    │              │               │  until ret_time  │            │
    └──────────────┘               └──────────────────┘            │
                                                                    │
  Commit trigger:                                                   │
    Option A: autocommit_period (e.g., 4 hours after last write)   │
    Option B: explicit commit (set read-only + retention-time)     │
                                                                    │
  Retention period:                                                │
    min_retention: set by admin (e.g., 7 years) ← cannot shorten  │
    max_retention: set by admin (e.g., 99 years) ← cannot exceed  │
    file_retention: set per file (within min/max range)            │
    ↑ Can be EXTENDED but NEVER shortened                          │
                                                                    │
  Legal Hold:                                                      │
    hold_id: "litigation-2024-001"                                 │
    effect: retention_time = max(original, INFINITE) while held    │
    even if retention expires, file CANNOT be deleted under hold   │

SnapLock Volume Types:
  
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Compliance Mode:                                       │
  │    - NO ONE can delete before retention expiry          │
  │    - Volume cannot be destroyed if contains WORM files  │
  │    - Admin cannot override                              │
  │    - NetApp support cannot override                     │
  │    - Even "format disk" prevented by firmware           │
  │    → SEC 17a-4(f) compliant                            │
  │                                                         │
  │  Enterprise Mode:                                       │
  │    - "Privileged delete" available to compliance admin  │
  │    - Deletion logged in tamper-proof audit log          │
  │    - For internal governance (not SEC-regulated)        │
  │    - All other WORM guarantees apply                    │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

## 5. Compliance Clock

```
Tamper-Proof Clock (ComplianceClock):

  Problem: system clock can be changed by admin → circumvent retention
  Solution: separate compliance clock that cannot be set backward

  ┌──────────────────────────────────────────────────────┐
  │ ComplianceClock:                                     │
  │                                                      │
  │  Initialization:                                     │
  │    - Set ONCE during volume creation                 │
  │    - Initialized from system clock                   │
  │    - Then runs independently                         │
  │                                                      │
  │  Incrementing:                                       │
  │    - Internal NTP-like sync                          │
  │    - Maximum skew: ±1 second/day                     │
  │    - Can ONLY move forward                           │
  │    - Admin cannot set it backward                    │
  │    - Stored in non-volatile, tamper-evident storage  │
  │                                                      │
  │  System Clock Changed?                               │
  │    - ComplianceClock continues independently         │
  │    - Retention decisions use ComplianceClock only    │
  │    - Alert raised if drift > threshold               │
  │                                                      │
  │  Verification:                                       │
  │    compliance_clock_value → compared to external NTP │
  │    maximum_allowed_skew → 24 hours                   │
  │    if skew > 24h → alert (possible tampering)        │
  └──────────────────────────────────────────────────────┘
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Create SnapLock Volume
POST /api/v1/storage/volumes
{
  "name": "vol_compliance_email",
  "type": "snaplock",
  "snaplock_type": "compliance",     // compliance | enterprise
  "min_retention": "P7Y",            // ISO 8601 duration: 7 years
  "max_retention": "P30Y",           // 30 years max
  "default_retention": "P7Y",
  "autocommit_period": "PT4H",       // 4 hours after last write
  "volume_append_mode": false
}

# Commit File to WORM
PUT /api/v1/storage/volumes/{vol}/files/{path}/worm
{
  "retention_time": "2031-12-31T23:59:59Z",  // absolute time
  "litigation_hold": false
}
→ {
  "file": "/vol_compliance_email/inbox/msg-2024-001.eml",
  "state": "committed",
  "retention_time": "2031-12-31T23:59:59Z",
  "committed_at": "2024-07-15T10:30:00Z",
  "compliance_clock_at_commit": "2024-07-15T10:30:01Z"
}

# Extend Retention (always allowed)
PUT /api/v1/storage/volumes/{vol}/files/{path}/worm
{
  "retention_time": "2040-12-31T23:59:59Z"   // extend 9 more years
}

# Shorten Retention (REJECTED)
PUT /api/v1/storage/volumes/{vol}/files/{path}/worm
{
  "retention_time": "2028-12-31T23:59:59Z"   // shorter than current
}
→ 403 Forbidden: "Cannot shorten retention period on compliance volume"

# Legal Hold
POST /api/v1/storage/volumes/{vol}/legal-holds
{
  "hold_id": "litigation-2024-SEC-investigation",
  "description": "SEC investigation hold on all 2023 trading records",
  "scope": {
    "path_prefix": "/vol_compliance_email/trading/2023/",
    "pattern": "*.eml"
  }
}

# Privileged Delete (Enterprise mode only)
DELETE /api/v1/storage/volumes/{vol}/files/{path}?privileged=true
{
  "reason": "court-ordered destruction order #12345",
  "authorized_by": "compliance-officer@corp.com"
}
→ {
  "result": "deleted",
  "audit_record": "audit-2024-001234",
  "irreversible": true
}

# S3 Object Lock API (S3-compatible)
PUT /{bucket}/{key}?object-lock
x-amz-object-lock-mode: COMPLIANCE
x-amz-object-lock-retain-until-date: 2031-12-31T23:59:59Z
```

### Data Model

```
WORM File Metadata (stored with inode):

struct WORMAttributes {
  worm_state:        enum { APPENDABLE, COMMITTED, EXPIRED }
  retention_time:    timestamp    // compliance clock timestamp
  committed_at:      timestamp    // when file was committed  
  hold_count:        uint32       // number of active legal holds
  hold_ids:          []string     // litigation hold identifiers
  snaplock_type:     enum { COMPLIANCE, ENTERPRISE }
  autocommit_eligible: bool       // if autocommit timer running
  autocommit_at:     timestamp    // when autocommit will trigger
  fingerprint:       sha256_hash  // content hash at commit time
}

Audit Log Entry:
struct AuditRecord {
  id:            uuid
  timestamp:     timestamp       // compliance clock time
  operation:     enum { COMMIT, HOLD_SET, HOLD_RELEASE, EXTEND,
                        PRIV_DELETE, EXPIRE, ACCESS }
  file_path:     string
  user:          string
  source_ip:     string
  result:        enum { SUCCESS, DENIED }
  reason:        string          // for denied operations
  retention_old: timestamp
  retention_new: timestamp
}

// Audit log itself is WORM — stored in SnapLock compliance volume
// Cannot be modified or deleted
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **File count** | B-tree index over WORM metadata | Billions of files |
| **Volume capacity** | FlexVol up to 300TB; FlexGroup for larger | Petabytes |
| **Retention tracking** | Indexed by expiry date for efficient GC | O(log N) lookups |
| **Legal holds** | Hold ID index; O(1) hold check per file access | 1000s of concurrent holds |
| **Audit logs** | Time-partitioned WORM volumes; archived to cold tier | Decades of history |

## 8. No Data Loss

| Scenario | Protection |
|----------|------------|
| **Admin tries to delete** | Firmware/OS-level enforcement; delete syscall returns EPERM |
| **Disk failure** | RAID-DP + SyncMirror; data always recoverable |
| **Controller failure** | HA failover; WORM metadata in NVRAM + on-disk |
| **Retention clock tampering** | ComplianceClock independent of system clock; hardware-protected |
| **Volume destroy attempt** | Cannot destroy volume with unexpired WORM files |
| **Snapcopy/replication** | SnapMirror replicates WORM attributes; DR copy also WORM-protected |

## 9. Latency

| Operation | Latency | Notes |
|-----------|---------|-------|
| **Write (before commit)** | Normal write latency (~200μs) | No overhead pre-commit |
| **WORM commit** | <1ms (metadata update) | Set inode flags + retention time |
| **Read WORM file** | Normal read latency (~100μs) | No overhead for reads |
| **Retention check** | O(1) per access | Checked at VFS layer |
| **Legal hold apply** | O(N) for N matched files | Background job; metadata update |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Storage HW failure** | Data unavailable temporarily | HA pair + RAID-DP; no data loss |
| **Compliance clock battery** | Clock cannot advance | Redundant clock; alert + replace |
| **Software bug in WORM enforcement** | Could allow modification | Codeskey FIPS validation; firmware-level enforcement |
| **Audit log corruption** | Loss of compliance evidence | Replicated + checksummed; separate volume |
| **Firmware downgrade attack** | Could bypass WORM | Secure boot; firmware signing; no downgrade capability |

## 11. Availability

**Target: 99.99% (< 53 minutes downtime per year)**

WORM data is always available through:
- HA pair active/passive failover (<60s)
- MetroCluster for site-level DR
- SnapMirror DR copies retain WORM attributes
- No maintenance window breaks WORM guarantees

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Tamper prevention** | Firmware-enforced WORM; secure boot chain |
| **Admin protection** | Even cluster admin cannot bypass compliance mode |
| **Encryption** | NVE/NAE at rest; TLS in transit; KMIP for key management |
| **Access control** | RBAC: compliance administrator role for policy management |
| **Audit immutability** | Audit logs stored in separate SnapLock compliance volume |
| **Digital signatures** | Optional content fingerprinting at commit (SHA-256) |
| **Multi-admin verification** | Require 2+ admin approval for policy changes |

## 13. Cost Constraints

**Estimated Cost (200TB, 7-year retention):**

| Component | Cost |
|-----------|------|
| **SnapLock storage (200TB)** | $600,000 (all-flash) or $200,000 (hybrid) |
| **SnapLock license** | $50,000/year |
| **Annual audit/compliance** | $20,000/year (Cohasset validation) |
| **Replication (DR copy)** | $200,000 (additional storage) |
| **Total (Year 1)** | **~$870,000** |
| **Cost per TB/year** | **~$620** |

S3 Object Lock (cloud alternative): ~$5/TB/month × 200TB = $12,000/month = $144,000/year — cheaper but check regulatory acceptance of cloud-only storage.

## Key Interview Discussion Points

1. **Compliance vs Enterprise mode?** — Compliance: nobody can delete (SEC 17a-4 requirement). Enterprise: privileged admin can delete with audit trail (for internal governance). Choice depends on regulatory requirement
2. **How is the compliance clock tamper-proof?** — Hardware-backed monotonic counter independent of system clock; battery-backed NVRAM; can only move forward; initialized once at volume creation. Even setting system clock backward doesn't affect retention
3. **What happens when retention expires?** — File becomes deletable but is NOT auto-deleted. Admin must explicitly remove. This prevents accidental data loss from misconfigured retention
4. **Legal hold vs retention?** — Legal hold is separate: even if retention expires, file under litigation hold cannot be deleted. Multiple holds can apply to same file. Hold count tracked in metadata
5. **S3 Object Lock compatibility?** — S3 Object Lock provides Governance mode (similar to Enterprise) and Compliance mode. ONTAP S3 supports both, mapping to SnapLock semantics. Useful for cloud-native WORM workflows
