# Design a Disaster Recovery System

## Overview
Design an enterprise disaster recovery system like NetApp MetroCluster / SyncMirror — providing zero RPO site-level protection with automated failover, transparent to applications, supporting both metro (synchronous) and long-distance (asynchronous) DR scenarios.

## 1. Requirements

**Functional:**
- Metro DR: synchronous mirroring between sites (RPO=0, RTO<2min)
- Long-distance DR: asynchronous replication (RPO=minutes, RTO<30min)
- Automated failover: detect site failure and switch to DR site
- Automated failback: reverse-replicate and resume at primary site
- Non-disruptive testing: validate DR without breaking replication
- Multi-site DR: primary → metro DR → long-distance vault

**Non-Functional:**
- RPO: 0 (sync), 15 minutes (async)
- RTO: <120 seconds (metro), <30 minutes (async)
- Failover: automated with mediator, or manual
- I/O impact during normal operation: <1ms additional latency
- Zero data loss during planned failover

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Protected data** | 500TB across 2 sites |
| **Site distance (metro)** | 50km (sync feasible) |
| **Site distance (long DR)** | 3,000km (async only) |
| **Daily change rate** | 2% = 10TB |
| **IOPs during normal** | 1M (shared between sites) |
| **DR test frequency** | Monthly |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│           MetroCluster DR Architecture                            │
│                                                                   │
│  Site A (Primary)              Site B (DR - Metro)               │
│  ┌──────────────────┐          ┌──────────────────┐              │
│  │ Controller A1    │          │ Controller B1    │              │
│  │  (Active for     │◄────────►│  (Standby for    │              │
│  │   SVM-prod)      │  Sync    │   SVM-prod)      │              │
│  │                  │  Mirror  │                  │              │
│  │ Controller A2    │          │ Controller B2    │              │
│  │  (Active for     │◄────────►│  (Standby for    │              │
│  │   SVM-dev)       │  Sync    │   SVM-dev)       │              │
│  ├──────────────────┤          ├──────────────────┤              │
│  │ Pool 0 (local)   │          │ Pool 0 (local)   │              │
│  │ Pool 1 (remote ──┼──mirror──┼─ Pool 1 (remote) │              │
│  │   mirrored)      │          │   mirrored)      │              │
│  └──────────────────┘          └──────────────────┘              │
│          │                              │                        │
│     ISL (Inter-Switch Links)            │                        │
│     FC or IP, redundant paths           │                        │
│          │                              │                        │
│  ┌───────┴──────────────────────────────┴──────────┐            │
│  │              SAN Fabric (stretched)              │            │
│  │  FC Switch A1 ◄─────ISL─────► FC Switch B1      │            │
│  │  FC Switch A2 ◄─────ISL─────► FC Switch B2      │            │
│  └──────────────────────────────────────────────────┘            │
│                                                                   │
│  ┌──────────────────────────────────────┐                        │
│  │  Mediator (Tiebreaker) at Site C     │                        │
│  │  Lightweight VM                      │                        │
│  │  - Monitors both sites              │                        │
│  │  - Breaks tie in split-brain        │                        │
│  │  - Triggers AUSO (auto failover)    │                        │
│  └──────────────────────────────────────┘                        │
│                                                                   │
│                    Site C (Vault/Archive)                         │
│  ┌──────────────────────────────────────────┐                    │
│  │  SnapMirror Async from Site A            │                    │
│  │  RPO: 15 min                             │                    │
│  │  SnapLock Vault: immutable copies        │                    │
│  └──────────────────────────────────────────┘                    │
└───────────────────────────────────────────────────────────────────┘
```

## 4. SyncMirror (RAID-Level Mirroring)

```
SyncMirror Architecture:

  Each aggregate has 2 plexes (RAID groups):
  
  Aggregate: aggr1_mirrored
  ┌─────────────────────────────────────────────────┐
  │                                                  │
  │  Plex 0 (Pool 0 - Site A local disks)           │
  │  ┌───────────────────────────────────┐          │
  │  │ RAID Group 1: D1 D2 D3 D4 P1 P2  │ (RAID-DP)│
  │  │ RAID Group 2: D5 D6 D7 D8 P3 P4  │          │
  │  └───────────────────────────────────┘          │
  │                                                  │
  │  Plex 1 (Pool 1 - Site B remote disks)          │
  │  ┌───────────────────────────────────┐          │
  │  │ RAID Group 1: D1'D2'D3'D4'P1'P2' │ (RAID-DP)│
  │  │ RAID Group 2: D5'D6'D7'D8'P3'P4' │          │
  │  └───────────────────────────────────┘          │
  │                                                  │
  │  Write Path:                                     │
  │    Write → NVRAM (local + remote mirror)        │
  │         → Plex 0 (local RAID) simultaneously    │
  │         → Plex 1 (remote RAID via ISL)          │
  │    ACK when BOTH plexes acknowledged             │
  │                                                  │
  │  Read Path:                                      │
  │    Read → Plex 0 (local, preferred) only        │
  │    No cross-site read latency                    │
  │                                                  │
  │  Site A Failure:                                 │
  │    Plex 0 unavailable                            │
  │    Controller B serves from Plex 1              │
  │    Zero data loss (Plex 1 = mirror of Plex 0)   │
  └─────────────────────────────────────────────────┘
```

## 5. Failover & Failback

```
Automated Unattended Switchover (AUSO):

Normal State:
  Site A: serving I/O, mirroring to Site B
  Mediator: monitoring both sites, all healthy

Site A Failure Detection:
  T=0s:    Site A controllers stop heartbeat
  T=10s:   Site B detects loss of inter-site connectivity
  T=15s:   Mediator confirms Site A unreachable
  T=20s:   Mediator triggers AUSO at Site B
  
  AUSO Steps:
  1. Mediator: verify Site A truly down (not just network blip)
  2. Mediator → Site B: "failover approved"
  3. Site B: claim ownership of Site A's SVMs
  4. Site B: force-mount Site A's aggregates from Plex 1
  5. Site B: bring up SVMs (IP addresses move to Site B LIFs)
  6. Site B: start serving I/O from Plex 1 (mirrored data)
  7. Application: reconnect to same IP (now at Site B)
  
  Total RTO: ~120 seconds (mostly detection + validation)

Failback (Site A Restored):

  T=0:     Site A hardware repaired, booted
  T+5min:  Site A contacts Mediator
  T+10min: Resync: Site B → Site A (incremental, only changes during outage)
  T+30min: Resync complete. Aggregates mirrored again.
  T+31min: Planned switchback:
           1. Quiesce (brief I/O pause)
           2. Final sync
           3. Transfer SVM ownership back to Site A
           4. Resume I/O at Site A
           5. Re-establish normal mirroring
  
  Failback downtime: <2 minutes (planned switchback)

Split-Brain Prevention:
  ┌──────────────────────────────────────────────┐
  │ Scenario: ISL down, both sites operational   │
  │                                              │
  │ Without mediator: both sites may claim SVMs  │
  │ → data divergence → DISASTER                 │
  │                                              │
  │ With mediator:                               │
  │   Site A can reach mediator? → stays primary │
  │   Site B can reach mediator? → stays standby │
  │   Neither reaches mediator? → no failover    │
  │   (prefer availability loss over split-brain) │
  └──────────────────────────────────────────────┘
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# MetroCluster Status
GET /api/v1/metrocluster/status
→ {
  "configuration": "ip",           // ip | fc
  "state": "normal",              // normal | switchover | switchback
  "partner_reachable": true,
  "mediator_reachable": true,
  "auso_enabled": true,
  "nodes": [
    {"name": "site-a-01", "state": "normal", "mirroring": "in-sync"},
    {"name": "site-a-02", "state": "normal", "mirroring": "in-sync"},
    {"name": "site-b-01", "state": "waiting", "role": "dr-partner"},
    {"name": "site-b-02", "state": "waiting", "role": "dr-partner"}
  ]
}

# Manual Switchover (planned)
POST /api/v1/metrocluster/switchover
{
  "type": "negotiated",           // negotiated (clean) | forced (emergency)
  "validate_first": true
}

# Switchback (after failover)
POST /api/v1/metrocluster/switchback
{
  "heal_aggregates": true,        // resync Plex 0 first
  "heal_root_aggregates": true,
  "override_vetoes": false
}

# DR Test (non-disruptive)
POST /api/v1/metrocluster/dr-test
{
  "type": "simulation",           // simulation | live_clone
  "svm": "svm-prod",
  "report_only": true
}
→ {
  "test_result": "PASS",
  "rpo_achieved": "0s",
  "rto_estimated": "95s",
  "aggregates_healthy": true,
  "plex_mirroring": "in-sync",
  "mediator_status": "reachable"
}
```

### Mediator Protocol

```
Mediator Tiebreaker Logic:

class DRMediator:
  def monitor(self):
    while True:
      site_a = self.heartbeat(SITE_A)
      site_b = self.heartbeat(SITE_B)
      
      if site_a.alive and site_b.alive:
        # Normal: both sites up
        self.state = NORMAL
        
      elif not site_a.alive and site_b.alive:
        # Site A failed: trigger AUSO
        if self.confirm_site_down(SITE_A, retries=3):
          self.trigger_auso(surviving_site=SITE_B)
          
      elif site_a.alive and not site_b.alive:
        # Site B failed: no action needed (A is primary)
        self.alert("Site B unreachable")
        
      elif not site_a.alive and not site_b.alive:
        # Both unreachable from mediator
        # Do NOT trigger failover (could be mediator network issue)
        self.alert("CRITICAL: both sites unreachable from mediator")
      
      sleep(5)  # heartbeat interval
  
  def trigger_auso(self, surviving_site):
    # Set mailbox: authorize failover
    surviving_site.set_mailbox("auso_approved", True)
    # Surviving site's ONTAP reads mailbox → starts switchover
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Protected data** | SyncMirror per aggregate; MetroCluster per HA pair | 500TB per HA pair |
| **Site distance (sync)** | Metro: up to 300km (IP MetroCluster) | 300km max |
| **Site distance (async)** | SnapMirror async for 3rd site | Unlimited |
| **Nodes per MetroCluster** | 4+4 (8 total, 4 HA pairs) | 8 nodes |
| **ISL bandwidth** | Multiple 100Gbps links | 400Gbps aggregate |

## 8. No Data Loss

| Scenario | RPO | Explanation |
|----------|-----|-------------|
| **Site failure (metro sync)** | 0 | Every write mirrored to both plexes; surviving plex has all data |
| **Planned switchover** | 0 | Negotiated: flush all in-flight I/O before switch |
| **Unplanned switchover (AUSO)** | 0 | NVRAM mirrored; replayed at surviving site |
| **ISL failure** | 0 | I/O pauses until ISL restored (sync mode); or degrades to async |
| **3rd site vault** | 15 min | Async RPO; immutable SnapLock copies for compliance |

## 9. Latency

| Operation | Metro (50km) | Long-Distance (3000km) |
|-----------|-------------|------------------------|
| Write (normal, sync mirror) | +0.5ms overhead | N/A (async only) |
| Read (normal) | No overhead (read from local plex) | No overhead |
| Failover (AUSO) | ~120 seconds total RTO | N/A |
| Failover (manual) | ~60 seconds | ~30 minutes (break + mount) |
| Switchback | ~2 minutes downtime + resync time | ~1-4 hours |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Single site failure** | Transparent failover; <2min RTO | Automated AUSO via mediator |
| **ISL failure (sites up)** | Sync mirror paused; I/O continues locally | Redundant ISLs; auto-resync when ISL restored |
| **Mediator failure** | AUSO disabled; manual failover still works | Redundant mediator; mediator HA |
| **Split-brain** | Potential data divergence | Mediator prevents; if unavoidable: manual resolution with reconciliation |
| **Both sites fail** | Data unavailable | 3rd site vault allows recovery (RPO=15min) |

## 11. Availability

**Target: 99.999% (< 5 minutes downtime per year)**

```
Availability Calculation:
  Single site availability: 99.99%
  MetroCluster (2 sites): 1 - (1-0.9999)^2 = 99.999999%
  
  Minus failover time (~2 min per event, ~1 event/year):
  Effective: ~99.9996% (2 minutes/year)
  
  With 3rd async site: survive regional disaster
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **ISL encryption** | MACsec (Layer 2) or IPsec for MetroCluster IP |
| **Data at rest** | Both sites encrypted (NSE/NVE); same keys via KMIP |
| **Mediator security** | HTTPS with mutual certificate authentication |
| **Access control** | DR operations require cluster admin role |
| **Audit** | All switchover/switchback events logged with timestamp and initiator |
| **Compliance** | 3rd site SnapLock vault for regulatory retention |

## 13. Cost Constraints

**Estimated Cost (500TB, 2 metro sites + 1 vault):**

| Component | Cost |
|-----------|------|
| **Site A infrastructure** | 500TB all-flash HA pair: $1.2M |
| **Site B infrastructure** | 500TB all-flash HA pair: $1.2M |
| **ISL connectivity** | 2× 100Gbps dark fiber (50km): $20,000/month |
| **Mediator (Site C)** | VM: $500/month |
| **3rd site vault storage** | 500TB capacity tier: $300,000 |
| **MetroCluster license** | $100,000/year |
| **Total (Year 1)** | **~$3,146,000** |

Compared to cost of downtime: if 1 hour of downtime = $1M+ revenue impact, MetroCluster pays for itself with a single prevented outage.

## Key Interview Discussion Points

1. **RPO=0 vs RPO=15min cost?** — RPO=0 requires synchronous mirror (doubling storage + dedicated ISL). RPO=15min uses async replication (cheaper, longer distance). Most businesses need RPO=0 only for tier-1 critical systems
2. **How does the mediator prevent split-brain?** — Three-site quorum: failover only if mediator confirms primary is truly down. If mediator unreachable, no automatic failover (manual only). Prefer data safety over automated response
3. **What about stretch SAN vs MetroCluster IP?** — FC MetroCluster: stretched SAN fabric (FC switches at both sites). IP MetroCluster: uses IP network (cheaper, more flexible, up to 300km). IP is the modern approach
4. **Non-disruptive DR testing?** — Clone volumes at DR site; bring up test SVM with different IPs; test applications against clone. No impact on production replication
5. **Multi-tier DR architecture?** — Tier 1 (critical DB): MetroCluster sync RPO=0. Tier 2 (important apps): SnapMirror async RPO=15min. Tier 3 (archive): daily backup to cloud. Cost vs criticality trade-off
