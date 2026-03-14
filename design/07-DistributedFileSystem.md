# Design a Distributed File System

Examples: HDFS, CephFS, GlusterFS, Google File System (GFS)

---

## 1. Requirements

### Functional
- Create, read, write, delete files and directories
- Support large files (GB to TB)
- Hierarchical namespace (directories and paths)
- Support concurrent readers and writers
- Append-heavy workload (logs, data pipelines)

### Non-Functional
- High throughput for large sequential reads/writes
- Fault tolerance (survive multiple node failures)
- Horizontal scalability (petabytes of data)
- Strong consistency for metadata operations
- Data locality awareness for compute frameworks

---

## 2. High-Level Architecture

```
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │   Client 1   │   │   Client 2   │   │   Client 3   │
  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
         │                  │                  │
         └──────────┬───────┘──────────────────┘
                    │    metadata ops (ls, open, mkdir)
           ┌────────▼────────┐
           │   Name Node     │   ← Metadata Server
           │  (Master)       │
           │  ┌────────────┐ │
           │  │ Namespace  │ │   file → block mapping
           │  │ Tree       │ │   block → datanode mapping
           │  │ Block Map  │ │
           │  └────────────┘ │
           │                 │
           │  Standby NN     │   ← HA (hot standby)
           └────────┬────────┘
                    │    data ops (read/write blocks)
         ┌──────────┼──────────┐
         ▼          ▼          ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │DataNode 1│ │DataNode 2│ │DataNode 3│
   │┌────────┐│ │┌────────┐│ │┌────────┐│
   ││Block A ││ ││Block A ││ ││Block B ││
   ││Block C ││ ││Block B ││ ││Block C ││
   │└────────┘│ │└────────┘│ │└────────┘│
   └──────────┘ └──────────┘ └──────────┘
```

---

## 3. Write Path

```
  Client                  NameNode              DataNodes
    │                        │                      │
    │──create("/data/f1")──▶│                      │
    │                        │──allocate blocks     │
    │                        │  block_1 → [DN1,DN2,DN3]
    │◀──block locations──────│                      │
    │                        │                      │
    │──write block_1 data──────────────────────────▶│ DN1
    │                        │                      │
    │                 DN1 ──pipeline──▶ DN2 ──pipeline──▶ DN3
    │                        │                      │
    │◀──ACK (all replicas)───│                      │
    │                        │                      │
    │──write block_2 data──────────────────────────▶│
    │       ...              │                      │
    │                        │                      │
    │──close()──────────────▶│                      │
    │                        │──commit file metadata │
    │◀──success──────────────│                      │

  Pipeline replication:
  • Client writes to DN1 only
  • DN1 forwards to DN2, DN2 forwards to DN3
  • ACK flows back: DN3 → DN2 → DN1 → Client
  • Minimises client network usage
```

---

## 4. Read Path

```
  Client                  NameNode              DataNodes
    │                        │                      │
    │──open("/data/f1")────▶│                      │
    │                        │                      │
    │◀──block locations──────│                      │
    │   block_1: [DN1, DN2, DN3]                    │
    │   block_2: [DN2, DN3, DN4]                    │
    │                        │                      │
    │──read block_1──────────────────▶ DN1 (closest)│
    │◀──data───────────────────────── DN1           │
    │                                               │
    │──read block_2──────────────────▶ DN2 (closest)│
    │◀──data───────────────────────── DN2           │

  Data locality:
  • Client prefers DataNode on same rack
  • Then same data center
  • Then any available replica
  • Compute frameworks (Spark, MapReduce) schedule tasks
    on nodes that hold the data blocks
```

---

## 5. Block Management

```
  File: /data/logfile.csv (320 MB)

  ┌─────────────────────────────────────────────────┐
  │  Block size: 128 MB (configurable)              │
  │                                                 │
  │  Block 1 (128 MB) → DN1, DN3, DN5              │
  │  Block 2 (128 MB) → DN2, DN4, DN6              │
  │  Block 3 (64 MB)  → DN1, DN2, DN4              │
  │                                                 │
  │  Replication factor: 3 (default)                │
  │  Placement policy:                              │
  │    Replica 1: local rack                        │
  │    Replica 2: different rack                    │
  │    Replica 3: same rack as replica 2            │
  │                                                 │
  │  Why large blocks?                              │
  │  • Reduce NameNode metadata overhead            │
  │  • Amortise seek time for large reads           │
  │  • Fewer network connections per file           │
  └─────────────────────────────────────────────────┘
```

---

## 6. NameNode HA (High Availability)

```
  ┌──────────────┐          ┌──────────────┐
  │  Active NN   │          │  Standby NN  │
  │  (primary)   │          │  (hot backup)│
  └──────┬───────┘          └──────┬───────┘
         │                         │
         │    shared edit log      │
         │  ┌──────────────────┐   │
         └─▶│  Journal Nodes   │◀──┘
            │  (quorum-based)  │
            │  JN1, JN2, JN3  │
            └──────────────────┘

  • Active NN writes all metadata changes to Journal Nodes
  • Standby NN reads from Journal Nodes and applies changes
  • On failure: Standby promotes to Active (< 30s)
  • Fencing: ensure old Active cannot write (STONITH)

  FsImage + EditLog:
  • FsImage: periodic snapshot of entire namespace
  • EditLog: all changes since last snapshot
  • Standby periodically checkpoints (merges EditLog into FsImage)
```

---

## 7. DataNode Failure & Recovery

```
  ┌──────────────────────────────────────────────┐
  │  Heartbeat mechanism:                        │
  │  DataNode ──heartbeat (every 3s)──▶ NameNode │
  │                                              │
  │  No heartbeat for 10 min → mark DN as dead   │
  │                                              │
  │  NameNode detects under-replicated blocks:   │
  │  Block A was on [DN1, DN2, DN3]              │
  │  DN3 is dead → only 2 replicas remaining     │
  │                                              │
  │  Re-replication:                             │
  │  NameNode instructs DN1 or DN2 to copy       │
  │  Block A to DN4 → restore 3 replicas         │
  │                                              │
  │  Priority: blocks with 1 replica replicated  │
  │  first (highest risk of data loss)           │
  └──────────────────────────────────────────────┘
```

---

## 8. Scaling Challenges

```
  NameNode bottleneck:
  ┌──────────────────────────────────────────────┐
  │  All metadata in NameNode memory             │
  │  Each block: ~150 bytes of metadata          │
  │  100 million blocks → ~15 GB RAM             │
  │  1 billion blocks → ~150 GB RAM              │
  │                                              │
  │  Solutions:                                  │
  │  1. Federation: multiple NameNodes, each     │
  │     owns a portion of the namespace          │
  │     /user → NN1, /data → NN2, /logs → NN3   │
  │                                              │
  │  2. Scale-out metadata (CephFS approach):    │
  │     Distribute metadata across MDS cluster   │
  │     Dynamic subtree partitioning             │
  │                                              │
  │  3. Larger block size: fewer blocks per file │
  └──────────────────────────────────────────────┘
```

---

## 9. HDFS vs CephFS Comparison

| Feature | HDFS | CephFS |
|---|---|---|
| Metadata | Single NameNode (+ HA) | Distributed MDS cluster |
| Data placement | NameNode decides | CRUSH algorithm (client computes) |
| Replication | Pipeline replication | Primary-copy with PG placement |
| POSIX compliance | No (append-only, no random writes) | Yes (full POSIX) |
| Best for | Batch processing (Hadoop/Spark) | General-purpose distributed FS |

---

---

## 10. Low-Level Design (LLD)

### API Contract (HDFS-style)
```
  Client → NameNode (metadata):
  POST   /nn/create?path=/data/file.txt&replication=3&blocksize=128MB
         Response: { "file_id": long, "block_locations": [] }

  GET    /nn/open?path=/data/file.txt
         Response: { "blocks": [{ "block_id": long, "datanodes": [dn1,dn2,dn3] }] }

  POST   /nn/mkdirs?path=/data/logs/2026/
  GET    /nn/ls?path=/data/&recursive=false
  DELETE /nn/delete?path=/data/old_file.txt&recursive=false

  Client → DataNode (data):
  PUT    /dn/block/{block_id}  (pipeline write to chain of DNs)
  GET    /dn/block/{block_id}?offset=0&length=128MB
  GET    /dn/block/{block_id}/checksum
```

### Namespace Metadata (NameNode)
```
  In-memory structures:
  ┌──────────────────────────────────────────────┐
  │  INode Tree (B-tree / flat array):           │
  │  /data/ → INode{id=1, type=DIR, children=[]} │
  │  /data/file.txt → INode{id=2, type=FILE,    │
  │    blocks=[blk_001, blk_002], repl=3}        │
  │                                              │
  │  Block Map:                                  │
  │  blk_001 → [DN-1, DN-3, DN-5]               │
  │  blk_002 → [DN-2, DN-4, DN-6]               │
  │                                              │
  │  Memory footprint:                           │
  │  ~150 bytes per file + ~150 bytes per block  │
  │  100M files → ~30 GB NameNode RAM            │
  └──────────────────────────────────────────────┘

  Persistence (EditLog + FsImage):
  ┌──────────────────────────────────────────────┐
  │  EditLog: append-only journal of operations  │
  │    [CREATE /data/file.txt] [ADD_BLOCK blk_1] │
  │  FsImage: periodic checkpoint of full state  │
  │  Checkpoint: merge EditLog into FsImage      │
  │  (done by SecondaryNN / StandbyNN)           │
  └──────────────────────────────────────────────┘
```

### DataNode Block Storage
```
  On-disk layout per DataNode:
  /data/
    /BP-xxx/current/finalized/
      /subdir0/
        blk_001         (data file, 128 MB)
        blk_001.meta     (checksums, 1 MB)
      /subdir1/
        blk_002
        blk_002.meta

  Block Report: each DN sends full block list to NN every 6 hours
  Incremental Block Report: deltas every 5 minutes
  Heartbeat: every 3 seconds (DN → NN alive signal)
```

---

## 11. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Storage Scaling:                                       │
  │  • Add DataNodes → linear capacity increase             │
  │  • Each DN: 12-36 disks × 16 TB = 192-576 TB           │
  │  • 1000 DNs → 500+ PB total capacity                   │
  │                                                         │
  │  Namespace Scaling (NameNode bottleneck):                │
  │  • Federation: multiple NameNodes, each owns a          │
  │    namespace volume (e.g., /data → NN1, /logs → NN2)   │
  │  • ViewFs: client-side routing to correct NN            │
  │  • Each NN: handle ~100M files in 64 GB RAM             │
  │                                                         │
  │  Throughput Scaling:                                     │
  │  • Read parallelism: client reads blocks from multiple  │
  │    DataNodes simultaneously                              │
  │  • Short-circuit local reads: bypass network if block   │
  │    is on same machine as client                          │
  │  • HDFS Router-based Federation (RBF) for unified view  │
  │                                                         │
  │  Balancer:                                               │
  │  • Background process equalizes disk usage across DNs   │
  │  • Throttled to avoid impacting live workloads          │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Write Pipeline Durability:                             │
  │  • Block replicated to 3 DataNodes before ACK to client│
  │  • Pipeline: DN1 → DN2 → DN3 (chain replication)       │
  │  • Client gets ACK only after all 3 DNs persist + fsync│
  │                                                         │
  │  Checksums:                                             │
  │  • CRC32C per 512-byte chunk in .meta file             │
  │  • DataNode verifies on write and periodic scrubbing   │
  │  • Client verifies on read → corrupt block → re-read   │
  │    from different replica                               │
  │                                                         │
  │  Under-Replication Handling:                             │
  │  • NameNode tracks replication count per block          │
  │  • Block with < 3 replicas → re-replication job queued │
  │  • Priority: 1 replica left > 2 replicas left          │
  │  • Rack-aware: ensure replicas in 2+ racks             │
  │                                                         │
  │  NameNode Durability:                                    │
  │  • EditLog: written to local disk + shared journal     │
  │    (NFS or JournalNodes — Quorum Journal Manager)      │
  │  • StandbyNN replays journal entries (hot standby)     │
  │  • FsImage: periodic checkpoint to durable storage      │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Latency

```
  Latency Targets:
  ┌──────────────────────┬──────────┬──────────┐
  │ Operation            │ p50      │ p99      │
  ├──────────────────────┼──────────┼──────────┤
  │ Create file (NN)     │ 5 ms     │ 20 ms   │
  │ Open file (NN)       │ 2 ms     │ 10 ms   │
  │ Read block (DN)      │ 10 ms    │ 50 ms   │
  │ Write block (pipeline)│ 50 ms   │ 200 ms  │
  │ ls (1000 entries)    │ 10 ms    │ 50 ms   │
  └──────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Short-circuit reads: bypass TCP for local blocks    │
  │  • Edits batching: NameNode batches EditLog fsyncs     │
  │  • Data locality: process data where blocks reside     │
  │  • Block caching: pin hot blocks in DN memory (SSD)    │
  │  • Read-ahead: OS page cache for sequential reads      │
  │  • Write pipelining: stream data while replicating     │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ DataNode crash         │ Blocks re-replicated from other │
  │                        │ replicas; client retries to     │
  │                        │ alternative DN                  │
  ├────────────────────────┼─────────────────────────────────┤
  │ NameNode crash         │ StandbyNN takes over via ZKFC   │
  │                        │ (ZooKeeper Failover Controller) │
  │                        │ Automatic failover < 30s        │
  ├────────────────────────┼─────────────────────────────────┤
  │ Network partition      │ Fencing: only one NN can be     │
  │                        │ active (prevent split brain)    │
  ├────────────────────────┼─────────────────────────────────┤
  │ Disk failure           │ JBOD: mark disk as failed;      │
  │                        │ blocks on failed disk →         │
  │                        │ re-replicate                    │
  ├────────────────────────┼─────────────────────────────────┤
  │ Rack failure           │ Rack-aware placement: at least  │
  │                        │ 1 replica in different rack      │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 15. Availability

```
  Target: 99.9% (batch workloads tolerant of brief outages)

  ┌─────────────────────────────────────────────────────────┐
  │  NameNode HA:                                           │
  │  • Active + Standby NN (hot standby)                    │
  │  • Shared EditLog via JournalNodes (3 or 5 nodes)       │
  │  • ZKFC: automatic failover using ZooKeeper             │
  │  • Failover time: ~30 seconds                           │
  │                                                         │
  │  DataNode Availability:                                 │
  │  • Blocks replicated to 3 nodes across 2+ racks         │
  │  • Client automatically retries on other replicas       │
  │  • Decommission: graceful removal with re-replication   │
  │                                                         │
  │  Rolling Upgrades:                                      │
  │  • Upgrade DNs one at a time (replication covers)       │
  │  • Upgrade StandbyNN → failover → upgrade old Active    │
  │  • No cluster downtime during upgrades                   │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication:                                        │
  │  • Kerberos: mandatory for production HDFS              │
  │  • Each DN and NN has Kerberos principal                 │
  │  • Delegation tokens for long-running jobs               │
  │                                                         │
  │  Authorization:                                         │
  │  • POSIX-style permissions (user/group/other, rwx)      │
  │  • HDFS ACLs: fine-grained access per user/group        │
  │  • Ranger/Sentry: centralized policy management         │
  │                                                         │
  │  Encryption:                                            │
  │  • In-transit: SASL/TLS for RPC, HTTPS for WebHDFS     │
  │  • At-rest: HDFS Transparent Data Encryption (TDE)      │
  │    per encryption zone with KMS-managed keys             │
  │  • Key rotation without re-encrypting data (DEK/KEK)    │
  │                                                         │
  │  Audit:                                                  │
  │  • Audit log: all file system operations logged          │
  │  • HDFS audit log → centralized SIEM                     │
  │  • Ranger audit: policy evaluation logs                  │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Storage:                                               │
  │  • JBOD (Just a Bunch of Disks) — no RAID overhead     │
  │  • 3× replication overhead (vs 1.5× for erasure coding)│
  │  • HDFS EC (Erasure Coding): RS(6,3) for cold data      │
  │    → 1.5× overhead, 50% storage saved                  │
  │                                                         │
  │  Compute:                                               │
  │  • DNs run on commodity hardware (cost-effective)       │
  │  • NameNode needs high RAM (128-256 GB for large clusters)│
  │  • Collocate compute (YARN/Spark) with storage (data    │
  │    locality) → reduce network costs                     │
  │                                                         │
  │  Cost Estimate (1 PB cluster, 3× replication):          │
  │  • 60 DataNodes × 16×16TB HDD = ~$250K hardware        │
  │  • 2 NameNodes (256 GB RAM each) = ~$20K hardware       │
  │  • Network (25/100 GbE): ~$30K                          │
  │  • Annual: ~$100K ops (power, cooling, admin)           │
  │  • Total: ~$400K for 1 PB usable (3-year amortized)    │
  │  • With EC on cold data: ~$300K (25% savings)          │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **NameNode single point of failure?** — HA with standby NN + journal nodes; federation for scale
2. **Why is block size 128 MB?** — Reduces metadata overhead, amortises disk seek time, optimised for batch processing
3. **How does data locality help MapReduce?** — Scheduler places tasks on nodes with data blocks → avoid network transfer
4. **HDFS vs object storage?** — HDFS: mutable files, directory hierarchy, fast sequential access. Object storage: immutable objects, flat namespace, HTTP API
5. **How do you handle small files problem?** — HAR (Hadoop Archive), sequence files, or compaction into larger files
