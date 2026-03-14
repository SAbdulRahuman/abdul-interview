# Design a Distributed Storage Networking System

## Overview
Design a multi-protocol storage networking system like NetApp ONTAP Unified Storage — supporting NFS, SMB/CIFS, iSCSI, FC, NVMe-oF, and S3 through a single storage platform. The system provides protocol translation, unified data management, and multi-protocol access to the same data.

## 1. Requirements

**Functional:**
- Multi-protocol support: NFS v3/v4.1/pNFS, SMB 3.1.1/Multichannel, iSCSI, FC 32Gbps, NVMe-oF, S3
- Unified namespace: same data accessible via file (NFS/SMB) and block (iSCSI/FC) protocols
- Protocol translation: file vs block vs object semantics mapping
- Multi-tenancy: per-SVM protocol configuration and isolation
- LIF management: logical interfaces for protocol endpoints
- Session management: NFS state, SMB sessions, iSCSI sessions

**Non-Functional:**
- Wire-speed protocol processing (100Gbps line rate)
- NFS latency: <200μs (read), <500μs (write)
- SMB latency: <300μs (read), <600μs (write)
- iSCSI/NVMe-oF latency: <100μs (read), <200μs (write)
- Concurrent sessions: 100K+ NFS clients, 50K+ SMB sessions

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Total protocols** | 7 (NFS3, NFS4.1, SMB3, iSCSI, FC, NVMe-oF, S3) |
| **Concurrent NFS mounts** | 100,000 |
| **Concurrent SMB sessions** | 50,000 |
| **iSCSI sessions** | 10,000 |
| **Aggregate throughput** | 200 Gbps (all protocols combined) |
| **IOPs** | 2M (mixed 70% read / 30% write) |

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│             Unified Multi-Protocol Storage System                 │
│                                                                   │
│  NFS Clients    SMB Clients    iSCSI Hosts    S3 Clients         │
│  (Linux/Unix)   (Windows)     (VMware/Linux)   (Apps)            │
│  ┌────────┐    ┌────────┐    ┌────────┐     ┌────────┐          │
│  │ mount  │    │ \\srv  │    │ iscsid │     │ aws s3 │          │
│  │ nfs:// │    │ \share │    │ target │     │ cli    │          │
│  └───┬────┘    └───┬────┘    └───┬────┘     └───┬────┘          │
│      │             │             │               │                │
│      └─────────────┼─────────────┼───────────────┘                │
│                    ▼             ▼                                 │
│  ┌──────────────────────────────────────────────────┐            │
│  │               Network Layer                       │            │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐ │            │
│  │  │ LIF    │  │ LIF    │  │ LIF    │  │ LIF    │ │            │
│  │  │ NFS    │  │ SMB    │  │ iSCSI  │  │ S3     │ │            │
│  │  │ port   │  │ port   │  │ port   │  │ port   │ │            │
│  │  │ e0a    │  │ e0b    │  │ e0c    │  │ e0d    │ │            │
│  │  └────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘ │            │
│  └───────┼───────────┼───────────┼────────────┼─────┘            │
│          ▼           ▼           ▼            ▼                   │
│  ┌──────────────────────────────────────────────────┐            │
│  │           Protocol Processing Layer               │            │
│  │                                                   │            │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐    │            │
│  │  │ NFS Server│  │ SMB Server│  │ iSCSI     │    │            │
│  │  │ NFSv3/v4  │  │ SMB 3.1.1 │  │ Target    │    │            │
│  │  │ pNFS      │  │ Multipath │  │ NVMe-oF   │    │            │
│  │  │ Kerberos  │  │ Kerberos  │  │ CHAP      │    │            │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘    │            │
│  │        │              │               │           │            │
│  │        └──────────────┼───────────────┘           │            │
│  │                       ▼                           │            │
│  │  ┌────────────────────────────────────────┐      │            │
│  │  │  Unified Storage Layer (WAFL / VFS)    │      │            │
│  │  │                                        │      │            │
│  │  │  File semantics: inodes, directories   │      │            │
│  │  │  Block semantics: LUNs (files-as-LUNs) │      │            │
│  │  │  Object semantics: buckets, keys       │      │            │
│  │  │                                        │      │            │
│  │  │  Permission mapping:                   │      │            │
│  │  │    UNIX ↔ NTFS ACL ↔ S3 IAM policy    │      │            │
│  │  └────────────────────────────────────────┘      │            │
│  └──────────────────────────────────────────────────┘            │
│                         │                                        │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────┐            │
│  │              RAID / Storage Layer                 │            │
│  │         SSD / HDD / Cloud Tier                    │            │
│  └──────────────────────────────────────────────────┘            │
└──────────────────────────────────────────────────────────────────┘
```

## 4. Protocol Deep Dive

```
NFS Architecture:
═════════════════

  NFSv3 (Stateless):
    Client → RPC over TCP/UDP port 2049
    Operations: LOOKUP, READ, WRITE, CREATE, REMOVE, GETATTR
    Stateless: each request self-contained
    Locking: separate NLM (Network Lock Manager) protocol
    
  NFSv4.1 (Stateful, Session-based):
    Client → compound RPCs over TCP port 2049
    Session: CREATE_SESSION → SEQUENCE → operations
    Delegations: server delegates file to client (local caching)
    pNFS: parallel NFS (client reads data directly from storage nodes)
    
  pNFS Data Layout:
    ┌─────────────────────────────────────────────────┐
    │ Client → MDS (metadata server): LAYOUTGET       │
    │ MDS returns: file layout (which blocks on which │
    │              data servers)                       │
    │ Client → DS1: READ block 0-99                   │
    │ Client → DS2: READ block 100-199                │
    │ Client → DS3: READ block 200-299                │
    │ (parallel I/O, bypasses MDS bottleneck)          │
    │                                                  │
    │ Throughput: N × single-server throughput         │
    └─────────────────────────────────────────────────┘

SMB Architecture:
═════════════════

  SMB 3.1.1 Features:
    - Multichannel: multiple TCP connections for throughput
    - SMB Direct (RDMA): zero-copy I/O over RoCEv2
    - Encryption: AES-128-GCM / AES-256-GCM per session
    - Continuous Availability (CA): transparent failover
    - Witness protocol: cluster-aware client reconnection
    
  SMB Multichannel:
    Client NIC 1 ──TCP──► Server NIC 1 (e0a) ─┐
    Client NIC 2 ──TCP──► Server NIC 2 (e0b) ──┤── Aggregate
    Client NIC 3 ──TCP──► Server NIC 3 (e0c) ──┤   bandwidth
    Client NIC 4 ──TCP──► Server NIC 4 (e0d) ─┘   = 4× single NIC

iSCSI Architecture:
═══════════════════

  iSCSI PDU (Protocol Data Unit):
    ┌──────────────────────────────────────────────┐
    │ TCP Header                                    │
    │ ┌────────────────────────────────────────────┐│
    │ │ iSCSI BHS (Basic Header Segment) - 48 bytes││
    │ │  Opcode: SCSI_CMD / SCSI_RESP / DATA_OUT  ││
    │ │  LUN, CmdSN, ExpStatSN                    ││
    │ │  CDB (SCSI Command Descriptor Block)      ││
    │ ├────────────────────────────────────────────┤│
    │ │ Additional Header Segments                 ││
    │ ├────────────────────────────────────────────┤│
    │ │ Data Segment (payload)                     ││
    │ └────────────────────────────────────────────┘│
    └──────────────────────────────────────────────┘
    
  MPIO (Multipath I/O):
    Host → Path 1 (portal group 1) → Target
    Host → Path 2 (portal group 2) → Target
    Policy: round-robin / least-queue-depth / failover-only

NVMe-oF Architecture:
═════════════════════

  NVMe over Fabrics: native NVMe command set over RDMA/TCP/FC
  
  Advantages over iSCSI:
    - No SCSI translation overhead
    - Multi-queue: up to 64K queues × 64K entries each
    - Lower latency: ~20μs vs ~100μs (iSCSI)
    - Higher IOPs: 1M+ per port
    
  Transport options:
    NVMe/RDMA (RoCEv2): lowest latency, needs lossless fabric
    NVMe/TCP: runs on standard Ethernet, easier deployment
    NVMe/FC: runs on existing FC infrastructure
```

## 5. Unified Permission Model

```
Multi-Protocol Access to Same Data:
════════════════════════════════════

  Problem: NFS uses UNIX permissions, SMB uses NTFS ACLs
  Solution: Unified security style per volume/qtree

  Security Styles:
  ┌──────────┬─────────────────────────────────────────┐
  │ Style    │ Behavior                                │
  ├──────────┼─────────────────────────────────────────┤
  │ UNIX     │ UNIX perms authoritative                │
  │          │ SMB access: map Windows user→UNIX uid   │
  │          │ via name-mapping rules                  │
  ├──────────┼─────────────────────────────────────────┤
  │ NTFS     │ NTFS ACLs authoritative                 │
  │          │ NFS access: map UNIX uid→Windows SID    │
  │          │ via name-mapping or LDAP                │
  ├──────────┼─────────────────────────────────────────┤
  │ Mixed    │ Last writer wins (per-file decision)    │
  │          │ If set via NFS → UNIX perms stored      │
  │          │ If set via SMB → NTFS ACLs stored       │
  └──────────┴─────────────────────────────────────────┘

  Name Mapping:
    UNIX uid 1001 (jdoe) ←→ CORP\jdoe (Windows SID)
    Configured via:
      1. Local name-mapping table
      2. LDAP (unified identity source)
      3. Rules: pattern match (CORP\(*) → \1)
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# LIF (Logical Interface) Management
POST /api/v1/network/lifs
{
  "name": "nfs-data-lif1",
  "svm": "svm-prod",
  "ip": "10.0.1.100",
  "netmask": "255.255.255.0",
  "home_port": "e0a",
  "home_node": "node-1",
  "protocols": ["nfs"],
  "failover_policy": "sfo-partner-only",
  "auto_revert": true
}

# NFS Export Policy
POST /api/v1/protocols/nfs/export-policies
{
  "name": "prod-export",
  "svm": "svm-prod",
  "rules": [
    {
      "index": 1,
      "match": "10.0.1.0/24",                // client subnet
      "protocol": ["nfs3", "nfs4"],
      "security": ["krb5p"],                  // Kerberos privacy
      "rwrule": "krb5p",
      "superuser": "krb5p",
      "anon": 65534                           // nobody
    },
    {
      "index": 2,
      "match": "0.0.0.0/0",
      "protocol": ["nfs3"],
      "security": ["sys"],
      "rwrule": "never",                      // read-only for others
      "rorule": "sys"
    }
  ]
}

# SMB Share
POST /api/v1/protocols/cifs/shares
{
  "name": "engineering",
  "svm": "svm-prod",
  "path": "/vol/eng_data",
  "acl": [
    {"user": "CORP\\eng-team", "permission": "full_control"},
    {"user": "Everyone", "permission": "read"}
  ],
  "properties": ["continuously_available", "encrypt_data"],
  "oplocks": true,
  "access_based_enumeration": true
}

# iSCSI Target
POST /api/v1/protocols/san/iscsi/services
{
  "svm": "svm-prod",
  "target_name": "iqn.1992-08.com.netapp:sn.abc123:vs.svm-prod"
}

POST /api/v1/protocols/san/luns
{
  "svm": "svm-prod",
  "path": "/vol/lun_vol/lun0",
  "size": "1TB",
  "os_type": "linux",
  "space_allocation": true,            // thin provisioning
  "space_reserve": false
}

POST /api/v1/protocols/san/igroups
{
  "name": "esx-cluster-01",
  "protocol": "iscsi",
  "os_type": "vmware",
  "initiators": [
    "iqn.1998-01.com.vmware:esx01:abc123",
    "iqn.1998-01.com.vmware:esx02:def456"
  ]
}
# Map LUN to igroup
POST /api/v1/protocols/san/lun-maps
{
  "lun": "/vol/lun_vol/lun0",
  "igroup": "esx-cluster-01",
  "lun_id": 0
}

# NVMe-oF Subsystem
POST /api/v1/protocols/nvme/subsystems
{
  "name": "nvme-subsys-01",
  "svm": "svm-prod",
  "os_type": "linux",
  "hosts": [
    {"nqn": "nqn.2014-08.org.nvmexpress:host01:uuid:abc123"}
  ]
}

# S3 Bucket
POST /api/v1/protocols/s3/buckets
{
  "name": "data-lake-bucket",
  "svm": "svm-s3",
  "size": "10TB",
  "versioning": "enabled",
  "policy": {
    "statements": [
      {
        "effect": "allow",
        "principal": "arn:aws:iam::12345:user/analyst",
        "action": ["s3:GetObject", "s3:PutObject"],
        "resource": "arn:aws:s3:::data-lake-bucket/*"
      }
    ]
  }
}
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **NFS connections** | Stateless (v3) + session pooling (v4.1) | 100K+ mounts |
| **SMB sessions** | Per-SVM session tables; multichannel for throughput | 50K+ sessions |
| **iSCSI sessions** | Multi-portal groups; load balance across LIFs | 10K+ sessions |
| **Throughput** | LIF distribution across ports; LACP LAG | 200 Gbps aggregate |
| **pNFS / parallel NFS** | Data servers in parallel; MDS for metadata | Linear scale-out |

## 8. No Data Loss

| Scenario | Protection |
|----------|------------|
| **NFS client crash during write** | NFS COMMIT ensures data flushed to NVRAM before ACK |
| **SMB session disconnect** | Continuous Availability: durable handles survive failover |
| **iSCSI path failure** | MPIO: I/O retried on alternate path; no data lost |
| **LIF migration** | LIF moves to healthy port; in-flight I/O retried by protocol stack |
| **Controller failover** | SFO: partner takes over SVMs; NFS/SMB/iSCSI sessions preserved |

## 9. Latency

| Protocol | Read Latency | Write Latency | Notes |
|----------|-------------|---------------|-------|
| **NFS v3** | 150-200μs | 400-600μs | Stateless; commit required |
| **NFS v4.1** | 150-250μs | 400-600μs | Session overhead minimal |
| **SMB 3.1.1** | 200-300μs | 500-700μs | Encryption adds ~50μs |
| **SMB Direct** | 50-100μs | 100-200μs | RDMA: zero-copy |
| **iSCSI** | 100-200μs | 200-400μs | TCP overhead |
| **NVMe/RDMA** | 20-50μs | 50-100μs | Native NVMe + RDMA |
| **NVMe/TCP** | 50-100μs | 100-200μs | Lower than iSCSI |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **NIC failure** | LIF failover to different port (<10s) | LIF failover groups; auto-revert on repair |
| **Switch failure** | Dual-homed: traffic moves to alternate switch | LACP/vPC; dual fabric |
| **Protocol server crash** | Session interrupted | HA: partner takes over protocol sessions |
| **Kerberos KDC down** | New sessions fail; existing sessions continue | Cached tickets; multiple KDCs |
| **DNS failure** | Name resolution fails | Local host resolution; DNS caching; multiple DNS servers |

## 11. Availability

**Target: 99.99% (< 53 minutes downtime per year)**

- LIF failover: <10 seconds (client retries transparently)
- Controller failover: <60 seconds (SFO)
- SMB Continuous Availability: zero visible downtime for CA shares
- NFS: clients retry (v3 is stateless; v4.1 has session recovery)
- iSCSI MPIO: path failover <30 seconds

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **NFS authentication** | AUTH_SYS, Kerberos 5 (krb5/krb5i/krb5p) |
| **SMB authentication** | NTLM, Kerberos; AES-256-GCM encryption |
| **iSCSI authentication** | CHAP (mutual); IPsec optional |
| **S3 authentication** | HMAC-SHA256 signed requests (AWS SigV4) |
| **Data in transit** | TLS 1.3 (NFS over TLS); SMB 3 encryption; IPsec |
| **Data at rest** | NVE / NAE with KMIP key management |
| **Multi-tenancy** | Per-SVM isolation; separate network namespaces |

## 13. Cost Constraints

**Estimated Cost (multi-protocol platform, 100TB):**

| Component | Cost |
|-----------|------|
| **HA pair (100TB all-flash)** | $400,000 |
| **Network (8× 100GbE ports)** | $30,000 |
| **Protocol licenses (NFS, SMB, iSCSI, S3)** | $50,000/year |
| **FC switches (if FC needed)** | $40,000 |
| **Total Year 1** | **~$520,000** |
| **Per-protocol cost** | ~$130,000/protocol |

Unified storage saves ~40% vs deploying separate NFS filer + SAN + object store.

## Key Interview Discussion Points

1. **Why unified vs separate?** — Single data copy accessible via multiple protocols eliminates data silos, reduces storage cost, simplifies management. Trade-off: increased complexity in permission mapping
2. **NFS vs SMB performance?** — NFS is typically lower latency (simpler protocol, less metadata). SMB has higher overhead but SMB Direct (RDMA) closes the gap. For mixed Linux+Windows environments, unified storage avoids data duplication
3. **iSCSI vs FC vs NVMe-oF?** — FC: proven, lossless, dedicated fabric (expensive). iSCSI: IP-based, cheaper, good enough for most workloads. NVMe-oF: lowest latency, best for all-flash; NVMe/TCP makes it accessible without special HBAs
4. **Multi-protocol locking?** — NFS uses NLM (v3) or state-based locks (v4). SMB uses oplocks/leases. Cross-protocol: advisory locks coordinated by storage OS; byte-range locks mapped between protocols
5. **LIF migration and non-disruptive ops?** — LIFs can migrate between ports/nodes for maintenance. NFS clients retry; SMB CA handles keep sessions alive. iSCSI MPIO switches paths. Result: zero downtime for planned operations
