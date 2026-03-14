# Chapter 4 — Storage in Kubernetes

> **Critical chapter for Dell interview** — Dell builds enterprise storage (PowerStore, PowerFlex, PowerScale, Unity) and their CSI drivers. Deep understanding of Kubernetes storage subsystem is essential.

---

## 1. Volume Types Overview

```
┌──────────────────────────────────────────────────────────┐
│                  Kubernetes Volumes                       │
│                                                          │
│  Ephemeral            Persistent             Projected   │
│  ┌──────────┐         ┌──────────────┐      ┌─────────┐│
│  │ emptyDir │         │PersistentVol │      │projected ││
│  │ hostPath │         │ (PV/PVC)     │      │configMap ││
│  │ generic  │         │ CSI-backed   │      │secret    ││
│  │ ephemeral│         │              │      │downward  ││
│  └──────────┘         └──────────────┘      │ API      ││
│                                              │svcAcct   ││
│                                              │ token    ││
│                                              └─────────┘│
└──────────────────────────────────────────────────────────┘
```

### Common Volume Types

| Type | Persistence | Use Case |
|------|------------|----------|
| `emptyDir` | Pod lifetime | Scratch space, caching, inter-container sharing |
| `hostPath` | Node lifetime | Access host files (CSI drivers, device plugins) |
| `configMap` | ConfigMap lifetime | Config files mounted as volumes |
| `secret` | Secret lifetime | Credentials mounted as files |
| `projected` | Combined | Merge multiple sources into one directory |
| `persistentVolumeClaim` | Beyond pod | Database storage, stateful apps |
| `csi` | CSI driver | Enterprise storage (Dell, NetApp, Pure) |

### emptyDir

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 1Gi       # Evicted if exceeded
      medium: ""            # Default: node disk
      # medium: Memory      # tmpfs (RAM-backed, counts against memory limit)
```

### hostPath (Use with caution)

```yaml
volumes:
  - name: dev
    hostPath:
      path: /dev
      type: Directory
      # Types: DirectoryOrCreate, Directory, FileOrCreate, File,
      #        Socket, CharDevice, BlockDevice
```

**Warning**: hostPath bypasses pod isolation. Only use for system components (CSI node plugins, device plugins, monitoring agents).

---

## 2. Persistent Volumes (PV)

Cluster-scoped storage resource provisioned by admin or dynamically via StorageClass.

### PV Lifecycle

```
┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
│ Available │───→│   Bound   │───→│ Released  │───→│  Deleted  │
│           │    │ (to PVC)  │    │ (PVC del) │    │ (reclaim) │
└───────────┘    └───────────┘    └───────────┘    └───────────┘
                                         │
                                         ├── Retain: PV stays, data preserved
                                         ├── Delete: PV + storage deleted
                                         └── Recycle: deprecated (rm -rf /vol/*)
```

### PV YAML (Static Provisioning)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-powerstore-001
  labels:
    storage-tier: premium
spec:
  capacity:
    storage: 100Gi
  
  accessModes:
    - ReadWriteOnce       # Single node read-write
  
  persistentVolumeReclaimPolicy: Retain
  storageClassName: dell-powerstore
  
  # CSI volume source
  csi:
    driver: csi-powerstore.dellemc.com
    volumeHandle: "vol-abc-123"          # ID on storage backend
    fsType: ext4
    volumeAttributes:
      protocol: iSCSI
      storagePool: default
  
  # Node affinity (topology constraints)
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - us-east-1a
  
  mountOptions:
    - noatime
    - nodiratime
```

### Access Modes

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| `ReadWriteOnce` | RWO | Single node read-write |
| `ReadOnlyMany` | ROX | Multiple nodes read-only |
| `ReadWriteMany` | RWX | Multiple nodes read-write (NFS, CephFS, PowerScale) |
| `ReadWriteOncePod` | RWOP | Single pod read-write (K8s 1.29+ GA) |

```
RWO: Block storage (iSCSI, FC, NVMe-oF) — most common
RWX: File storage (NFS, Dell PowerScale/Isilon) — shared access
RWOP: Strongest guarantee — exclusive to one pod (even on same node)
```

---

## 3. Persistent Volume Claims (PVC)

Namespace-scoped request for storage.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-storage
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  
  storageClassName: dell-powerstore-iscsi
  
  resources:
    requests:
      storage: 50Gi
  
  # Optional: select specific PV by label
  selector:
    matchLabels:
      storage-tier: premium
  
  # Volume mode
  volumeMode: Filesystem    # or Block (raw block device)

  # Data source for cloning or restoring from snapshot
  # dataSource:
  #   name: my-snapshot
  #   kind: VolumeSnapshot
  #   apiGroup: snapshot.storage.k8s.io
```

### PVC Binding Process

```
PVC Created (Pending)
        │
        ▼
┌──────────────────────────────────────┐
│  PV-PVC Binding Controller          │
│                                      │
│  1. Find matching PV:               │
│     ✓ StorageClass matches          │
│     ✓ Access modes satisfied        │
│     ✓ Capacity >= request           │
│     ✓ Selector labels match         │
│     ✓ Volume mode matches           │
│                                      │
│  2. If no matching PV:              │
│     → Dynamic provisioning          │
│     → StorageClass provisioner      │
│       creates new PV                 │
│                                      │
│  3. Bind: PV.spec.claimRef = PVC   │
│           PVC.spec.volumeName = PV  │
│           Both get status: Bound    │
└──────────────────────────────────────┘
```

### Using PVC in Pods

```yaml
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata      # Use subdirectory (avoid lost+found)
  
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: database-storage
        readOnly: false
```

---

## 4. Storage Classes

Enable **dynamic provisioning** — PVs are created automatically when PVCs are requested.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dell-powerstore-iscsi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Default SC
provisioner: csi-powerstore.dellemc.com

parameters:
  # Dell PowerStore specific parameters
  csi.storage.k8s.io/fstype: ext4
  storagePool: pool-0
  nasServer: nas-server-1          # For NFS volumes
  protocol: iSCSI                  # iSCSI, FC, NFS, NVMe
  arrayID: "PS000000001"
  
  # CSI specific
  csi.storage.k8s.io/provisioner-secret-name: powerstore-creds
  csi.storage.k8s.io/provisioner-secret-namespace: dell-storage

reclaimPolicy: Delete          # Delete PV when PVC is deleted
# Retain — keep PV and data (manual cleanup)
# Delete — delete PV and backend storage

allowVolumeExpansion: true     # Allow PVC resize

volumeBindingMode: WaitForFirstConsumer
# Immediate — provision as soon as PVC is created
# WaitForFirstConsumer — wait until pod using PVC is scheduled
#   (enables topology-aware provisioning)

mountOptions:
  - noatime
  - nodiratime

# Restrict to specific topology
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - us-east-1a
          - us-east-1b
```

### Multiple Storage Classes Example

```yaml
# Tier 1: High-performance NVMe
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-nvme
provisioner: csi-powerstore.dellemc.com
parameters:
  protocol: NVMe
  storagePool: nvme-pool
  csi.storage.k8s.io/fstype: xfs
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
# Tier 2: Standard iSCSI
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-iscsi
provisioner: csi-powerstore.dellemc.com
parameters:
  protocol: iSCSI
  storagePool: sas-pool
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# Tier 3: Shared file (NFS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-nfs
provisioner: csi-powerscale.dellemc.com
parameters:
  AccessZone: System
  AzServiceIP: 10.1.1.100
  IsiPath: /ifs/kubernetes
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

---

## 5. CSI (Container Storage Interface)

Standardized interface for storage plugins — replaces in-tree volume plugins.

### CSI Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 KUBERNETES CLUSTER                       │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │        CSI Controller (Deployment)        │           │
│  │                                          │           │
│  │  ┌──────────────┐  ┌──────────────────┐  │           │
│  │  │external-     │  │ CSI Driver       │  │           │
│  │  │provisioner   │──│ Controller       │  │           │
│  │  └──────────────┘  │ Plugin           │  │           │
│  │                    │                  │  │           │
│  │  ┌──────────────┐  │ gRPC socket:     │  │           │
│  │  │external-     │──│ /csi/csi.sock    │──┤──→ Storage│
│  │  │attacher      │  │                  │  │    Array  │
│  │  └──────────────┘  │ Implements:      │  │   (Dell   │
│  │                    │ • CreateVolume    │  │ PowerStore│
│  │  ┌──────────────┐  │ • DeleteVolume   │  │    etc.)  │
│  │  │external-     │──│ • ControllerPub  │  │           │
│  │  │snapshotter   │  │ • CreateSnapshot │  │           │
│  │  └──────────────┘  └──────────────────┘  │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │      CSI Node Plugin (DaemonSet)          │  Per Node│
│  │                                          │           │
│  │  ┌──────────────┐  ┌──────────────────┐  │           │
│  │  │node-driver-  │  │ CSI Driver       │  │           │
│  │  │registrar     │──│ Node Plugin      │  │           │
│  │  └──────────────┘  │                  │  │           │
│  │                    │ Implements:      │  │           │
│  │                    │ • NodeStageVol   │  │           │
│  │                    │ • NodePublishVol │  │           │
│  │                    │ • NodeGetInfo    │  │           │
│  │                    └──────────────────┘  │           │
│  └──────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

### CSI Sidecar Components

| Sidecar | Watches | Action |
|---------|---------|--------|
| **external-provisioner** | PersistentVolumeClaim | Calls `CreateVolume` / `DeleteVolume` |
| **external-attacher** | VolumeAttachment | Calls `ControllerPublishVolume` / `ControllerUnpublishVolume` |
| **external-snapshotter** | VolumeSnapshot | Calls `CreateSnapshot` / `DeleteSnapshot` |
| **external-resizer** | PVC (size change) | Calls `ControllerExpandVolume` |
| **node-driver-registrar** | — | Registers CSI driver with kubelet |
| **livenessprobe** | — | Health endpoint for CSI driver |

### CSI Volume Lifecycle

```
1. PVC Created
     │
     ▼
2. external-provisioner → CSI CreateVolume()
     │                     → Storage array creates LUN/volume
     ▼
3. PV Created (Bound to PVC)
     │
     ▼
4. Pod Scheduled to Node
     │
     ▼
5. external-attacher → CSI ControllerPublishVolume()
     │                  → Storage array maps LUN to node (iSCSI login)
     ▼
6. VolumeAttachment object created
     │
     ▼
7. kubelet → CSI NodeStageVolume()
     │        → Mount device to staging dir (/var/lib/kubelet/plugins/...)
     │        → Format if needed (mkfs.ext4)
     ▼
8. kubelet → CSI NodePublishVolume()
     │        → Bind mount from staging to pod dir
     │        → /var/lib/kubelet/pods/<uid>/volumes/...
     ▼
9. Container sees volume at mountPath

=== Pod Deletion (reverse) ===

10. kubelet → CSI NodeUnpublishVolume()
      │        → Unmount from pod dir
      ▼
11. kubelet → CSI NodeUnstageVolume()
      │        → Unmount from staging dir
      ▼
12. external-attacher → CSI ControllerUnpublishVolume()
      │                  → Storage array unmaps LUN from node
      ▼
13. Reclaim Policy:
      Delete → external-provisioner → CSI DeleteVolume()
      Retain → PV stays in Released state
```

### CSI Driver Object

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: csi-powerstore.dellemc.com
spec:
  attachRequired: true         # Controller publish needed
  podInfoOnMount: true         # Pass pod info to NodePublish
  fsGroupPolicy: File          # Apply fsGroup to all files
  volumeLifecycleModes:
    - Persistent               # Normal PV/PVC
    - Ephemeral                # CSI ephemeral volumes
  storageCapacity: true        # Report capacity
  tokenRequests:
    - audience: ""             # Bound SA token for auth
```

---

## 6. Volume Snapshots

Point-in-time copy of a volume.

### Components

```yaml
# VolumeSnapshotClass — defines snapshot provider
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: dell-powerstore-snap
driver: csi-powerstore.dellemc.com
deletionPolicy: Delete    # or Retain
parameters:
  csi.storage.k8s.io/snapshotter-secret-name: powerstore-creds
  csi.storage.k8s.io/snapshotter-secret-namespace: dell-storage
---
# VolumeSnapshot — user request
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot-20240115
spec:
  volumeSnapshotClassName: dell-powerstore-snap
  source:
    persistentVolumeClaimName: database-storage
---
# VolumeSnapshotContent — cluster-scoped, binds to VolumeSnapshot
# (auto-created by external-snapshotter for dynamic snapshots)
```

### Restore from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-restored
spec:
  storageClassName: dell-powerstore-iscsi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi    # Must be >= snapshot size
  dataSource:
    name: db-snapshot-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## 7. Volume Cloning

Create a new PVC from an existing PVC (without going through a snapshot).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-clone
spec:
  storageClassName: dell-powerstore-iscsi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: database-storage    # Source PVC
    kind: PersistentVolumeClaim
```

### Snapshot vs Clone

| Feature | Snapshot | Clone |
|---------|---------|-------|
| Data at | Point-in-time capture | Current state |
| Result | VolumeSnapshot object | New PVC (independent) |
| Storage backend | Copy-on-write (space efficient) | Full or COW copy |
| Use case | Backup, recovery | Test/dev, migration |

---

## 8. Volume Expansion

Resize a PVC without downtime (if supported by CSI driver and StorageClass).

```bash
# StorageClass must have allowVolumeExpansion: true

# Resize PVC
kubectl patch pvc database-storage -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Watch for completion
kubectl get pvc database-storage -w
```

### Expansion Flow

```
PVC spec.resources.requests.storage changed (50Gi → 100Gi)
        │
        ▼
external-resizer → CSI ControllerExpandVolume()
        │           → Storage array expands LUN
        ▼
PV.spec.capacity updated to 100Gi
        │
        ▼
kubelet → CSI NodeExpandVolume()      # Online filesystem resize
        │  → resize2fs (ext4) or xfs_growfs (xfs)
        ▼
PVC status.capacity updated to 100Gi
PVC condition: FileSystemResizePending → cleared
```

### Online vs Offline Expansion

| Mode | Description | When |
|------|-------------|------|
| **Online** | Expand while pod is running | Most modern CSI drivers + ext4/xfs |
| **Offline** | Must delete pod, expand, recreate | Older drivers or block size constraints |

---

## 9. Ephemeral Volumes

Volumes with pod-scoped lifecycle — created and deleted with the pod.

### Generic Ephemeral Volumes

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: scratch
          mountPath: /scratch
  
  volumes:
    - name: scratch
      ephemeral:
        volumeClaimTemplate:
          metadata:
            labels:
              type: scratch
          spec:
            accessModes: ["ReadWriteOnce"]
            storageClassName: fast-ssd
            resources:
              requests:
                storage: 10Gi
# PVC auto-created: <pod-name>-scratch
# PVC auto-deleted when pod is deleted
```

### CSI Ephemeral Volumes

```yaml
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: vault-secrets
```

---

## 10. Storage Capacity & Topology

### CSIStorageCapacity

```yaml
# Auto-created by CSI driver with storageCapacity: true
apiVersion: storage.k8s.io/v1
kind: CSIStorageCapacity
metadata:
  name: csisc-abc123
  namespace: dell-storage
storageClassName: dell-powerstore-iscsi
nodeTopology:
  matchLabels:
    topology.kubernetes.io/zone: us-east-1a
capacity: 500Gi            # Available capacity
maximumVolumeSize: 16Ti    # Max single volume
```

### Topology-Aware Provisioning

```yaml
# StorageClass
volumeBindingMode: WaitForFirstConsumer

# Flow:
# 1. PVC created → stays Pending (no immediate provisioning)
# 2. Pod using PVC is scheduled → scheduler picks node
# 3. Node's zone/region topology passed to CSI provisioner
# 4. CSI provisions volume in same zone as node
# 5. PV bound with nodeAffinity matching provisioned topology

# This prevents: Volume in zone-a, pod scheduled to zone-b
# (which would fail for zonal block storage)
```

---

## 11. Raw Block Volumes

Expose storage as a raw block device instead of a mounted filesystem.

```yaml
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block          # ← Key difference
  resources:
    requests:
      storage: 100Gi
  storageClassName: dell-powerstore-iscsi
---
# Pod
spec:
  containers:
    - name: db
      image: custom-db:v1
      volumeDevices:           # Not volumeMounts!
        - name: data
          devicePath: /dev/xvda   # Raw block device
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: raw-block-pvc
```

### Block vs Filesystem

| Aspect | Filesystem | Block |
|--------|-----------|-------|
| Access | File I/O (read/write) | Direct block I/O (dd, database engines) |
| Format | ext4, xfs, btrfs | No filesystem — raw bytes |
| Use case | Most applications | Databases with own storage engine, VMs |
| Performance | Filesystem overhead | Lower latency (no FS layer) |

---

## 12. fsGroup and Volume Permissions

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000           # Applied to all mounted volumes
    fsGroupChangePolicy: OnRootMismatch  # or Always
    # OnRootMismatch: Only change if root dir permissions don't match
    # Always: Change on every mount (slower for large volumes)
  
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
```

### fsGroup Behavior

```
Without fsGroup:
  Files owned by root:root → app (uid 1000) can't write

With fsGroup: 2000:
  1. All files chown'd to :2000  (supplemental group)
  2. setgid bit set on directories
  3. New files created with group 2000
  4. App (uid 1000, supplemental group 2000) can read/write

Performance impact:
  - Recursive chown on large volumes → slow mount
  - OnRootMismatch → check root only, skip if already correct
```

---

## 13. Dell CSI Drivers

### Dell CSI Product Matrix

| Driver | Storage Platform | Protocols | Access Modes |
|--------|-----------------|-----------|-------------|
| **csi-powerstore** | Dell PowerStore | iSCSI, FC, NFS, NVMe-TCP | RWO, ROX, RWX (NFS) |
| **csi-powerscale** | Dell PowerScale (Isilon) | NFS | RWO, ROX, RWX |
| **csi-powerflex** | Dell PowerFlex (VxFlex) | SDC (proprietary), NFS | RWO, ROX, RWX (NFS) |
| **csi-unity** | Dell Unity | iSCSI, FC, NFS | RWO, ROX, RWX (NFS) |
| **csi-powermax** | Dell PowerMax | iSCSI, FC | RWO, ROX |

### Dell CSI PowerStore Deployment

```yaml
# Typical deployment components:
# 1. Controller (Deployment, 2 replicas for HA)
#    - csi-powerstore driver container
#    - external-provisioner sidecar
#    - external-attacher sidecar
#    - external-snapshotter sidecar
#    - external-resizer sidecar
#    - external-health-monitor sidecar
#
# 2. Node (DaemonSet, every node or selected nodes)
#    - csi-powerstore driver container (privileged)
#    - node-driver-registrar sidecar
#
# 3. Secrets, StorageClasses, VolumeSnapshotClasses
```

### Dell CSI Features

| Feature | Description |
|---------|-------------|
| **Dynamic provisioning** | Create/delete volumes on demand |
| **Snapshots** | Point-in-time snapshots with restore |
| **Cloning** | Clone volumes from PVC or snapshot |
| **Expansion** | Online volume resize |
| **Raw block** | Direct block device access |
| **Topology** | Zone/rack-aware provisioning |
| **Multi-protocol** | iSCSI, FC, NFS, NVMe in same driver |
| **Replication** | Dell Replication CRDs for DR |
| **Encryption** | Storage-side encryption |
| **QoS** | IOPS/bandwidth limits via parameters |

---

## Interview Questions

1. **Explain the CSI architecture. What are the controller and node components?**
   - Controller: runs as Deployment. Handles CreateVolume, DeleteVolume, ControllerPublishVolume (attach to node), snapshots. Sidecar containers (external-provisioner, attacher, snapshotter) watch K8s objects and call CSI gRPC methods.
   - Node: runs as DaemonSet. Handles NodeStageVolume (mount to staging), NodePublishVolume (bind mount to pod), NodeGetInfo (topology). Runs privileged for mount operations.

2. **What is the difference between RWO, RWX, ROX, and RWOP?**
   - RWO: one node read-write (block storage)
   - ROX: many nodes read-only
   - RWX: many nodes read-write (file storage like NFS/PowerScale)
   - RWOP: one pod read-write (strictest, prevents two pods on same node)

3. **How does dynamic provisioning work?**
   - PVC references StorageClass → external-provisioner watches PVCs → calls CSI CreateVolume with StorageClass parameters → storage array creates volume → PV created and bound to PVC.

4. **What is WaitForFirstConsumer and why is it important?**
   - Delays volume provisioning until pod is scheduled. Ensures volume is created in the same topology zone as the pod. Without it, a volume might be provisioned in zone-a but pod scheduled to zone-b (fails for zonal block storage).

5. **How do volume snapshots work end-to-end?**
   - User creates VolumeSnapshot referencing PVC → external-snapshotter calls CSI CreateSnapshot → storage array creates point-in-time snapshot → VolumeSnapshotContent created (cluster-scoped, references actual snapshot). To restore: create PVC with dataSource referencing VolumeSnapshot.

6. **How would you design a CSI driver for Dell PowerStore?**
   - Controller service: authenticate to PowerStore API, implement CreateVolume (create LUN), DeleteVolume, snapshot operations. Node service: run iSCSI login (NodeStage), format filesystem and mount (NodePublish). Use external sidecars for K8s integration. Support multi-protocol by parameterizing iSCSI/FC/NFS. Implement topology reporting for rack/zone awareness. Add health monitoring for proactive failure detection.

7. **What happens to PVCs when a StatefulSet is scaled down?**
   - By default, PVCs are NOT deleted (data is preserved). StatefulSet.spec.persistentVolumeClaimRetentionPolicy (K8s 1.27+) controls this: `whenScaled: Delete` removes PVCs on scale-down, `whenDeleted: Retain` keeps PVCs on StatefulSet deletion.

8. **How does volume expansion work? Can it happen online?**
   - PVC spec.resources.requests.storage increased → ControllerExpandVolume (backend resize) → NodeExpandVolume (filesystem resize). Online expansion: supported by ext4, xfs, and most modern CSI drivers — no pod restart needed. Offline: delete pod, wait for resize, recreate.

---

*Next: [Chapter 5 — Networking](Networking.md)*
