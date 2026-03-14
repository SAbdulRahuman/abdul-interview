# Design a Consensus System

## Overview
Design a distributed consensus system implementing Raft (or Paxos) — enabling a cluster of nodes to agree on a shared state (leader election, replicated log, configuration changes) with strong consistency guarantees despite node failures and network partitions.

## 1. Requirements

**Functional:**
- Leader election: elect single leader from N nodes in bounded time
- Log replication: replicate ordered entries to majority before commit
- Safety: never return inconsistent (stale or conflicting) results
- Membership changes: add/remove nodes without downtime
- Snapshot & compaction: prevent unbounded log growth

**Non-Functional:**
- Commit latency: <10ms (LAN), <100ms (WAN)
- Leader election time: <5 seconds
- Availability: tolerates ⌊(N-1)/2⌋ failures (3-node: 1 failure, 5-node: 2 failures)
- Throughput: 100K entries/second (batched)

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Cluster size** | 5 nodes (typical) |
| **Log entries/sec** | 100,000 (batched) |
| **Entry size** | 256 bytes average |
| **Log throughput** | 25 MB/s |
| **Snapshot interval** | Every 100K entries |
| **State machine size** | 1-100GB |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│               Raft Consensus Architecture                     │
│                                                               │
│  Client                                                      │
│    │                                                         │
│    │ (1) Write request                                       │
│    ▼                                                         │
│  ┌────────────────────────────────────┐                      │
│  │ Node 1 (LEADER)          Term: 5  │                      │
│  │                                   │                      │
│  │ ┌─────────────────────────────┐   │                      │
│  │ │ Log: [1:SET x=1] [2:SET y=2]│   │                      │
│  │ │      [3:SET z=3] [4:DEL x]  │   │                      │
│  │ │      [5:SET w=4] ◄── new    │   │                      │
│  │ └─────────────────────────────┘   │                      │
│  │                                   │                      │
│  │ State Machine (applied: 1-4)      │                      │
│  └──────┬─────────┬─────────┬────────┘                      │
│         │ (2)     │ (2)     │ (2)                           │
│         │ AppendEntries RPCs                                 │
│         ▼         ▼         ▼                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                    │
│  │ Node 2   │ │ Node 3   │ │ Node 4   │  Node 5           │
│  │ FOLLOWER │ │ FOLLOWER │ │ FOLLOWER │  (failed)          │
│  │ Term: 5  │ │ Term: 5  │ │ Term: 5  │                    │
│  │ Log: 1-5 │ │ Log: 1-5 │ │ Log: 1-4 │                    │
│  │ (3) ACK  │ │ (3) ACK  │ │ catching │                    │
│  └──────────┘ └──────────┘ │   up     │                    │
│                             └──────────┘                    │
│                                                               │
│  (4) Majority ACK received (Node 1,2,3 = 3/5)              │
│  (5) Entry 5 COMMITTED → apply to state machine            │
│  (6) Reply to client: SUCCESS                               │
└───────────────────────────────────────────────────────────────┘
```

## 4. Leader Election

```
Election Protocol:

Node States:
  FOLLOWER → CANDIDATE → LEADER

Timeline:
  T=0:    Leader sends heartbeats every 150ms
  T=0-∞:  Followers reset election timer on heartbeat receipt
  
  Leader Failure:
  T=300ms: Follower's election timer expires (randomized 150-300ms)
           → becomes CANDIDATE
           → increments term (5→6)
           → votes for self
           → sends RequestVote RPCs to all nodes

RequestVote RPC:
  message RequestVote {
    term: 6,
    candidate_id: "node-3",
    last_log_index: 5,
    last_log_term: 5
  }

Vote Decision:
  def handle_request_vote(self, request):
    # Reject if candidate's term is old
    if request.term < self.current_term:
      return VoteResponse(granted=False)
    
    # Reject if already voted for someone else this term
    if self.voted_for is not None and self.voted_for != request.candidate_id:
      return VoteResponse(granted=False)
    
    # Reject if candidate's log is LESS up-to-date
    if not self.is_log_up_to_date(request):
      return VoteResponse(granted=False)
    
    # Grant vote
    self.voted_for = request.candidate_id
    self.persist_state()  # durably write voted_for
    return VoteResponse(granted=True)
  
  def is_log_up_to_date(self, request):
    """Candidate must have log at least as up-to-date as voter"""
    my_last_term = self.log[-1].term
    my_last_index = len(self.log) - 1
    
    if request.last_log_term != my_last_term:
      return request.last_log_term > my_last_term
    return request.last_log_index >= my_last_index

Election Result:
  - Receives majority votes → becomes LEADER
  - Receives AppendEntries from new leader → steps down to FOLLOWER
  - Election timeout again → new election with term+1
  
  Split Vote Prevention:
    Randomized election timeout (150-300ms) → unlikely two candidates start simultaneously
```

## 5. Log Replication

```
AppendEntries RPC:

  message AppendEntries {
    term: 5,
    leader_id: "node-1",
    prev_log_index: 4,        // index of entry before new ones
    prev_log_term: 5,         // term of prev_log_index entry
    entries: [                 // new entries to replicate
      { index: 5, term: 5, command: "SET w=4" }
    ],
    leader_commit: 4           // leader's commit index
  }

Follower Processing:
  def handle_append_entries(self, request):
    # Reject if leader's term is old
    if request.term < self.current_term:
      return AppendResponse(success=False)
    
    # Log consistency check:
    # My log at prev_log_index must match prev_log_term
    if self.log[request.prev_log_index].term != request.prev_log_term:
      # Log diverged! Delete conflicting entries
      self.log.truncate(request.prev_log_index)
      return AppendResponse(success=False)
    
    # Append new entries
    self.log.extend(request.entries)
    
    # Update commit index
    if request.leader_commit > self.commit_index:
      self.commit_index = min(request.leader_commit, len(self.log)-1)
      self.apply_committed_entries()
    
    return AppendResponse(success=True)

Commit Rule (Leader):
  Entry is committed when:
    1. Stored on majority of nodes
    2. Entry is from CURRENT term (Raft safety property)
  
  After commit:
    - Leader applies to state machine → responds to client
    - Followers apply when they learn of leader's commit index
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Client Write (goes through leader)
POST /api/v1/consensus/write
{
  "command": "SET",
  "key": "config/max_connections",
  "value": "1000"
}
→ {
  "status": "committed",
  "index": 42,
  "term": 5,
  "leader": "node-1:8080"
}

# Client Read (linearizable)
POST /api/v1/consensus/read
{
  "key": "config/max_connections",
  "consistency": "linearizable"    // linearizable | serializable
}
# Linearizable: leader confirms leadership (heartbeat round) before responding
# Serializable: any node can respond (may be stale)

# Cluster Membership Change
POST /api/v1/consensus/membership
{
  "action": "add",
  "node": {
    "id": "node-6",
    "address": "10.0.0.6:8080"
  }
}
# Uses joint consensus: old_config + new_config → new_config (two-phase)

# Get Cluster Status
GET /api/v1/consensus/status
→ {
  "state": "leader",
  "term": 5,
  "commit_index": 42,
  "last_applied": 42,
  "cluster": [
    {"id": "node-1", "state": "leader", "match_index": 42},
    {"id": "node-2", "state": "follower", "match_index": 42},
    {"id": "node-3", "state": "follower", "match_index": 41},
    {"id": "node-4", "state": "follower", "match_index": 40},
    {"id": "node-5", "state": "follower", "match_index": 0, "status": "down"}
  ]
}
```

### Persistent State & WAL

```
Persistent State (must survive crash):
  ┌──────────────────────────────────────────┐
  │ WAL on disk:                              │
  │   current_term: 5                         │
  │   voted_for: "node-1" (or null)           │
  │   log_entries: [                          │
  │     {index:1, term:1, cmd:"SET x=1"},    │
  │     {index:2, term:1, cmd:"SET y=2"},    │
  │     ...                                   │
  │     {index:42, term:5, cmd:"SET w=4"}    │
  │   ]                                       │
  └──────────────────────────────────────────┘

  Why persistent?
    - current_term: prevents voting twice in same term after crash
    - voted_for: prevents double-voting (split-brain)
    - log: preserves committed entries for replay

Snapshot Compaction:
  Problem: log grows unbounded
  Solution: periodically snapshot state machine + truncate log
  
  Snapshot = { state_machine_state, last_included_index, last_included_term }
  
  After snapshot at index 40:
    - Delete log entries 1-40
    - Keep entries 41-42 (uncommitted or recent)
    - New follower: send snapshot first, then log entries after snapshot
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Write throughput** | Batch entries into single AppendEntries RPC | 100K entries/sec |
| **Read throughput** | Lease-based reads (leader serves without round-trip) | 500K reads/sec |
| **Cluster size** | 3, 5, or 7 nodes (odd for majority). More = slower commits | 7 nodes max typical |
| **State machine size** | Snapshot compaction; snapshot transfer for new nodes | 100GB+ |
| **Learner nodes** | Non-voting read replicas; don't affect commit latency | Unlimited learners |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Leader crash after client ACK** | Entry committed to majority → survives; new leader has it |
| **Leader crash before majority ACK** | Entry NOT committed → may be lost; client retries (idempotent) |
| **Follower crash** | Rejoins, catches up from leader's log; no data lost |
| **All nodes crash** | Persistent WAL; all nodes recover committed entries on restart |
| **Network partition** | Minority partition can't commit; majority partition continues. No conflicts possible |

## 9. Latency

| Operation | LAN (same DC) | WAN (cross-region) |
|-----------|---------------|---------------------|
| Write (commit) | 2-5ms | 50-200ms (RTT-bound) |
| Read (linearizable) | 2-5ms | 50-200ms |
| Read (serializable) | <1ms | <1ms (local) |
| Leader election | 1-3 seconds | 5-15 seconds |

**Optimization:** Batching (amortize per-entry RPC cost), pipelining (don't wait for ACK before sending next batch), lease-based reads (leader serves reads without confirming leadership for lease duration).

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Leader failure** | Writes blocked until election (~3s) | Pre-vote: candidate checks eligibility before disrupting cluster |
| **Network partition** | Minority partition: read-only. Majority: continues | Learner nodes for reads; partition detection via heartbeat |
| **Slow follower** | Commit not blocked (only need majority) | Follower catches up asynchronously; snapshot transfer if too far behind |
| **Split-brain** | IMPOSSIBLE by design (only one leader per term, majority required) | Term numbers + voting rules guarantee single leader |

## 11. Availability

**Target: Tolerates ⌊(N-1)/2⌋ failures**

```
Availability by Cluster Size:
  3 nodes: survive 1 failure   (99.99% with 99.9% per-node)
  5 nodes: survive 2 failures  (99.9999%)
  7 nodes: survive 3 failures  (99.99999%)
  
  Trade-off: more nodes → better availability, slower commits
  
Best Practice:
  - 5 nodes for critical data (etcd, ZooKeeper)
  - 3 nodes for less critical
  - Spread across failure domains (racks, AZs)
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Node authentication** | Mutual TLS between all cluster members |
| **Client authentication** | TLS + API keys or certificates |
| **Authorization** | ACLs on key prefixes (etcd-style) |
| **Encryption in transit** | TLS 1.3 for all RPCs |
| **Encryption at rest** | Encrypted WAL and snapshots |
| **Audit** | Log all state changes with term + index for forensics |

## 13. Cost Constraints

**Estimated Cost (5-node Raft cluster):**

| Component | Cost |
|-----------|------|
| **Nodes** | 5× (8-core, 32GB RAM, NVMe SSD): $2,500/month (cloud) |
| **Network** | Low-latency same-region: included |
| **Storage** | 500GB NVMe per node for WAL + snapshots: included |
| **Total** | **~$2,500/month** |

Consensus is lightweight infrastructure — cost is minimal compared to the data systems it coordinates.

## Key Interview Discussion Points

1. **Raft vs Paxos?** — Raft: easier to understand, single leader, log-structured. Paxos: more flexible (Multi-Paxos), harder to implement correctly. Same safety guarantees. Raft preferred for implementation
2. **Why odd number of nodes?** — Majority of 3 = 2, majority of 4 = 3. 4 nodes gives same fault tolerance as 3 (1 failure) but higher commit cost. Always use 3, 5, or 7
3. **Linearizable reads: why expensive?** — Leader may be deposed without knowing. Must confirm leadership (heartbeat round-trip) before serving read. Alternative: lease-based reads with bounded clock skew
4. **How to add/remove nodes safely?** — Joint consensus (Raft): commit new config through both old and new majority. Prevents two independent majorities during transition
5. **What if the leader is slow, not failed?** — Followers' election timers expire → new election → old leader discovers higher term → steps down. Pre-vote optimization: candidate asks "would you vote for me?" before starting election
