# Design a Data Deduplication and Compression System

## Overview
Design an inline and post-process deduplication and compression engine for enterprise storage — reducing physical capacity by 3-5× through fingerprint-based duplicate detection, variable-length chunking, and adaptive compression, similar to NetApp ONTAP storage efficiency.

## 1. Requirements

**Functional:**
- Inline dedup: detect and eliminate duplicates at write time
- Post-process dedup: background dedup for existing data
- Inline compression: LZ4 (fast) or ZSTD (high ratio) at block level
- Cross-volume dedup: within same aggregate
- Data compaction: pack small blocks into full disk blocks
- Space savings reporting and estimation

**Non-Functional:**
- Inline dedup latency overhead: <5% write latency impact
- Compression throughput: line-rate (no bottleneck on write path)
- Dedup ratio: 2-3× for VDI, 1.5-2× for databases
- Compression ratio: 1.5-2.5× depending on data type
- CPU overhead: < 15% additional

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Logical capacity** | 500TB across 100 volumes |
| **Physical capacity** | 150TB after dedup+compression (3.3× savings) |
| **Write throughput** | 5 GB/s sustained |
| **Block size** | 4KB (dedup granularity) |
| **Fingerprint table** | 125 billion entries (500TB / 4KB) |
| **Fingerprint table size** | ~2.5TB (20 bytes per entry) |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│           Deduplication & Compression Pipeline                 │
│                                                               │
│  Write I/O                                                    │
│     │                                                         │
│     ▼                                                         │
│  ┌──────────────────────────────────────────────┐             │
│  │  1. Receive Write Block (4KB-64KB)           │             │
│  └──────────┬───────────────────────────────────┘             │
│             │                                                 │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────┐             │
│  │  2. Compute Fingerprint (SHA-256 or xxHash)  │             │
│  │     fingerprint = hash(block_data)           │             │
│  └──────────┬───────────────────────────────────┘             │
│             │                                                 │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────┐             │
│  │  3. Lookup in Fingerprint Table              │             │
│  │     ┌─────────────────────────────────┐      │             │
│  │     │ Hash │ Phys Block │ Ref Count   │      │             │
│  │     │──────┼────────────┼─────────────│      │             │
│  │     │ 0xA1 │ blk_50000  │ 3           │      │             │
│  │     │ 0xB2 │ blk_50001  │ 1           │      │             │
│  │     │ 0xC3 │ blk_50002  │ 7           │      │             │
│  │     └─────────────────────────────────┘      │             │
│  └──────┬──────────────┬────────────────────────┘             │
│         │ MISS         │ HIT                                  │
│         ▼              ▼                                      │
│  ┌──────────────┐  ┌──────────────────────────┐              │
│  │ 4a. Compress │  │ 4b. Deduplicate:         │              │
│  │    (LZ4/ZSTD)│  │   Point to existing block│              │
│  │    Write new │  │   Increment ref count    │              │
│  │    phys block│  │   No physical write!     │              │
│  └──────┬───────┘  └──────────┬───────────────┘              │
│         │                     │                               │
│         ▼                     ▼                               │
│  ┌──────────────────────────────────────────────┐             │
│  │  5. Data Compaction                          │             │
│  │     Pack compressed blocks (< 4KB) into      │             │
│  │     single physical 4KB blocks               │             │
│  │     ┌──────────────────────────────────┐     │             │
│  │     │ Phys 4KB Block:                  │     │             │
│  │     │ [cBlkA 1.2KB][cBlkB 0.8KB]      │     │             │
│  │     │ [cBlkC 1.5KB][pad 0.5KB]        │     │             │
│  │     └──────────────────────────────────┘     │             │
│  └──────────────────────────────────────────────┘             │
│                                                               │
│  Read Path (reverse):                                        │
│     Logical block → Block map → Physical block               │
│     → Decompress if compressed                               │
│     → Return original data                                   │
└───────────────────────────────────────────────────────────────┘
```

## 4. Fingerprint Table Design

```
Fingerprint Table Architecture:

On-Disk Layout (persistent):
┌──────────────────────────────────────────────────┐
│  Fingerprint Store (per aggregate)               │
│                                                  │
│  Structure: B+ tree on SSD                       │
│  Key: fingerprint (32 bytes - SHA-256 truncated) │
│  Value: {physical_block, ref_count, flags}       │
│                                                  │
│  Size per entry: 20 bytes                        │
│  Total entries: 125 billion (500TB / 4KB)        │
│  Total size: ~2.5TB (stored on SSD)              │
└──────────────────────────────────────────────────┘

In-Memory Cache (hot entries):
┌──────────────────────────────────────────────────┐
│  Bloom Filter (first-pass check):                │
│    Size: 16GB RAM                                │
│    False positive rate: 1%                       │
│    → 99% of unique blocks skip disk lookup       │
│                                                  │
│  Hot Fingerprint Cache:                          │
│    LRU cache: 64GB RAM                           │
│    Hit rate: ~80% for typical workloads          │
│    Entry: {fingerprint, phys_block, ref_count}   │
└──────────────────────────────────────────────────┘

Lookup Flow:
  1. Check Bloom filter (in RAM) → "definitely not" or "maybe"
  2. If "maybe": check LRU cache (in RAM)
  3. If cache miss: read B+ tree from SSD (~200μs)
  4. If found: dedup (increment ref, point to existing block)
  5. If not found: new block → compress → write → insert fingerprint
```

## 5. Compression Algorithms

```
Adaptive Compression Selection:

┌────────────────────────────────────────────────────┐
│  Algorithm      │ Speed    │ Ratio │ Use Case      │
│─────────────────┼──────────┼───────┼───────────────│
│  LZ4 (default)  │ 3 GB/s   │ 2:1   │ General, perf │
│  ZSTD-1         │ 1.5 GB/s │ 2.5:1 │ Balanced      │
│  ZSTD-3         │ 800 MB/s │ 3:1   │ Cold data     │
│  None           │ Wire     │ 1:1   │ Pre-compressed│
└────────────────────────────────────────────────────┘

Adaptive Selection Logic:
  class CompressionSelector:
    def select(self, block):
      # Detect if data is already compressed/encrypted
      entropy = shannon_entropy(block[:256])
      if entropy > 7.5:  # 8.0 = random
        return NONE  # already compressed/encrypted
      
      # Check volume QoS: if latency-sensitive, use LZ4
      if block.volume.qos_policy.latency_sensitive:
        return LZ4
      
      # Check if volume is being tiered to cold
      if block.temperature == COLD:
        return ZSTD_3  # maximize compression
      
      return LZ4  # default: speed over ratio

Compression Groups (8 blocks):
  Instead of compressing individual 4KB blocks:
  - Group 8 consecutive 4KB blocks → 32KB unit
  - Compress as single unit (better ratio, cross-block patterns)
  - Store: compressed size ≤ 7 blocks? Save remainder
  
  Example:
    8 × 4KB blocks (32KB logical)
    Compressed to 12KB (2.67× ratio)
    Stored in 3 physical 4KB blocks + metadata
    Savings: 5 × 4KB = 20KB per group
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Enable Volume Dedup
PATCH /api/v1/volumes/{vol_uuid}/efficiency
{
  "dedup": {
    "enabled": true,
    "mode": "inline",            // inline | background | both
    "cross_volume": false
  },
  "compression": {
    "enabled": true,
    "algorithm": "lz4",          // lz4 | zstd
    "inline": true
  },
  "compaction": {
    "enabled": true              // pack small blocks
  }
}

# Get Space Savings Report
GET /api/v1/volumes/{vol_uuid}/efficiency/stats
→ {
  "logical_size_gb": 1024,
  "physical_size_gb": 320,
  "total_ratio": "3.2:1",
  "dedup_savings_gb": 450,
  "dedup_ratio": "1.8:1",
  "compression_savings_gb": 254,
  "compression_ratio": "1.8:1",
  "compaction_savings_gb": 12
}

# Run Background Dedup Job
POST /api/v1/volumes/{vol_uuid}/efficiency/start
{
  "scan_type": "full",           // full | incremental
  "max_duration_hours": 4,
  "priority": "low"
}
```

### Reference Counting & Garbage Collection

```
Reference Count Management:

Block Lifecycle:
  Write new (unique) block:
    → allocate physical block
    → set ref_count = 1
    → insert fingerprint in table
    
  Dedup match:
    → increment ref_count
    → map logical block to existing physical block
    → NO new physical allocation
    
  Delete/overwrite block:
    → decrement ref_count
    → if ref_count == 0:
        → free physical block
        → remove fingerprint entry
        → update space accounting
    → if ref_count > 0:
        → block still referenced by other logical blocks
        → only remove this mapping

Garbage Collection (background):
  class DedupGarbageCollector:
    def run(self):
      for entry in fingerprint_table.scan():
        actual_refs = count_block_references(entry.phys_block)
        if actual_refs == 0:
          # Orphaned: no logical blocks reference this
          free_block(entry.phys_block)
          fingerprint_table.remove(entry.fingerprint)
        elif actual_refs != entry.ref_count:
          # Inconsistency: correct ref count
          entry.ref_count = actual_refs
          fingerprint_table.update(entry)
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Data volume** | Fingerprint table on SSD; Bloom filter in RAM | 500TB+ per aggregate |
| **Write throughput** | Hardware-accelerated hashing; parallel compression | 5 GB/s inline |
| **Fingerprint lookups** | Bloom filter eliminates 99% of misses; LRU cache for hits | 1M lookups/s |
| **Cross-volume dedup** | Aggregate-wide fingerprint table shared across volumes | 200 volumes per aggregate |
| **Background dedup** | Low-priority scanner; rate-limited to avoid perf impact | Process 10TB/day |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Ref count corruption** | Periodic GC verifies actual references vs stored count; WAFL checksums on metadata |
| **Hash collision** | SHA-256: collision probability negligible (<2^-128); byte-compare optional for paranoid mode |
| **Compression corruption** | Checksum on compressed block; decompress-verify on read; fallback to backup if corrupt |
| **Fingerprint table loss** | Rebuilt from scanning all blocks (slow but possible); RAID-protected SSD |
| **Power failure during dedup** | NVRAM journals all metadata changes; atomic commit for ref count updates |

## 9. Latency

| Operation | Without Efficiency | With Inline Dedup+Compress |
|-----------|--------------------|---------------------------|
| 4KB write (unique) | 200μs | 250μs (+25%) — hash + compress |
| 4KB write (duplicate) | 200μs | 100μs (-50%) — no physical write |
| 4KB read (uncompressed) | 200μs | 200μs (same) |
| 4KB read (compressed) | 200μs | 220μs (+10%) — decompress cost |

**Net effect:** For workloads with >30% duplication, overall latency improves (fewer physical writes).

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Dedup engine crash** | Writes proceed without dedup | Fallback: write as unique; post-process dedup later |
| **Fingerprint table corruption** | Dedup disabled until rebuilt | RAID-protected SSD; rebuild from block scan |
| **Hash collision (theoretical)** | Silent data corruption | SHA-256 probability negligible; byte-compare option |
| **Compression bug** | Block unreadable | Checksum detects; restore from snapshot/replication |

## 11. Availability

**Target: 99.999% — dedup/compression are inline data services, must not affect availability**

```
Availability Design:
  - Dedup engine failure: bypass and write without dedup (no downtime)
  - Compression failure: bypass and write uncompressed (no downtime)
  - Controller failover: partner controller continues with same fingerprint table
  - Non-disruptive upgrade: dedup metadata compatible across versions
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Encrypted data** | Encryption applied AFTER compression (compress then encrypt for security) |
| **Dedup with encryption** | Per-volume encryption keys: dedup only within same encryption boundary |
| **Cross-tenant dedup** | Disabled by default (dedup reveals data existence); per-tenant fingerprint tables |
| **Fingerprint privacy** | Fingerprints are one-way hashes; cannot reverse to original data |

## 13. Cost Constraints

**Savings Calculation (500TB logical):**

| Without Efficiency | With Efficiency | Savings |
|-------------------|-----------------|---------|
| 500TB physical SSDs needed | 150TB physical SSDs needed | 350TB saved |
| 500TB × $100/TB = $50,000/month | 150TB × $100/TB = $15,000/month | **$35,000/month** |

| Efficiency Cost | |
|----------------|---|
| **Additional CPU** | 15% overhead → $0 (existing controllers) |
| **RAM for Bloom + Cache** | 80GB → included in controller |
| **SSD for fingerprint table** | 2.5TB → $500/month |
| **Net monthly savings** | **$34,500/month** |

ROI: immediate — dedup/compression is a pure cost optimizer.

## Key Interview Discussion Points

1. **Inline vs post-process dedup?** — Inline: saves space immediately, slight latency overhead. Post-process: no latency impact but needs temporary extra space. Best: both (inline + background cleanup)
2. **Hash collision risk?** — SHA-256: 2^128 operations for 50% collision probability. For practical purposes, zero risk. Optional byte-compare adds certainty at cost of extra read
3. **Why not dedup encrypted data across tenants?** — Dedup reveals data existence (if my hash matches yours, we have same data). Security violation. Solution: per-tenant dedup boundaries
4. **Compression before or after encryption?** — Always compress first. Encrypted data has high entropy and won't compress. But: compress → encrypt
5. **Scaling fingerprint table for petabytes?** — Bloom filter (RAM) eliminates 99% lookups. B+ tree on SSD for persistent storage. Partition by aggregate for parallelism
