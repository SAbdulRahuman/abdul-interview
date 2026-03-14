# Design SDN for Storage Networks

## Overview
Design a Software-Defined Networking system for enterprise storage like NetApp ONTAP Network Management — providing VLAN management, LIF (Logical Interface) migration, network QoS, IPspace isolation, broadcast domain management, and failover groups for non-disruptive storage operations.

## 1. Requirements

**Functional:**
- VLAN and broadcast domain management for storage networks
- LIF (Logical Interface) lifecycle: create, migrate, failover, auto-revert
- IPspace isolation: separate network namespaces per tenant/function
- Network QoS: traffic shaping per protocol/tenant
- Failover group management: define which ports a LIF can failover to
- DNS, routing, and firewall policy per SVM
- Service policies: control which services (NFS, SMB, iSCSI, mgmt) run on each LIF

**Non-Functional:**
- LIF migration: <10 seconds, non-disruptive to clients
- Failover: automatic, <30 seconds
- Scale: 1000+ LIFs across 8-node cluster
- Network isolation: zero cross-tenant traffic leakage
- QoS enforcement: wire-speed classification and rate limiting

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Cluster nodes** | 8 (4 HA pairs) |
| **Physical ports** | 8 × 8 = 64 (100GbE) |
| **VLANs** | 50 (data, management, intercluster, etc.) |
| **IPspaces** | 10 (Default, Cluster, per-tenant) |
| **LIFs** | 500+ (data LIFs + mgmt + intercluster) |
| **SVMs** | 50 (multi-tenant) |
| **Aggregate bandwidth** | 6.4 Tbps (64 × 100GbE) |

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│         Software-Defined Storage Network Architecture            │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐      │
│  │                  IPspace: "Default"                     │      │
│  │                                                        │      │
│  │  SVM: svm-prod                    SVM: svm-dev         │      │
│  │  ┌──────────────────────┐        ┌──────────────┐     │      │
│  │  │ LIF: nfs-prod        │        │ LIF: nfs-dev │     │      │
│  │  │   10.0.1.100         │        │  10.0.2.100  │     │      │
│  │  │   VLAN 100           │        │  VLAN 200    │     │      │
│  │  │   Home: node1:e0a    │        │  Home: node2 │     │      │
│  │  │   Current: node1:e0a │        │  :e0a        │     │      │
│  │  ├──────────────────────┤        └──────────────┘     │      │
│  │  │ LIF: smb-prod        │                              │      │
│  │  │   10.0.1.101         │                              │      │
│  │  │   VLAN 100           │                              │      │
│  │  │   Home: node1:e0b    │                              │      │
│  │  ├──────────────────────┤                              │      │
│  │  │ LIF: iscsi-prod      │                              │      │
│  │  │   10.0.3.100         │                              │      │
│  │  │   VLAN 300           │                              │      │
│  │  │   Home: node1:e0c    │                              │      │
│  │  │   (NO failover -     │                              │      │
│  │  │    iSCSI uses MPIO)  │                              │      │
│  │  └──────────────────────┘                              │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐      │
│  │                  IPspace: "Cluster"                     │      │
│  │  (isolated network for cluster interconnect)           │      │
│  │  Cluster LIFs: 192.168.0.x (private, auto-configured) │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐      │
│  │                  IPspace: "Tenant-A"                    │      │
│  │  (completely isolated from Default)                    │      │
│  │  SVM: tenant-a-svm1                                    │      │
│  │    LIF: 172.16.0.100 (overlapping IP OK - diff IPspace)│      │
│  │    Routing table: independent                          │      │
│  │    DNS: tenant-A DNS servers                           │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  Physical Layer:                                                 │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Node 1          Node 2          Node 3          Node 4│      │
│  │ ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───┐│      │
│  │ │e0a e0b e0c │  │e0a e0b e0c │  │e0a e0b e0c │  │...││      │
│  │ │e0d e0e e0f │  │e0d e0e e0f │  │e0d e0e e0f │  │   ││      │
│  │ │e0g e0h     │  │e0g e0h     │  │e0g e0h     │  │   ││      │
│  │ └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─┬─┘│      │
│  │       │               │               │            │   │      │
│  │ ┌─────┴───────────────┴───────────────┴────────────┴─┐│      │
│  │ │            Switch Fabric (Leaf-Spine)               ││      │
│  │ │  ToR Switch 1 ──── Spine 1 ──── ToR Switch 3      ││      │
│  │ │  ToR Switch 2 ──── Spine 2 ──── ToR Switch 4      ││      │
│  │ └────────────────────────────────────────────────────┘│      │
│  └────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

## 4. LIF Lifecycle & Migration

```
LIF (Logical Interface) Architecture:
══════════════════════════════════════

  LIF = Virtual NIC that can move between physical ports/nodes

  LIF Properties:
    ┌──────────────────────────────────────────────┐
    │ LIF: "nfs-data-lif1"                         │
    │                                              │
    │ IP Address:     10.0.1.100/24                │
    │ Home Node:      node-1                       │
    │ Home Port:      e0a-100 (port e0a, VLAN 100) │
    │ Current Node:   node-1 (or wherever migrated)│
    │ Current Port:   e0a-100                      │
    │ SVM:            svm-prod                     │
    │ Service Policy: default-data-files           │
    │                 (allows: nfs, smb, dns, ldap)│
    │ Failover Group: data-failover-group          │
    │ Auto Revert:    true                         │
    │ Status:         up                           │
    └──────────────────────────────────────────────┘

  LIF Migration (Non-Disruptive):
    ┌────────────────────────────────────────────────┐
    │ Before: LIF on node-1:e0a                      │
    │                                                │
    │ Step 1: Admin triggers migration               │
    │   network interface migrate -lif nfs-lif1      │
    │   -dest-node node-2 -dest-port e0a             │
    │                                                │
    │ Step 2: GARP (Gratuitous ARP) sent             │
    │   "10.0.1.100 is now at node-2:e0a MAC"       │
    │   Switch updates MAC table                     │
    │                                                │
    │ Step 3: New packets arrive at node-2           │
    │   NFS clients retry → server responds          │
    │   Total disruption: <10 seconds                │
    │                                                │
    │ Step 4: (if auto-revert enabled)               │
    │   When node-1:e0a comes back healthy           │
    │   LIF auto-migrates home                       │
    │   Another GARP sent                            │
    │                                                │
    │ After: LIF back on node-1:e0a (home)           │
    └────────────────────────────────────────────────┘

  Failover Group:
    ┌────────────────────────────────────────────────┐
    │ Group: "data-failover-group"                   │
    │                                                │
    │ Ports:                                         │
    │   node-1:e0a  (home - preferred)               │
    │   node-1:e0b  (same node, different port)      │
    │   node-2:e0a  (HA partner node)                │
    │   node-2:e0b  (HA partner node)                │
    │                                                │
    │ Failover Order:                                │
    │   1. Same node, different port (fastest)       │
    │   2. HA partner, same port name                │
    │   3. HA partner, different port                │
    │                                                │
    │ NOT included: nodes in different HA pair       │
    │ (data not local → performance degradation)     │
    └────────────────────────────────────────────────┘
```

## 5. IPspace & Broadcast Domain

```
IPspace Architecture:
═════════════════════

  IPspace = isolated network namespace with its own:
    - Routing table
    - DNS configuration
    - NIS configuration
    - Broadcast domains
    - LIFs
    - Ports

  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  IPspace: "Default"                                   │
  │  ┌─────────────────────────────────────────────────┐ │
  │  │ Broadcast Domain: "data-bd"                     │ │
  │  │   Ports: node1:e0a-100, node2:e0a-100, ...     │ │
  │  │   MTU: 9000 (jumbo frames)                      │ │
  │  │   Subnet: 10.0.1.0/24                           │ │
  │  │   Gateway: 10.0.1.1                             │ │
  │  ├─────────────────────────────────────────────────┤ │
  │  │ Broadcast Domain: "mgmt-bd"                     │ │
  │  │   Ports: node1:e0M, node2:e0M, ...             │ │
  │  │   MTU: 1500                                     │ │
  │  │   Subnet: 192.168.1.0/24                        │ │
  │  └─────────────────────────────────────────────────┘ │
  │  Routing: 0.0.0.0/0 → 10.0.1.1                      │
  │  DNS: 10.0.0.53, 10.0.0.54                          │
  │                                                       │
  │  IPspace: "Tenant-A"                                  │
  │  ┌─────────────────────────────────────────────────┐ │
  │  │ Broadcast Domain: "tenant-a-data"               │ │
  │  │   Ports: node3:e0a-400, node4:e0a-400          │ │
  │  │   MTU: 9000                                     │ │
  │  │   Subnet: 172.16.0.0/24 (can overlap!)         │ │
  │  └─────────────────────────────────────────────────┘ │
  │  Routing: independent table                          │
  │  DNS: tenant-A's DNS servers                         │
  │                                                       │
  │  Key: IPspaces provide COMPLETE network isolation    │
  │  - Overlapping IPs allowed between IPspaces          │
  │  - No routing between IPspaces                       │
  │  - Each IPspace = separate virtual router            │
  └──────────────────────────────────────────────────────┘

Network QoS:
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  Per-Interface Rate Limiting:                          │
  │    LIF nfs-prod: max 50 Gbps (data LIF)               │
  │    LIF mgmt:     max  1 Gbps (management only)        │
  │    LIF repl:     max 10 Gbps (replication traffic)     │
  │                                                        │
  │  DSCP Marking:                                         │
  │    NFS/SMB data:       DSCP 0  (best effort)          │
  │    iSCSI:              DSCP 46 (expedited forwarding)  │
  │    Replication:        DSCP 18 (assured forwarding)    │
  │    Management:         DSCP 48 (network control)       │
  │                                                        │
  │  ECN (Explicit Congestion Notification):              │
  │    Enable for RoCEv2/NVMe-RDMA (lossless required)    │
  │    PFC (Priority Flow Control) on storage VLANs       │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# IPspace Management
POST /api/v1/network/ipspaces
{
  "name": "tenant-a",
  "ports": ["node3:e0a", "node4:e0a"]
}

# Broadcast Domain
POST /api/v1/network/broadcast-domains
{
  "name": "tenant-a-data",
  "ipspace": "tenant-a",
  "mtu": 9000,
  "ports": [
    {"node": "node3", "port": "e0a-400"},   // VLAN 400
    {"node": "node4", "port": "e0a-400"}
  ]
}

# VLAN
POST /api/v1/network/vlans
{
  "node": "node3",
  "port": "e0a",
  "vlan_id": 400,
  "broadcast_domain": "tenant-a-data"
}

# LIF
POST /api/v1/network/lifs
{
  "name": "tenant-a-nfs-lif1",
  "svm": "tenant-a-svm",
  "ip": {"address": "172.16.0.100", "netmask": "255.255.255.0"},
  "home_node": "node3",
  "home_port": "e0a-400",
  "service_policy": "default-data-files",
  "failover": {
    "group": "tenant-a-failover",
    "policy": "sfo-partner-only"
  },
  "auto_revert": true,
  "ipspace": "tenant-a"
}

# LIF Migration
POST /api/v1/network/lifs/{lif_id}/migrate
{
  "destination_node": "node4",
  "destination_port": "e0a-400"
}
→ {
  "status": "migrated",
  "from": "node3:e0a-400",
  "to": "node4:e0a-400",
  "migration_time_ms": 2500,
  "garp_sent": true,
  "clients_affected": 0     // transparent to clients
}

# Service Policy
POST /api/v1/network/service-policies
{
  "name": "custom-data-policy",
  "svm": "svm-prod",
  "services": [
    "data-nfs",
    "data-cifs",
    "data-dns-server",
    "data-flexcache"
  ]
  // Excludes: management-ssh, management-https
  // Effect: LIF with this policy cannot be used for SSH/HTTPS
}

# Failover Group
POST /api/v1/network/failover-groups
{
  "name": "tenant-a-failover",
  "ipspace": "tenant-a",
  "targets": [
    {"node": "node3", "port": "e0a-400"},
    {"node": "node3", "port": "e0b-400"},
    {"node": "node4", "port": "e0a-400"},
    {"node": "node4", "port": "e0b-400"}
  ]
}

# Route
POST /api/v1/network/routes
{
  "svm": "tenant-a-svm",
  "destination": "0.0.0.0/0",
  "gateway": "172.16.0.1",
  "metric": 10
}
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **LIFs per cluster** | Hash table indexed by IP; O(1) lookup | 10,000+ LIFs |
| **VLANs per port** | 802.1Q tagging; up to 4094 per port | 4094 VLANs |
| **IPspaces** | Independent routing tables | 100+ IPspaces |
| **Bandwidth** | LACP link aggregation; multi-port bonding | Tbps per node |
| **LIF migration** | GARP-based; parallel migrations supported | <10s per migration |

## 8. No Data Loss

| Scenario | Protection |
|----------|------------|
| **Port failure** | LIF failover to different port in failover group; no I/O lost |
| **Node failure** | LIF migrates to HA partner; NVRAM replayed; zero data loss |
| **Switch failure** | Dual-homed topology; alternate paths available |
| **VLAN misconfiguration** | Broadcast domain health check; alerts on MTU mismatch |
| **Split network** | Mediator-assisted site selection (MetroCluster); no split-brain |

## 9. Latency

| Operation | Latency | Notes |
|-----------|---------|-------|
| **LIF migration** | <10 seconds | GARP propagation + switch MAC table update |
| **LIF failover (port)** | <5 seconds | Same node, different port |
| **LIF failover (node)** | <30 seconds | HA partner takeover |
| **Normal I/O (local LIF)** | 0 overhead | Direct path |
| **I/O via migrated LIF** | +50-100μs | Cross-node forwarding until client re-routes |
| **VLAN tagging** | 0 overhead | Hardware offload |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Physical port failure** | LIF failover <5s | Failover groups; auto-revert |
| **LACP bond member failure** | Reduced bandwidth | Remaining members handle traffic |
| **Switch failure** | Half of dual-homed paths lost | Dual switch fabric; rapid failover |
| **MTU mismatch** | Packet drops (jumbo frames) | Health monitors; broadcast domain checks |
| **IP conflict** | Service disruption | Duplicate address detection (DAD); alerts |

## 11. Availability

**Target: 99.999% (< 5 minutes downtime per year)**

- LIF failover handles port/node failures automatically
- Dual-homed network topology provides path redundancy
- Auto-revert ensures LIFs return home after repair
- IPspace isolation prevents tenant issues from affecting others
- Service policies prevent accidental protocol exposure

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Network isolation** | IPspace: complete L3 isolation per tenant |
| **VLAN isolation** | 802.1Q: L2 segmentation per traffic type |
| **Service policy** | Restrict LIF to specific protocols (no SSH on data LIFs) |
| **Firewall policy** | Per-LIF IP-based ACLs for management access |
| **IPsec** | End-to-end encryption for inter-cluster/replication traffic |
| **MACsec** | L2 encryption on inter-switch links |
| **RBAC** | Network admin role separate from storage admin |

## 13. Cost Constraints

**Estimated Cost (8-node cluster networking):**

| Component | Cost |
|-----------|------|
| **100GbE NICs (64 ports total)** | $128,000 |
| **Leaf-spine switches (4 leaf + 2 spine)** | $180,000 |
| **Optics and cabling** | $30,000 |
| **Dark fiber (for MetroCluster ISL)** | $15,000/month |
| **SDN management (included in ONTAP)** | $0 incremental |
| **Total Year 1** | **~$518,000** |

## Key Interview Discussion Points

1. **Why LIF abstraction?** — Decouples storage identity (IP address) from physical location. Enables non-disruptive operations: upgrade node → migrate LIFs → upgrade → migrate back. Clients see no change
2. **IPspace vs VLAN isolation?** — VLAN provides L2 isolation. IPspace provides L3 isolation with independent routing tables. VLANs share a routing table within an IPspace. For true multi-tenancy with overlapping IPs, IPspaces are required
3. **Why no LIF failover for iSCSI?** — iSCSI LIFs don't failover because MPIO (Multipath I/O) handles path management at the host level. Changing a target IP mid-session would break iSCSI sessions. Instead, MPIO switches to alternate path
4. **GARP-based migration limitations?** — GARP works within a broadcast domain (L2). For L3 migration across subnets, would need DNS update or BGP announcement (not typically used for storage). That's why LIF migration is within broadcast domain only
5. **Storage network best practices?** — Separate VLANs for: data, management, intercluster, cluster. Jumbo frames (MTU 9000) for data VLANs. PFC/ECN for RoCEv2. Dual switches for redundancy. Never mix storage and general-purpose traffic
