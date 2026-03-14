# Design an Erasure Coding System

## Overview
Design an erasure coding storage system like NetApp RAID-TEC / Ceph Erasure Coding / HDFS EC — providing space-efficient data protection beyond simple mirroring. Erasure coding encodes data into k data fragments + m parity fragments such that any k of (k+m) fragments can reconstruct the original data, tolerating up to m simultaneous failures.

## 1. Requirements

**Functional:**
- Encode data blocks into k data + m parity fragments (Reed-Solomon)
- Reconstruct data from any k fragments (tolerate up to m failures)
- Support multiple EC profiles: 4+2, 6+3, 8+4, RAID-DP (equivalent to 14+2)
- Online rebuild: reconstruct failed fragment from surviving fragments
- Stripe-level and object-level erasure coding
- Compatibility with tiered storage (fast tier + capacity tier)

**Non-Functional:**
- Space efficiency: 1.5× overhead (vs 3× for 3-way replication)
- Rebuild time: <4 hours for 8TB disk replacement
- Encode throughput: >2GB/s per node
- Decode (read) overhead: minimal for aligned reads; tolerable for degraded reads
- MTTDL: >10^18 hours (triple parity / RAID-TEC)

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Total capacity** | 5PB raw, 3.3PB usable (6+3 EC) |
| **Stripe width** | 9 fragments per stripe (6 data + 3 parity) |
| **Fragment size** | 4MB (object-level EC) or 4KB (stripe-level) |
| **Daily writes** | 20TB (new data) |
| **Disk failure rate** | 2% AFR (annual failure rate) |
| **Concurrent rebuilds** | Up to 3 per RAID group |

## 3. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│              Erasure Coding Storage System                      │
│                                                                │
│  Client Write Path:                                            │
│  ┌──────────┐                                                  │
│  │ Client   │  Data block: "ABCDEFGHIJKLMNOPQRSTUVWX"         │
│  └────┬─────┘                                                  │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────────────────────────────┐              │
│  │  EC Encoder (Reed-Solomon GF(2^8))          │              │
│  │                                              │              │
│  │  Input: k=6 data fragments                   │              │
│  │  ┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐      │              │
│  │  │ D0 ││ D1 ││ D2 ││ D3 ││ D4 ││ D5 │      │              │
│  │  │ABCD││EFGH││IJKL││MNOP││QRST││UVWX│      │              │
│  │  └────┘└────┘└────┘└────┘└────┘└────┘      │              │
│  │                                              │              │
│  │  Generate: m=3 parity fragments              │              │
│  │  ┌────┐┌────┐┌────┐                         │              │
│  │  │ P0 ││ P1 ││ P2 │   (calculated via       │              │
│  │  │xxxx││yyyy││zzzz│    Galois Field math)    │              │
│  │  └────┘└────┘└────┘                         │              │
│  │                                              │              │
│  │  Total: 9 fragments, any 6 reconstruct data  │              │
│  │  Space overhead: 9/6 = 1.5× (vs 3× replica) │              │
│  └──────────────────────────────────────────────┘              │
│       │                                                        │
│       ▼  Distribute fragments to different failure domains     │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  Node1  Node2  Node3  Node4  Node5  Node6  Node7    │      │
│  │  ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   │      │
│  │  │D0│   │D1│   │D2│   │D3│   │D4│   │D5│   │P0│   │      │
│  │  └──┘   └──┘   └──┘   └──┘   └──┘   └──┘   └──┘   │      │
│  │                                                      │      │
│  │  Node8  Node9                                        │      │
│  │  ┌──┐   ┌──┐                                         │      │
│  │  │P1│   │P2│   ← Each fragment on different node     │      │
│  │  └──┘   └──┘     (or different disk in RAID-TEC)     │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                │
│  Tolerate: any 3 of 9 nodes/disks failing simultaneously      │
│  Reconstruct: read any 6 of 9 fragments → decode → original   │
└────────────────────────────────────────────────────────────────┘
```

## 4. Reed-Solomon Encoding Mathematics

```
Reed-Solomon Encoding (Galois Field GF(2^8)):

  Encoding Matrix (Vandermonde / Cauchy):
  
  For k=6, m=3:
  
  ┌                              ┐   ┌    ┐   ┌    ┐
  │  1  0  0  0  0  0            │   │ D0 │   │ D0 │  (identity:
  │  0  1  0  0  0  0            │   │ D1 │   │ D1 │   data passes
  │  0  0  1  0  0  0            │   │ D2 │   │ D2 │   through
  │  0  0  0  1  0  0            │ × │ D3 │ = │ D3 │   unchanged)
  │  0  0  0  0  1  0            │   │ D4 │   │ D4 │
  │  0  0  0  0  0  1            │   │ D5 │   │ D5 │
  │  α₀⁰ α₁⁰ α₂⁰ α₃⁰ α₄⁰ α₅⁰│   │    │   │ P0 │  (parity:
  │  α₀¹ α₁¹ α₂¹ α₃¹ α₄¹ α₅¹│   │    │   │ P1 │   GF multiply
  │  α₀² α₁² α₂² α₃² α₄² α₅²│   │    │   │ P2 │   + accumulate)
  └                              ┘   └    ┘   └    ┘
  
  Where αᵢ are elements of GF(2^8), all math is modular

  Recovery (3 fragments lost):
    1. Select any 6 surviving fragments
    2. Extract corresponding 6 rows from encoding matrix
    3. Invert the 6×6 sub-matrix (Gaussian elimination in GF)
    4. Multiply inverse × surviving data = original data
    
  Example: D1, D3, P0 lost (3 failures):
    Surviving: D0, D2, D4, D5, P1, P2 (6 fragments)
    Take rows 0, 2, 4, 5, 7, 8 from encoding matrix
    Invert → recovery matrix
    recovery_matrix × [D0, D2, D4, D5, P1, P2]ᵀ = [D0..D5]ᵀ

  Computational Cost:
    Encode: O(k × m × fragment_size) GF multiplications
    Decode (no failure): zero cost — read k data fragments directly
    Decode (degraded): O(k² × fragment_size) — matrix inversion + multiply

  Implementation (ISA-L library):
    // Intel ISA-L provides SIMD-optimized GF arithmetic
    ec_encode_data(len, k, m, encode_matrix, data_ptrs, parity_ptrs);
    ec_encode_data_update(len, k, m, row, encode_matrix, data, parity_ptrs);
```

## 5. RAID-TEC vs Object-Level EC

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  RAID-TEC (Triple Erasure Coding) — NetApp                   │
│  ═══════════════════════════════════════                      │
│  Stripe-level EC across disks within an aggregate:           │
│                                                               │
│  RAID Group (29 disks):                                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ D0 D1 D2 ... D25  P_row  P_diag  P_antidiag          │  │
│  │ ─── ─── ───       ───    ───     ───                  │  │
│  │ 26 data disks + 3 parity disks per RAID group         │  │
│  │ Tolerate: any 3 disk failures simultaneously          │  │
│  │ Space overhead: 29/26 = 1.115× (88.5% efficiency!)    │  │
│  │                                                        │  │
│  │ Parity calculation (simplified):                       │  │
│  │   P_row = XOR of all data in row                      │  │
│  │   P_diag = XOR along diagonal pattern                 │  │
│  │   P_antidiag = XOR along anti-diagonal pattern        │  │
│  │                                                        │  │
│  │ Write penalty: each write updates 3 parity disks      │  │
│  │ Read: direct read from data disk (no decode needed)   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  vs.                                                          │
│                                                               │
│  Object-Level EC (Ceph / HDFS) — Distributed                │
│  ════════════════════════════════════════════                  │
│  EC across nodes in a cluster:                               │
│                                                               │
│  Object (e.g., 24MB):                                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Split into k=6 fragments of 4MB each                  │  │
│  │ Generate m=3 parity fragments of 4MB each             │  │
│  │ Place each fragment on different node/failure domain    │  │
│  │                                                        │  │
│  │ Space overhead: 9/6 = 1.5× (66.7% efficiency)        │  │
│  │ Tolerate: any 3 node failures                         │  │
│  │                                                        │  │
│  │ Write: client encodes + sends 9 fragments to 9 nodes  │  │
│  │ Read: client reads 6 data fragments (parallel)        │  │
│  │ Degraded read: read from 6 surviving → decode         │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  Comparison:                                                  │
│  ┌──────────────┬───────────┬──────────────┐                 │
│  │ Property     │ RAID-TEC  │ Object EC    │                 │
│  ├──────────────┼───────────┼──────────────┤                 │
│  │ Efficiency   │ 88.5%     │ 66.7%        │                 │
│  │ Failure scope│ Disk      │ Node/rack    │                 │
│  │ Write penal. │ 3× parity │ k+m writes   │                 │
│  │ Read latency │ Single    │ k parallel   │                 │
│  │ Rebuild I/O  │ Disk-local│ Network-wide │                 │
│  │ Best for     │ Enterprise│ Scale-out    │                 │
│  │              │ arrays    │ clusters     │                 │
│  └──────────────┴───────────┴──────────────┘                 │
└──────────────────────────────────────────────────────────────┘
```

## 6. Rebuild Process

```
Disk/Fragment Failure Rebuild:

  Failed: Disk D2 in a 6+3 EC stripe

  Step 1: Identify affected stripes
    - Scan metadata: all stripes containing D2 fragments
    - Priority: stripes with most degradation first
    
  Step 2: Read k surviving fragments per stripe
    - Read D0, D1, D3, D4, D5, P0 (any 6 of remaining 8)
    - Prefer data fragments (no decode needed)
    - Parallel reads from multiple disks/nodes
    
  Step 3: Reconstruct D2
    - Build recovery matrix from encoding matrix
    - Multiply: recovery_matrix × surviving_fragments = D2
    
  Step 4: Write reconstructed D2 to spare disk/node
    - Atomic per-stripe reconstruction
    - Checksum verification after write

  Rebuild Time Estimation (8TB disk, RAID-TEC):
    Data to reconstruct: 8TB
    Read speed: 200 MB/s per surviving disk
    Parallel reads: 28 disks × 200 MB/s = 5600 MB/s effective
    Rebuild bottleneck: spare disk write = 200 MB/s
    Time: 8TB / 200 MB/s ≈ 11 hours (but 20% I/O budget → ~4 hrs)
    
  Priority Rebuild (Urgent Reconstruction):
    - Focus on stripes with double degradation first
    - Allocate more I/O bandwidth to rebuild
    - Can reduce rebuild time by 50% with full-speed rebuild
    
  MTTDL Calculation (RAID-TEC, 29 disks, AFR=2%):
    P(4 failures before rebuild completes)
    = C(29,4) × (AFR)^4 × (rebuild_time/8760)^3
    ≈ 10^-18 per year → practically impossible
```

## 7. Low-Level Design (LLD)

### API Contracts

```
# Create EC Pool/Profile
POST /api/v1/ec/profiles
{
  "name": "ec-6-3",
  "k": 6,                          // data fragments
  "m": 3,                          // parity fragments
  "fragment_size": 4194304,         // 4MB
  "algorithm": "reed_solomon",
  "implementation": "isa_l",       // Intel ISA-L (SIMD optimized)
  "placement_rule": "rack_aware"   // fragments on different racks
}

# Encode Object
POST /api/v1/ec/encode
{
  "object_id": "obj-2024-001",
  "data": "<base64-encoded data or reference>",
  "profile": "ec-6-3"
}
→ {
  "object_id": "obj-2024-001",
  "fragments": [
    {"id": "frag-0", "type": "data",   "node": "node-1", "checksum": "a1b2..."},
    {"id": "frag-1", "type": "data",   "node": "node-2", "checksum": "c3d4..."},
    ...
    {"id": "frag-6", "type": "parity", "node": "node-7", "checksum": "e5f6..."},
    {"id": "frag-7", "type": "parity", "node": "node-8", "checksum": "g7h8..."},
    {"id": "frag-8", "type": "parity", "node": "node-9", "checksum": "i9j0..."}
  ],
  "encoding_time_ms": 12,
  "space_amplification": 1.5
}

# Decode / Read Object
GET /api/v1/ec/objects/{object_id}
→ Reads k data fragments in parallel; returns decoded object
   If degraded: reads k surviving fragments, decodes, returns

# Rebuild (triggered automatically or manually)
POST /api/v1/ec/rebuild
{
  "failed_node": "node-3",
  "target_node": "node-10",
  "priority": "high",              // high | normal | background
  "io_bandwidth_limit": "500MB/s"
}
→ {
  "rebuild_job_id": "rebuild-001",
  "fragments_to_rebuild": 1234567,
  "estimated_time": "3h 42m",
  "progress_endpoint": "/api/v1/ec/rebuild/rebuild-001"
}

# RAID-TEC Aggregate Status
GET /api/v1/storage/aggregates/{aggr}/raid
→ {
  "raid_type": "raid_tec",
  "raid_groups": [
    {
      "name": "rg0",
      "disks": 29,
      "data_disks": 26,
      "parity_disks": 3,
      "failed_disks": 0,
      "rebuilding": false,
      "checksum": "block"
    }
  ],
  "scrub_last": "2024-07-10T00:00:00Z",
  "scrub_errors_found": 0
}
```

## 8. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Data volume** | Multiple RAID groups / EC pools | Petabytes |
| **Encode throughput** | ISA-L SIMD (AVX-512): 10GB/s per core | Scale with CPU |
| **Fragment distribution** | Rack-aware placement; CRUSH/Consistent Hash | 1000s of nodes |
| **Rebuild parallelism** | Read from all surviving nodes simultaneously | Proportional to cluster size |
| **EC profile flexibility** | Different profiles for different tiers (4+2, 8+4, 16+4) | Per-pool config |

## 9. No Data Loss

| Scenario | Protection |
|----------|------------|
| **1-2 disk failures** | Tolerated by RAID-DP (or RAID-TEC with margin) |
| **3 simultaneous failures** | RAID-TEC tolerates up to 3 (triple parity) |
| **Silent data corruption** | Block checksums detect; EC reconstruct from parity |
| **Media failure (URE)** | URE during rebuild: 2 remaining parity covers it |
| **Full node failure (EC)** | m=3: tolerate any 3 node failures; rebuild to spare nodes |

## 10. Latency

| Operation | Object EC (6+3) | RAID-TEC |
|-----------|-----------------|----------|
| **Write (normal)** | 1 encode + 9 parallel writes: ~5ms | 1 write + 3 parity updates: ~1ms |
| **Read (normal)** | 6 parallel reads + network: ~3ms | Single disk read: ~200μs |
| **Read (degraded)** | 6 reads + decode: ~8ms | Read + parity reconstruct: ~1ms |
| **Encode CPU** | ~0.1ms per MB (ISA-L AVX-512) | Hardware offload possible |
| **Decode CPU** | ~0.3ms per MB (matrix inversion) | Minimal (XOR-based) |

## 11. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Single disk/node failure** | No impact; background rebuild | Auto-rebuild to spare; priority if >1 failure |
| **Double failure** | Still protected (m≥3) | Accelerate rebuild; alert |
| **Triple failure (RAID-TEC)** | At tolerance limit | MTTDL ~10^18 hours; practically never |
| **Quadruple failure** | Data loss for affected stripes | Never observed in practice with RAID-TEC |
| **Silent corruption** | Undetected without checksums | T10-DIF / block checksums + scrubbing |
| **Rebuild storm** | High I/O; slow client ops | I/O throttling; QoS for client traffic |

## 12. Availability

**Target: 99.999% (< 5 minutes downtime per year)**

- RAID-TEC: continues serving I/O through up to 3 disk failures
- Object EC: continues serving through m node failures (degraded reads)
- No rebuild required for reads (parity reconstruction on-the-fly)
- Background scrub detects latent errors before they compound

## 13. Security

| Layer | Mechanism |
|-------|-----------|
| **Data confidentiality** | Encrypt before EC encode; each fragment is ciphertext |
| **Fragment integrity** | Per-fragment checksum (CRC-32C or SHA-256) |
| **Tamper detection** | Scrubbing: periodic verify of all fragments vs checksums |
| **Secure erasure** | Crypto-erase: destroy encryption key → fragments unreadable |
| **Access control** | EC metadata protected by storage system RBAC |

## 13. Cost Constraints

**Estimated Cost (5PB raw, 6+3 EC):**

| Component | Cost |
|-----------|------|
| **Storage nodes (9 nodes)** | 9 × $50,000 = $450,000 |
| **Raw disks (5PB, 18TB HDDs)** | 280 × $300 = $84,000 |
| **Networking (25GbE)** | $50,000 |
| **Total hardware** | **$584,000** |
| **Usable capacity** | 3.3PB |
| **Cost per usable TB** | **$177/TB** |

Comparison:
- 3× replication: 5PB raw → 1.67PB usable → $350/TB (2× more expensive)
- RAID-TEC: 5PB raw → 4.4PB usable → $133/TB (most efficient)

## Key Interview Discussion Points

1. **EC vs replication trade-off?** — EC (6+3): 1.5× overhead, higher CPU, longer rebuild, higher read latency. Replication (3×): 3× overhead, zero CPU, instant failover, lowest latency. Use EC for cold/warm data; replication for hot data
2. **Why not just RAID-6 (dual parity)?** — RAID-6 tolerates 2 failures. With modern 18TB disks, rebuild takes 10+ hours. Probability of a 2nd failure during that window is non-trivial. Triple parity (RAID-TEC) covers the 3rd failure during rebuild
3. **Reed-Solomon vs LRC?** — Local Reconstruction Codes (LRC) add local parity within subgroups, reducing rebuild I/O from k reads to subgroup-size reads. Azure uses LRC (12,2,2) — requires only 6 reads for local repair instead of 12. Trade-off: slightly more overhead for much faster repair
4. **ISA-L performance?** — Intel ISA-L uses SIMD (AVX2/AVX-512) for Galois Field arithmetic. Achieves 10-50GB/s encode throughput per core. Without SIMD, GF multiplication is 100× slower
5. **Write amplification?** — For k+m EC, each small write requires reading k fragments, re-encoding, writing k+m fragments. Solution: log-structured writes (batch small writes → full-stripe writes) or partial-stripe updates with journaling
