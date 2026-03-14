# Design a Distributed Transaction System

## Overview
Design a distributed transaction system supporting ACID guarantees across multiple shards/services — implementing Two-Phase Commit (2PC), Saga pattern, MVCC, and TrueTime-like global ordering for a globally distributed database similar to Google Spanner or CockroachDB.

## 1. Requirements

**Functional:**
- Distributed ACID transactions spanning multiple shards
- Serializable isolation (strongest guarantee)
- MVCC for non-blocking reads
- Global transaction ordering (external consistency)
- Deadlock detection and resolution
- Savepoints and rollback

**Non-Functional:**
- Single-shard transaction: <5ms
- Multi-shard transaction: <50ms (same region)
- Cross-region transaction: <200ms
- Throughput: 100K transactions/second
- Availability: 99.999%

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Shards** | 1,000 across 5 regions |
| **Nodes** | 5,000 (5 replicas per shard) |
| **Transactions/sec** | 100,000 |
| **Multi-shard txn ratio** | 20% (80% single-shard) |
| **Avg keys per transaction** | 5 |
| **Data size** | 50TB total |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│           Distributed Transaction Architecture                    │
│                                                                   │
│  Client                                                          │
│    │  BEGIN; UPDATE t1 SET x=1; UPDATE t2 SET y=2; COMMIT;       │
│    ▼                                                              │
│  ┌────────────────────────────────────────────────────────┐      │
│  │  Transaction Coordinator (Gateway)                      │      │
│  │                                                         │      │
│  │  txn_id: T42                                           │      │
│  │  status: PENDING                                       │      │
│  │  participants: [Shard-A, Shard-B]                      │      │
│  │  start_ts: TrueTime.now()                              │      │
│  │  commit_ts: (assigned at commit)                       │      │
│  └──────┬────────────────────┬─────────────────────────────┘      │
│         │                    │                                    │
│  ┌──────▼──────────┐  ┌─────▼──────────┐                        │
│  │ Shard A (Leader) │  │ Shard B (Leader) │                        │
│  │                  │  │                  │                        │
│  │ MVCC Store:      │  │ MVCC Store:      │                        │
│  │ key=x:           │  │ key=y:           │                        │
│  │  v@ts40: old     │  │  v@ts35: old     │                        │
│  │  v@ts42: 1 (new) │  │  v@ts42: 2 (new) │                        │
│  │  (write intent)  │  │  (write intent)  │                        │
│  │                  │  │                  │                        │
│  │ Lock Table:      │  │ Lock Table:      │                        │
│  │  x → T42 (Write) │  │  y → T42 (Write) │                        │
│  └──────────────────┘  └──────────────────┘                        │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  TrueTime / Hybrid Logical Clock (HLC)                    │    │
│  │  Provides globally ordered timestamps                      │    │
│  │  GPS + atomic clocks → bounded uncertainty [earliest,      │    │
│  │                                              latest]       │    │
│  └──────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

## 4. Two-Phase Commit (2PC)

```
2PC Protocol:

Phase 1: PREPARE
  Coordinator → all participants: PREPARE(txn_id=T42)
  
  Each participant:
    1. Validate transaction (constraints, conflicts)
    2. Write prepare record to WAL (durable)
    3. Acquire all locks
    4. Reply: VOTE_COMMIT or VOTE_ABORT
    
  If ANY participant votes ABORT → entire txn aborts
  
Phase 2: COMMIT (or ABORT)
  Coordinator:
    All votes = COMMIT:
      1. Write COMMIT decision to WAL (durable) ← commit point
      2. Assign commit_timestamp via TrueTime
      3. → all participants: COMMIT(txn_id=T42, commit_ts=ts42)
      
    Any vote = ABORT:
      1. Write ABORT decision to WAL
      2. → all participants: ABORT(txn_id=T42)
  
  Each participant on COMMIT:
    1. Make writes visible at commit_ts
    2. Release locks
    3. Write COMMITTED to WAL
    4. ACK to coordinator
    
Timeline:
  Client        Coordinator      Shard A         Shard B
    │                │               │               │
    │── BEGIN ──────►│               │               │
    │── UPDATE x ───►│── write ─────►│               │
    │── UPDATE y ───►│── write ─────►│──────────────►│
    │── COMMIT ─────►│               │               │
    │                │── PREPARE ───►│               │
    │                │── PREPARE ───►│──────────────►│
    │                │◄── VOTE_YES ──│               │
    │                │◄── VOTE_YES ──│◄──────────────│
    │                │               │               │
    │                │ (commit point)│               │
    │                │── COMMIT ────►│               │
    │                │── COMMIT ────►│──────────────►│
    │◄── SUCCESS ────│               │               │
```

## 5. MVCC (Multi-Version Concurrency Control)

```
MVCC Storage:

Key "account_balance" versions:
  ┌──────────────────────────────────────────────────┐
  │ Key: account_balance                              │
  │                                                   │
  │ Version History (newest first):                   │
  │   ts=42: value=1000, txn=T42 (write intent)     │
  │   ts=40: value=950,  committed ✓                 │
  │   ts=35: value=800,  committed ✓                 │
  │   ts=20: value=500,  committed ✓                 │
  │                                                   │
  │ Write Intent = uncommitted write, visible only    │
  │   to txn T42. Other txns see ts=40 value.        │
  └──────────────────────────────────────────────────┘

Read at timestamp ts=38:
  → scan versions: find latest where ts ≤ 38
  → return value=800 (ts=35)
  → skip ts=40 (future), skip ts=42 (uncommitted intent)

Read at timestamp ts=41:
  → return value=950 (ts=40, committed)
  → skip ts=42 (write intent, different txn)
  → if reader encounters write intent from concurrent txn:
      Option A: wait for txn to commit/abort (blocking)
      Option B: push timestamp forward (non-blocking, CockroachDB)

Write Conflict Detection:
  class MVCCStore:
    def write(self, key, value, txn):
      # Check for write-write conflict
      latest = self.get_latest_version(key)
      
      if latest.timestamp > txn.read_timestamp:
        # Another txn wrote after our read → conflict!
        raise WriteConflictError()  # txn must retry
      
      if latest.is_write_intent and latest.txn_id != txn.id:
        # Another concurrent txn has uncommitted write
        # Push or wait depending on priority
        self.resolve_intent(latest, txn)
      
      # Write our intent
      self.put_version(key, value, txn.timestamp, intent=True)
      self.lock_table.acquire(key, txn.id, WRITE)
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Begin Transaction
POST /api/v1/transactions
{
  "isolation": "serializable",    // serializable | snapshot | read_committed
  "priority": "normal",           // normal | high (affects conflict resolution)
  "read_only": false
}
→ { "txn_id": "T42", "read_ts": 1705312800.123 }

# Execute Statement within Transaction
POST /api/v1/transactions/{txn_id}/execute
{
  "sql": "UPDATE accounts SET balance = balance - 100 WHERE id = 'alice'",
  "params": {}
}
→ { "rows_affected": 1 }

# Commit Transaction
POST /api/v1/transactions/{txn_id}/commit
→ {
  "status": "committed",
  "commit_ts": 1705312800.456,
  "duration_ms": 12
}

# Abort Transaction (explicit or on conflict)
POST /api/v1/transactions/{txn_id}/abort
→ { "status": "aborted" }
```

### Deadlock Detection

```
Wait-For Graph:

  T1 → waits for T2 (T2 holds lock on key_x)
  T2 → waits for T3 (T3 holds lock on key_y)
  T3 → waits for T1 (T1 holds lock on key_z)
  
  → CYCLE DETECTED: deadlock!

Resolution:
  class DeadlockDetector:
    def detect(self):
      # Build wait-for graph from lock table
      graph = {}
      for lock in lock_table.all_locks():
        for waiter in lock.waiters:
          graph.setdefault(waiter.txn_id, []).append(lock.holder.txn_id)
      
      # DFS to find cycles
      cycles = find_cycles(graph)
      
      for cycle in cycles:
        # Abort the youngest transaction (least work lost)
        victim = min(cycle, key=lambda t: t.start_timestamp)
        victim.abort("deadlock detected")
    
    # Run every 100ms (distributed: coordinator per shard range)
```

### TrueTime / HLC

```
Global Timestamp Ordering:

TrueTime (Google Spanner):
  GPS + atomic clocks → returns interval [earliest, latest]
  Uncertainty: ε ≈ 7ms
  
  Commit wait: after assigning commit_ts, wait ε duration
  → guarantees: if T1 commits before T2 starts,
    T1.commit_ts < T2.start_ts (external consistency)

Hybrid Logical Clock (CockroachDB alternative):
  HLC = max(wall_clock, highest_seen_timestamp) + logical_counter
  
  No commit wait needed (no GPS)
  Trade-off: uncertainty window handled by read refresh:
    If read might be stale due to clock skew:
      → refresh read timestamp
      → if no conflicting writes in [old_ts, new_ts]: safe
      → if conflicting write found: restart transaction

  class HLC:
    def now(self):
      wall = physical_clock()
      self.ts = max(self.ts, wall)
      self.logical = 0 if self.ts > wall else self.logical + 1
      return (self.ts, self.logical)
    
    def update(self, remote_ts):
      """Called when receiving RPC with timestamp"""
      old = self.ts
      self.ts = max(self.ts, remote_ts.ts, physical_clock())
      if self.ts == old == remote_ts.ts:
        self.logical = max(self.logical, remote_ts.logical) + 1
      elif self.ts == old:
        self.logical += 1
      elif self.ts == remote_ts.ts:
        self.logical = remote_ts.logical + 1
      else:
        self.logical = 0
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Shards** | Hash or range partitioning; add shards for capacity | 1,000+ shards |
| **Transactions/sec** | Single-shard txn: no coordination overhead; batching | 100K+ txn/sec |
| **Multi-shard** | Parallel prepare across participants; pipeline 2PC | 20K multi-shard/sec |
| **Geographic** | Raft replication per shard; follow-the-sun leader placement | 5 regions |
| **Read scaling** | Follower reads at consistent timestamp; learner replicas | Unlimited read replicas |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Coordinator crash after prepare** | Participants hold locks; new coordinator recovers from WAL; complete or abort |
| **Participant crash after vote-commit** | WAL persists prepare record; on recovery, ask coordinator for decision |
| **Both crash** | WAL on both sides; recovery protocol resolves pending transactions |
| **Network partition during 2PC** | Blocking: participants that voted-commit hold locks until coordinator reachable. Non-blocking: 3PC or Paxos commit |

## 9. Latency

| Operation | Same Region | Cross-Region |
|-----------|-------------|--------------|
| Single-shard read | <1ms | <1ms (local replica) |
| Single-shard write | 3-5ms (Raft commit) | 3-5ms (regional leader) |
| Multi-shard read | 1-2ms | 50-100ms (if cross-region) |
| Multi-shard write (2PC) | 10-30ms | 100-300ms (2 RTTs cross-region) |

**Optimization:** Parallel prepare (all participants simultaneously), 1PC optimization for single-shard transactions, read-only transactions avoid 2PC entirely.

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Coordinator failure** | Pending txn blocked until recovery | Raft-replicated coordinator state; new leader continues |
| **Participant failure** | That shard unavailable | Raft replication; new leader elected; pending txn resolved |
| **Network partition** | Cross-partition txn blocked | Timeout → abort; retry when healed |
| **Clock skew** | Potential stale reads (HLC) or commit wait delay (TrueTime) | HLC: read refresh. TrueTime: wait ε (7ms) |

## 11. Availability

**Target: 99.999%**

```
Availability Design:
  - Each shard: 5 Raft replicas across AZs → survive 2 failures
  - Coordinator: Raft-replicated → no SPOF
  - Read availability: follower reads even during leader election
  - Write availability: new leader elected in <5s
  
  Single-shard txn availability ≈ shard availability ≈ 99.9999%
  Multi-shard txn availability = product of participant availabilities
    2-shard txn: 99.999% × 99.999% = 99.998%
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | mTLS between all nodes; client certificates |
| **Authorization** | Row-level security; column-level encryption for sensitive fields |
| **Encryption in transit** | TLS 1.3 for all inter-node and client communication |
| **Encryption at rest** | AES-256 for WAL and data files |
| **Audit** | Transaction log with client identity, timestamp, all mutations |
| **SQL injection** | Parameterized queries; statement allowlisting |

## 13. Cost Constraints

**Estimated Cost (50TB, 5 regions, 100K txn/sec):**

| Component | Cost/Month |
|-----------|------------|
| **Compute** | 5,000 nodes × $200/month (c5.xlarge): $1,000,000 |
| **Storage** | 50TB data × 5 replicas × $0.10/GB: $25,000 |
| **Network** | Cross-region transfer (~10TB/month): $5,000 |
| **GPS/atomic clocks** | Per-datacenter appliance: $50,000 one-time |
| **Total** | **~$1,030,000/month** |

Spanner/CockroachDB managed service: roughly $0.30/vCPU-hour + storage.

## Key Interview Discussion Points

1. **2PC blocking problem?** — If coordinator crashes after sending PREPARE, participants that voted COMMIT hold locks indefinitely. Solutions: replicate coordinator via Raft; use Paxos Commit (non-blocking); timeout + abort heuristic
2. **TrueTime vs HLC?** — TrueTime: bounded uncertainty, requires GPS/atomic clocks, enables commit-wait for external consistency. HLC: software-only, handles clock skew via read refresh, simpler deployment
3. **Serializable vs Snapshot Isolation?** — Serializable: no anomalies but more conflicts and aborts. Snapshot: allows write skew anomaly but higher throughput. Choose based on application correctness needs
4. **How does MVCC handle garbage collection?** — Old versions needed only until no active transaction reads them. GC watermark = oldest active transaction's read timestamp. Compact versions below watermark
5. **Saga pattern vs 2PC?** — 2PC: strong consistency but blocking and tight coupling. Saga: compensating transactions, eventual consistency, better for microservices. Use 2PC for database internals, Saga for service orchestration
