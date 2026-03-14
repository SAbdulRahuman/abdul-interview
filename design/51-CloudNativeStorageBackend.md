# Design a Cloud-Native Storage Backend

## Overview
Design a cloud-native storage backend like NetApp Trident / Kubernetes CSI Driver — providing dynamic persistent volume provisioning, storage orchestration, and data management for containerized workloads running on Kubernetes. The system bridges enterprise storage capabilities (snapshots, clones, replication, QoS) into the Kubernetes-native API model.

## 1. Requirements

**Functional:**
- CSI (Container Storage Interface) driver for Kubernetes
- Dynamic provisioning: create PVs on-demand from StorageClasses
- Backend support: ONTAP NAS/SAN, SolidFire, Cloud Volumes, E-Series
- Snapshot and clone operations via Kubernetes VolumeSnapshot API
- Volume import: adopt existing storage volumes into Kubernetes
- Storage topology awareness: zone/node affinity for data locality
- Resize: online volume expansion without pod restart

**Non-Functional:**
- Provisioning latency: <5 seconds (NAS), <15 seconds (SAN/LUN)
- Volume attach: <10 seconds
- Support 10,000+ PVs per cluster
- Non-disruptive backend upgrades
- Multi-cluster support with single backend

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Kubernetes clusters** | 50 (across dev/staging/prod) |
| **Pods per cluster** | 5,000 |
| **PVs per cluster** | 10,000 |
| **Total managed storage** | 500TB across all clusters |
| **Provisioning rate** | 100 PVs/minute (peak, CI/CD pipelines) |
| **StorageClasses** | 20 (different tiers, protocols, QoS) |

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│           Cloud-Native Storage Backend (Trident-like)            │
│                                                                   │
│  Kubernetes Cluster                                              │
│  ┌────────────────────────────────────────────────────────┐      │
│  │                                                        │      │
│  │  Application Pod                                       │      │
│  │  ┌────────────────────────────────────┐               │      │
│  │  │  Container                          │               │      │
│  │  │  mount: /data → PVC "my-data"      │               │      │
│  │  └────────────┬───────────────────────┘               │      │
│  │               │                                        │      │
│  │  PVC: "my-data" (PersistentVolumeClaim)               │      │
│  │    storageClassName: "ontap-gold"                      │      │
│  │    resources: 100Gi                                    │      │
│  │    accessModes: ReadWriteMany                         │      │
│  │               │                                        │      │
│  │               ▼                                        │      │
│  │  PV: "pvc-abc-123" (PersistentVolume)                 │      │
│  │    CSI driver: csi.trident.netapp.io                  │      │
│  │    volumeHandle: trident_pvc_abc_123                   │      │
│  │               │                                        │      │
│  │               ▼                                        │      │
│  │  ┌────────────────────────────────────────┐           │      │
│  │  │     CSI Driver (Trident)               │           │      │
│  │  │                                        │           │      │
│  │  │  ┌────────────┐  ┌─────────────────┐  │           │      │
│  │  │  │ Controller │  │ Node Plugin     │  │           │      │
│  │  │  │ Plugin     │  │ (DaemonSet)     │  │           │      │
│  │  │  │ (Deploy)   │  │                 │  │           │      │
│  │  │  │            │  │ - Mount/unmount │  │           │      │
│  │  │  │ - Create   │  │ - Format        │  │           │      │
│  │  │  │ - Delete   │  │ - Resize FS     │  │           │      │
│  │  │  │ - Snapshot  │  │ - Stage/publish │  │           │      │
│  │  │  │ - Clone    │  │ - iSCSI login   │  │           │      │
│  │  │  │ - Expand   │  │ - NFS mount     │  │           │      │
│  │  │  └──────┬─────┘  └─────────────────┘  │           │      │
│  │  │         │                               │           │      │
│  │  │         ▼                               │           │      │
│  │  │  ┌───────────────────────────────────┐ │           │      │
│  │  │  │  Backend Manager                  │ │           │      │
│  │  │  │                                   │ │           │      │
│  │  │  │  Backend: ontap-nas-prod           │ │           │      │
│  │  │  │    → ONTAP SVM (NFS volumes)      │ │           │      │
│  │  │  │  Backend: ontap-san-perf           │ │           │      │
│  │  │  │    → ONTAP SVM (iSCSI LUNs)      │ │           │      │
│  │  │  │  Backend: azure-netapp-files       │ │           │      │
│  │  │  │    → ANF (cloud volumes)          │ │           │      │
│  │  │  └──────────┬────────────────────────┘ │           │      │
│  │  └─────────────┼─────────────────────────┘           │      │
│  │                │                                      │      │
│  └────────────────┼──────────────────────────────────────┘      │
│                   │ REST API / ZAPI                              │
│                   ▼                                              │
│  ┌────────────────────────────────────────────────────────┐      │
│  │          Storage Backend                               │      │
│  │                                                        │      │
│  │  ONTAP Cluster / ANF / Cloud Volumes ONTAP            │      │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │      │
│  │  │ FlexVol  │  │ FlexVol  │  │ FlexVol  │  ...      │      │
│  │  │ (NFS PV) │  │ (iSCSI)  │  │ (NFS PV) │           │      │
│  │  │ trident_ │  │ trident_ │  │ trident_ │           │      │
│  │  │ pvc_abc  │  │ pvc_def  │  │ pvc_ghi  │           │      │
│  │  └──────────┘  └──────────┘  └──────────┘           │      │
│  └────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

## 4. CSI Driver Architecture

```
CSI (Container Storage Interface) Specification:
═════════════════════════════════════════════════

  CSI defines gRPC services between Kubernetes and storage driver:

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  Identity Service (both controller + node):              │
  │    GetPluginInfo() → driver name, version                │
  │    GetPluginCapabilities() → what driver supports        │
  │    Probe() → health check                                │
  │                                                          │
  │  Controller Service (centralized, 1 instance):           │
  │    CreateVolume()     → provisions on backend            │
  │    DeleteVolume()     → removes from backend             │
  │    CreateSnapshot()   → snapshot on backend              │
  │    DeleteSnapshot()   → removes snapshot                 │
  │    ControllerPublish()→ attach volume to node (SAN)      │
  │    ControllerUnpublish() → detach                        │
  │    ValidateCapability()→ can volume do RWX?              │
  │    ControllerExpand() → resize volume on backend         │
  │                                                          │
  │  Node Service (per-node, DaemonSet):                     │
  │    NodeStage()    → mount to global staging dir          │
  │    NodeUnstage()  → unmount from staging                 │
  │    NodePublish()  → bind-mount into pod                  │
  │    NodeUnpublish()→ unmount from pod                     │
  │    NodeExpand()   → resize filesystem                    │
  │    NodeGetInfo()  → node topology labels                 │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  Provisioning Flow (NFS):
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  1. User creates PVC:                                       │
  │     kind: PersistentVolumeClaim                             │
  │     spec:                                                   │
  │       storageClassName: ontap-gold                          │
  │       resources: {requests: {storage: 100Gi}}               │
  │       accessModes: [ReadWriteMany]                          │
  │                                                             │
  │  2. K8s PV Controller → CSI Controller Plugin:              │
  │     CreateVolume(name, capacity, parameters, topology)      │
  │                                                             │
  │  3. Trident Controller:                                     │
  │     a. Select backend matching StorageClass requirements    │
  │     b. ONTAP API: create FlexVol "trident_pvc_abc_123"     │
  │        - Size: 100GB                                        │
  │        - Thin provisioned                                   │
  │        - Dedup + compression enabled                        │
  │        - QoS: gold tier (10K IOPs max)                     │
  │        - Export policy: allow K8s node IPs                  │
  │     c. Return volume_id + NFS mount path                    │
  │                                                             │
  │  4. K8s creates PV object bound to PVC                      │
  │                                                             │
  │  5. Pod scheduled on node → CSI Node Plugin:                │
  │     NodeStage: mount -t nfs svm-prod:/trident_pvc_abc_123  │
  │                /var/lib/kubelet/plugins/.../globalmount     │
  │     NodePublish: bind-mount → /var/lib/kubelet/pods/.../    │
  │                  volumes/csi/pvc-abc-123/mount              │
  │                                                             │
  │  6. Pod sees volume at /data                                │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  Provisioning Flow (iSCSI / SAN):
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  1-2. Same PVC creation + CreateVolume call                 │
  │                                                             │
  │  3. Trident Controller:                                     │
  │     a. ONTAP: create LUN in volume                          │
  │     b. ONTAP: create igroup with node's IQN                │
  │     c. ONTAP: map LUN to igroup                             │
  │     d. Return volume_id + target portal + LUN ID            │
  │                                                             │
  │  4. ControllerPublish:                                      │
  │     Ensure LUN mapped to correct node's igroup              │
  │                                                             │
  │  5. NodeStage:                                              │
  │     a. iscsiadm -m discovery -t sendtargets                 │
  │     b. iscsiadm -m node --login                             │
  │     c. Multipath: detect all paths to LUN                   │
  │     d. mkfs.ext4 /dev/dm-X (if not formatted)              │
  │     e. mount /dev/dm-X /staging/path                        │
  │                                                             │
  │  6. NodePublish: bind-mount into pod                        │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

## 5. StorageClass & Backend Configuration

```
StorageClass Examples:
══════════════════════

# Gold tier: all-flash, snapshots, QoS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontap-gold
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-nas"
  storagePools: "ontap-aff/aggr1"
  snapshotPolicy: "default"
  snapshotReserve: "10"
  encryption: "true"
  unixPermissions: "0777"
  qosPolicy: "gold"              # max 50K IOPs
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer

# Economy tier: HDD-backed, no QoS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontap-economy
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-nas-economy"   # qtree-based (shared FlexVol)
  storagePools: "ontap-fas/aggr_hdd"
  snapshotPolicy: "none"
reclaimPolicy: Delete

# SAN (block) tier for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontap-san-db
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-san"
  storagePools: "ontap-aff/aggr_nvme"
  spaceAllocation: "true"
  fsType: "xfs"

Backend Configuration (TridentBackendConfig):
  apiVersion: trident.netapp.io/v1
  kind: TridentBackendConfig
  metadata:
    name: backend-ontap-nas-prod
  spec:
    version: 1
    storageDriverName: ontap-nas
    managementLIF: 10.0.0.10
    dataLIF: 10.0.1.10
    svm: svm-k8s
    credentials:
      name: ontap-creds       # K8s Secret
    storage:
      - labels:
          performance: premium
        defaults:
          spaceReserve: none
          encryption: "true"
          snapshotPolicy: default
          qosPolicy: gold
      - labels:
          performance: standard
        defaults:
          qosPolicy: silver
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Trident CRD: TridentBackendConfig
GET /apis/trident.netapp.io/v1/tridentbackendconfigs
→ {
  "items": [
    {
      "metadata": {"name": "backend-nas-prod"},
      "spec": {
        "storageDriverName": "ontap-nas",
        "managementLIF": "10.0.0.10",
        "svm": "svm-k8s",
        "status": "online",
        "volumes": 1234
      }
    }
  ]
}

# Volume Snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot-daily
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: postgres-data

# Creates ONTAP snapshot of the underlying FlexVol
# Near-instant, space-efficient (COW)

# Clone from Snapshot (Trident creates FlexClone)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-clone
spec:
  storageClassName: ontap-gold
  dataSource:
    name: db-snapshot-daily
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi

# Result: instant FlexClone on ONTAP (zero copy, COW)
# Perfect for dev/test environments

# Volume Import (adopt existing volume)
kubectl create -f -<<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: imported-legacy-data
  annotations:
    trident.netapp.io/importOriginalName: "legacy_vol_data01"
    trident.netapp.io/importBackend: "backend-nas-prod"
spec:
  storageClassName: ontap-gold
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 500Gi
EOF
# Trident renames volume to trident_pvc_xxx, creates PV+PVC
```

### Internal State Machine

```
Volume Lifecycle State Machine:

   PVC Created
       │
       ▼
   ┌──────────┐
   │ Pending  │  Waiting for StorageClass match + backend selection
   └────┬─────┘
        │ CSI CreateVolume called
        ▼
   ┌──────────┐
   │Creating  │  ONTAP API: create FlexVol/LUN
   └────┬─────┘
        │ Volume created on backend
        ▼
   ┌──────────┐
   │ Bound    │  PV created, bound to PVC
   └────┬─────┘
        │ Pod scheduled
        ▼
   ┌──────────┐
   │ Staged   │  NodeStage: NFS mount / iSCSI login
   └────┬─────┘
        │ NodePublish
        ▼
   ┌──────────┐
   │ Published│  Bind-mounted into pod; pod I/O active
   └────┬─────┘
        │ Pod deleted / rescheduled
        ▼
   ┌──────────┐
   │Unpublish │  Unmount from pod
   └────┬─────┘
        │ 
        ▼
   ┌──────────┐
   │ Unstage  │  NFS unmount / iSCSI logout
   └────┬─────┘
        │ PVC deleted
        ▼
   ┌──────────────┐
   │ Reclaim      │  Based on reclaimPolicy:
   │ Delete/Retain│    Delete: remove from backend
   └──────────────┘    Retain: keep for manual cleanup

Trident Orchestrator:
  class TridentOrchestrator:
    def create_volume(self, pvc):
      # 1. Find matching backends
      backends = self.match_backends(pvc.storage_class)
      
      # 2. Select best backend (round-robin, least-used, topology)
      backend = self.select_backend(backends, pvc.topology)
      
      # 3. Create volume on storage
      vol = backend.create_volume(
        name=f"trident_pvc_{pvc.uid}",
        size=pvc.requested_size,
        options=pvc.storage_class.parameters
      )
      
      # 4. Store volume metadata in CRD
      self.store_volume_record(pvc.uid, vol, backend)
      
      # 5. Return CSI volume info
      return CSIVolume(
        volume_id=vol.id,
        access_info=vol.nfs_path or vol.iscsi_target,
        topology=vol.node_affinity
      )
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **PVs per cluster** | Backend volume limits; pool selection | 10,000+ PVs |
| **Provisioning rate** | Parallel ONTAP API calls; connection pooling | 100 PVs/minute |
| **Backends** | Multiple backends per StorageClass; load balancing | 50+ backends |
| **Multi-cluster** | Shared ONTAP backend; per-cluster Trident instances | 50+ K8s clusters |
| **Volume size** | FlexVol up to 300TB; FlexGroup for larger | Petabytes |
| **Economy mode** | qtree-based: 1 FlexVol → many PVs (share overhead) | 5000 qtrees/vol |

## 8. No Data Loss

| Scenario | Protection |
|----------|------------|
| **Pod crash** | PV persists independently; new pod mounts same PV |
| **Node failure** | PV reattaches to rescheduled pod on new node |
| **Backend snapshot** | VolumeSnapshot → ONTAP snapshot; instant, consistent |
| **Accidental PV delete** | ReclaimPolicy: Retain; PV survives PVC deletion |
| **Backend failure** | ONTAP HA/MetroCluster; Trident retries on failover |
| **Kubernetes cluster destroy** | Storage volumes persist on backend; can be re-imported |

## 9. Latency

| Operation | NAS (NFS) | SAN (iSCSI) | Notes |
|-----------|-----------|-------------|-------|
| **Provision (CreateVolume)** | 2-5 seconds | 5-15 seconds | SAN: create vol + LUN + igroup + map |
| **Attach (NodeStage)** | <2 seconds (NFS mount) | 5-10 seconds (iSCSI login) | SAN: discovery + login + multipath |
| **Clone from snapshot** | <2 seconds | <5 seconds | FlexClone: instant (COW) |
| **Volume expand** | <3 seconds + FS resize | <5 seconds + FS resize | Online; no pod restart needed |
| **Steady-state I/O** | 200μs read, 500μs write | 100μs read, 200μs write | Same as bare-metal ONTAP |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Trident controller crash** | No new provisions; existing volumes unaffected | Kubernetes restarts Deployment; PVs already mounted continue |
| **Trident node plugin crash** | No new mounts on that node; existing mounts OK | DaemonSet restarts; kernel mount point persists |
| **ONTAP LIF migration** | Brief NFS retry; iSCSI path switch | NFS: kernel client retries; iSCSI: MPIO alternate path |
| **Backend offline** | New provisions fail; reads/writes fail | Trident queues; ONTAP HA restores; alarms |
| **CRD corruption** | Volume metadata lost | Trident can re-discover volumes from backend; import recovery |

## 11. Availability

**Target: 99.99% (< 53 minutes downtime per year)**

- Trident controller: Kubernetes Deployment with replicas
- Node plugin: DaemonSet (one per node, auto-restart)
- Backend: ONTAP HA pair (99.999%)
- Volume data: persists independently of Trident availability
- Upgrade: rolling update of Trident pods (non-disruptive)

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | ONTAP credentials stored in Kubernetes Secrets (encrypted at rest) |
| **Authorization** | RBAC: StorageClass-level access; namespace PVC quotas |
| **Network** | Backend management via dedicated VLAN; TLS for API calls |
| **Encryption at rest** | ONTAP NVE/NAE; per-volume encryption via StorageClass parameter |
| **Encryption in transit** | NFS over TLS; SMB 3 encryption; iSCSI with IPsec |
| **Pod security** | CSI node plugin runs privileged (mount); controller in non-root where possible |
| **Secrets management** | Integrate with Vault/external secrets operator for ONTAP credentials |

## 13. Cost Constraints

**Estimated Cost (500TB across 50 clusters):**

| Component | Cost |
|-----------|------|
| **ONTAP storage (500TB AFF)** | $2,500,000 |
| **Trident (open source)** | $0 |
| **Cloud alternative: CVO + ANF** | $50,000/month (250TB cloud) |
| **DevOps engineering** | $25,000/year (maintenance) |
| **Total Year 1 (hybrid)** | **~$3,125,000** |

Cost optimization strategies:
- Economy StorageClass (qtree-based): 5000 small PVs per FlexVol
- Thin provisioning + dedup: 2-5× space savings
- FabricPool: cold PV data tiers to S3 ($0.023/GB/month)
- FlexClone for dev/test: zero additional storage for cloned environments

## Key Interview Discussion Points

1. **Why CSI vs in-tree plugins?** — CSI is the Kubernetes standard for storage (in-tree deprecated since K8s 1.23). CSI provides: vendor independence, out-of-tree development cycle, gRPC-based (language agnostic), sidecar pattern (snapshotter, resizer, attacher)
2. **NAS vs SAN for Kubernetes?** — NAS (NFS): simpler (no iSCSI login, no multipath), supports ReadWriteMany (RWX), works across nodes. SAN (iSCSI): lower latency, block-level (better for databases), ReadWriteOnce (RWO) only. Recommendation: NFS for general workloads; iSCSI for databases
3. **Economy driver (qtree-based)?** — Problem: 1 FlexVol per PV wastes metadata overhead for small volumes. Solution: ontap-nas-economy creates qtrees within shared FlexVols. 1 FlexVol = up to 5000 PVs. Trade-off: shared snapshot policy, no per-PV QoS
4. **FlexClone for CI/CD?** — Every PR gets a full database clone via VolumeSnapshot + PVC with dataSource. FlexClone: zero copy, instant creation, only delta writes consume space. 100 clones of 1TB database = ~1TB (not 100TB)
5. **Multi-cluster storage management?** — Each K8s cluster has its own Trident instance pointing to shared ONTAP backend. Namespace isolation via SVM-per-team or export-policy-per-cluster. Astra Control: centralized management for backup/restore/DR across clusters
