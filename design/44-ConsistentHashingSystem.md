# Design a Consistent Hashing System

## Overview
Design a consistent hashing system for data placement in distributed storage — implementing virtual nodes, topology-aware placement (CRUSH algorithm), and minimal data movement during cluster scaling. Used in systems like Ceph, DynamoDB, Cassandra, and storage load balancers.

## 1. Requirements

**Functional:**
- Map data objects to storage nodes deterministically
- Minimal data movement when nodes join/leave (~1/N data moves)
- Topology-aware placement (rack, datacenter failure domains)
- Weighted nodes (larger nodes hold more data)
- Replication: place R replicas on distinct failure domains
- Client-side computation (no central lookup service)

**Non-Functional:**
- Lookup latency: <10μs (in-memory computation)
- Load balance: variance <15% across nodes
- Re-mapping on topology change: <1 second
- Support 10,000+ nodes

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Physical nodes** | 1,000 |
| **Virtual nodes per physical** | 150 |
| **Total virtual nodes** | 150,000 |
| **Objects** | 10 billion |
| **Replication factor** | 3 |
| **Lookups/second** | 1 million |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│           Consistent Hashing Ring                                 │
│                                                                   │
│                    Node A (vn: 50)                                │
│                       ┌──┐                                       │
│               ┌───────┤  ├───────┐                               │
│          Node D       └──┘       Node B                          │
│          (vn: 30)                (vn: 40)                        │
│            ┌──┐                  ┌──┐                            │
│       ┌────┤  ├──┐          ┌───┤  ├────┐                       │
│       │    └──┘  │          │   └──┘    │                       │
│       │          │          │           │                       │
│       │     Hash Ring (0 to 2^64-1)    │                       │
│       │          │          │           │                       │
│       │    ┌──┐  │          │   ┌──┐    │                       │
│       └────┤  ├──┘          └───┤  ├────┘                       │
│            └──┘                  └──┘                            │
│          Node E                  Node C                          │
│          (vn: 30)                (vn: 30)                        │
│               └───────┐  ┌───────┘                               │
│                       └──┘                                       │
│                                                                   │
│  Lookup: hash(key) → position on ring → walk clockwise          │
│          → first virtual node encountered = primary             │
│          → next distinct physical nodes = replicas              │
│                                                                   │
│  Virtual Nodes (VN):                                             │
│    Node A (large): 50 VNs spread around ring                    │
│    Node B (medium): 40 VNs                                      │
│    Node C-E (small): 30 VNs each                                │
│    More VNs = more uniform distribution                          │
└───────────────────────────────────────────────────────────────────┘
```

## 4. Virtual Node Implementation

```
Virtual Node Placement:

class ConsistentHashRing:
  def __init__(self, nodes, replication_factor=3):
    self.ring = SortedDict()  # position → virtual_node
    self.replication_factor = replication_factor
    
    for node in nodes:
      self.add_node(node)
  
  def add_node(self, node):
    # Weight determines number of virtual nodes
    num_vnodes = int(150 * node.weight)  # weight=1.0 → 150 vnodes
    
    for i in range(num_vnodes):
      # Deterministic position for each virtual node
      position = hash(f"{node.id}:{i}")  # SHA-256 → 64-bit
      self.ring[position] = VirtualNode(
        position=position,
        physical_node=node,
        vnode_id=i
      )
  
  def lookup(self, key, count=3):
    """Find N distinct physical nodes for key"""
    position = hash(key)
    
    # Walk clockwise from position
    result = []
    seen_nodes = set()
    
    for vnode in self.ring.items_from(position, wrap=True):
      if vnode.physical_node.id not in seen_nodes:
        result.append(vnode.physical_node)
        seen_nodes.add(vnode.physical_node.id)
        if len(result) == count:
          return result
    
    return result  # fewer than count if not enough nodes
  
  def remove_node(self, node):
    # Remove all virtual nodes for this physical node
    positions_to_remove = [
      pos for pos, vn in self.ring.items()
      if vn.physical_node.id == node.id
    ]
    for pos in positions_to_remove:
      del self.ring[pos]

Data Movement on Node Add:
  Before: 5 nodes, each owns ~20% of ring
  Add Node F: each existing node gives ~4% to F
  
  ┌─────────────────────────────────────────┐
  │ Node  │ Before │ After  │ Movement     │
  │───────┼────────┼────────┼──────────────│
  │ A     │ 20%    │ 16.7%  │ gives 3.3%  │
  │ B     │ 20%    │ 16.7%  │ gives 3.3%  │
  │ C     │ 20%    │ 16.7%  │ gives 3.3%  │
  │ D     │ 20%    │ 16.7%  │ gives 3.3%  │
  │ E     │ 20%    │ 16.7%  │ gives 3.3%  │
  │ F     │ -      │ 16.7%  │ receives all│
  │ Total movement: 16.7% (1/6 = optimal) │
  └─────────────────────────────────────────┘
```

## 5. CRUSH Algorithm (Topology-Aware)

```
CRUSH (Controlled Replication Under Scalable Hashing):

Topology Tree:
  root
  ├── datacenter-east
  │   ├── rack-1
  │   │   ├── node-1 (weight: 10)  [osd.0, osd.1]
  │   │   ├── node-2 (weight: 10)  [osd.2, osd.3]
  │   │   └── node-3 (weight: 5)   [osd.4]
  │   └── rack-2
  │       ├── node-4 (weight: 10)
  │       └── node-5 (weight: 10)
  └── datacenter-west
      ├── rack-3
      │   ├── node-6 (weight: 10)
      │   └── node-7 (weight: 10)
      └── rack-4
          ├── node-8 (weight: 10)
          └── node-9 (weight: 10)

CRUSH Placement Rule (3 replicas across racks):
  rule replicated_cross_rack {
    step take root                          # start at root
    step chooseleaf firstn 3 type rack      # pick 3 different racks
    step emit                               # output selected OSDs
  }

CRUSH Selection Algorithm:
  def crush_select(input_x, bucket, num_replicas, failure_domain):
    selected = []
    for r in range(num_replicas):
      attempt = 0
      while True:
        # Deterministic pseudo-random selection
        candidate = crush_hash(input_x, r, attempt) % bucket.size
        child = bucket.children[candidate]  # weighted selection
        
        # Check constraints
        if child.failure_domain in [s.failure_domain for s in selected]:
          attempt += 1  # same rack, try again
          continue
        if child.is_failed:
          attempt += 1
          continue
        
        selected.append(child)
        break
    
    return selected

Why CRUSH > Simple Consistent Hashing:
  1. Failure domain awareness (replicas on different racks)
  2. Weighted placement (proportional to node capacity)
  3. Deterministic: every client computes same result
  4. Stable: adding rack only moves ~1/N data
  5. No central directory: O(log N) computation
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Initialize Ring
POST /api/v1/ring/initialize
{
  "hash_function": "xxhash64",
  "vnodes_per_node": 150,
  "replication_factor": 3,
  "placement_strategy": "crush"   // ring | crush | jump_hash
}

# Add Node to Ring
POST /api/v1/ring/nodes
{
  "node_id": "node-10",
  "address": "10.0.3.10:9000",
  "weight": 1.5,                  // 1.5× capacity = 1.5× virtual nodes
  "rack": "rack-5",
  "datacenter": "dc-west",
  "tags": ["ssd", "high-iops"]
}

# Lookup Key Placement
GET /api/v1/ring/lookup?key=user:12345&replicas=3
→ {
  "key": "user:12345",
  "hash": "0xABCDEF0123456789",
  "placement": [
    {"node": "node-3", "rack": "rack-1", "role": "primary"},
    {"node": "node-7", "rack": "rack-3", "role": "replica"},
    {"node": "node-5", "rack": "rack-2", "role": "replica"}
  ]
}

# Get Rebalance Plan
POST /api/v1/ring/rebalance
{
  "trigger": "node_added",
  "node_id": "node-10"
}
→ {
  "movements": [
    {"key_range": "0xA000-0xA100", "from": "node-3", "to": "node-10"},
    {"key_range": "0xB200-0xB300", "from": "node-7", "to": "node-10"},
    ...
  ],
  "total_data_to_move_gb": 150,
  "estimated_duration_minutes": 45
}
```

### Hash Function Comparison

```
Hash Function Selection:

┌──────────────────────────────────────────────────────────────┐
│ Function     │ Speed       │ Distribution │ Use Case         │
│──────────────┼─────────────┼──────────────┼──────────────────│
│ xxHash64     │ 15 GB/s     │ Excellent    │ General (fast)   │
│ MurmurHash3  │ 8 GB/s      │ Excellent    │ Cassandra, etc.  │
│ SHA-256      │ 1 GB/s      │ Perfect      │ Security-critical│
│ CityHash     │ 12 GB/s     │ Excellent    │ Google systems   │
│ Jump Hash    │ ~0, O(ln N) │ Perfect      │ Monotonic keys   │
└──────────────────────────────────────────────────────────────┘

Jump Consistent Hash (Google):
  def jump_hash(key, num_buckets):
    """O(ln N) time, perfect distribution, monotonic"""
    b = -1
    j = 0
    while j < num_buckets:
      b = j
      key = (key * 2862933555777941757 + 1) & ((1<<64)-1)
      j = int((b + 1) * (1 << 31) / ((key >> 33) + 1))
    return b
    
  Pros: perfect distribution, O(ln N), minimal movement
  Cons: only supports adding/removing last bucket (not arbitrary)
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Nodes** | Virtual nodes ensure uniform distribution at any scale | 10,000+ nodes |
| **Objects** | Direct computation; no lookup table needed | Billions of objects |
| **Rebalance speed** | Parallel data migration; rate-limited | ~10 GB/s cluster-wide |
| **Ring updates** | Versioned ring; propagate via gossip protocol | <1 second convergence |
| **Weighted scaling** | Adjust virtual node count proportionally | 10× weight range |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **Node failure** | Replicas on other nodes; re-replicate to new node using ring successor |
| **Ring inconsistency** | Versioned ring with monotonic epoch; reject stale ring operations |
| **Hash collision** | 64-bit hash collisions rare (~10^-10 for 10B objects); chain on collision |
| **Rebalance crash** | Idempotent data transfer; resume from checkpoint |

## 9. Latency

| Operation | Latency |
|-----------|---------|
| **Key lookup** | <1μs (in-memory binary search on sorted ring) |
| **CRUSH computation** | <10μs (tree walk, O(log N)) |
| **Ring update propagation** | <1 second (gossip with 1,000 nodes) |
| **Data migration per GB** | ~1 second (10 Gbps network) |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Node failure** | Data on that node temporarily unavailable (primary) | Replicas serve reads; re-replicate from replicas to new node |
| **Ring split (inconsistent views)** | Different clients route to different nodes | Epoch-based ring versioning; reject operations on stale ring |
| **Hot key** | Single virtual node overloaded | Split hot virtual node; application-level caching |
| **Cascading failure** | Multiple node failures overwhelm survivors | Rate-limit rebalancing; prioritize serving traffic over recovery |

## 11. Availability

**Target: Dependent on replication factor**

```
Availability Model:
  R replicas, P(node_available) = 0.999
  
  P(data_available) = 1 - P(all replicas down)
                     = 1 - (1 - 0.999)^R
  
  R=1: 99.9%
  R=2: 99.9999%
  R=3: 99.9999999%
  
  With failure domain awareness:
    R=3 across 3 racks: survives entire rack failure
    R=3 across 3 DCs: survives entire datacenter failure
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Ring integrity** | Signed ring updates; only authorized nodes can modify topology |
| **Node authentication** | mTLS for all node-to-node communication |
| **Key isolation** | Tenant-prefixed keys; ring placement doesn't leak tenant data |
| **Access control** | Node membership requires admin approval |

## 13. Cost Constraints

**Cost impact of consistent hashing: infrastructure savings, not direct cost**

| Aspect | Without Consistent Hashing | With Consistent Hashing |
|--------|----------------------------|------------------------|
| **Node scaling** | Full reshuffling (100% data movement) | ~1/N data movement |
| **Downtime for scaling** | Hours (full rebalance) | Zero (gradual background) |
| **Central directory** | Required (SPOF, bottleneck) | Not needed (client computes) |
| **Load balancing** | Application-managed | Automatic via virtual nodes |

Implementation cost: negligible (library/algorithm). Operational savings: significant (automated scaling/placement).

## Key Interview Discussion Points

1. **Why virtual nodes?** — Without: adding 1 node moves ~50% data from adjacent node only. With 150 vnodes: each node gives ~1/N of data, uniformly distributed. Better load balance
2. **Consistent hashing vs CRUSH?** — Ring: simple, no topology awareness. CRUSH: failure-domain-aware placement, weighted, hierarchical. Use CRUSH for storage systems needing rack/DC-aware replication
3. **How to handle hot partitions?** — Virtual nodes spread each node's ownership. For single hot key: split virtual node, add read replicas, or application-level caching. Ring-level fixes have limits
4. **Data movement during rebalance?** — Rate-limit to avoid saturating network. Transfer in background. Serve reads from old location until transfer complete. Atomic swap of ownership after verification
5. **Jump hash vs ring hash?** — Jump hash: zero memory, perfect balance, O(ln N). But only supports adding/removing nodes at the end. Ring hash: arbitrary add/remove, needs virtual nodes for balance. Choose based on use case
