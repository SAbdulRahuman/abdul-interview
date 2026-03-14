# Chapter 11 — Cluster Operations

## 1. Cluster Upgrades

### Version Skew Policy

| Component | Allowed Skew from API Server |
|-----------|------------------------------|
| kubelet | -2 minor versions |
| kube-proxy | -2 minor versions (same as kubelet) |
| kubectl | ±1 minor version |
| kube-controller-manager | -1 minor version |
| kube-scheduler | -1 minor version |

### Upgrade Strategy

```
Phase 1: Upgrade Control Plane (one at a time in HA)
  1. Upgrade etcd (if separate)
  2. Upgrade kube-apiserver
  3. Upgrade kube-controller-manager
  4. Upgrade kube-scheduler

Phase 2: Upgrade Worker Nodes (rolling)
  For each node:
    1. kubectl cordon node     → Mark unschedulable
    2. kubectl drain node      → Evict pods (respects PDB)
    3. Upgrade kubelet + kube-proxy
    4. Restart kubelet
    5. kubectl uncordon node   → Mark schedulable
```

### kubeadm Upgrade

```bash
# Control plane upgrade
sudo apt-get update
sudo apt-get install -y kubeadm=1.30.0-*

kubeadm upgrade plan
kubeadm upgrade apply v1.30.0

sudo apt-get install -y kubelet=1.30.0-* kubectl=1.30.0-*
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Worker node upgrade
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
# On worker-1:
sudo apt-get install -y kubeadm=1.30.0-*
kubeadm upgrade node
sudo apt-get install -y kubelet=1.30.0-*
sudo systemctl daemon-reload && sudo systemctl restart kubelet
# Back on control plane:
kubectl uncordon worker-1
```

---

## 2. Node Operations

### Cordon / Drain / Uncordon

```bash
# Cordon: mark node unschedulable (existing pods stay)
kubectl cordon worker-1

# Drain: evict all pods (respects PDB, ignores DaemonSets)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=120 \
  --timeout=300s \
  --force            # Force delete pods not managed by controllers

# Uncordon: mark node schedulable again
kubectl uncordon worker-1

# Drain gotchas:
# - Blocks if PDB would be violated
# - emptyDir data is lost
# - Standalone pods (no controller) won't be rescheduled
# - DaemonSet pods are always ignored
```

---

## 3. etcd Operations

### Backup

```bash
# Snapshot save
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
etcdctl snapshot status /backup/etcd-20240115-0200.db --write-out=table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | a1b2c3d4 |  1234567 |       5432 |     25 MB  |
# +----------+----------+------------+------------+
```

### Restore

```bash
# Stop API server and etcd
# Restore snapshot to new data dir
etcdctl snapshot restore /backup/etcd-20240115-0200.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.1.10:2380 \
  --initial-advertise-peer-urls=https://10.0.1.10:2380

# Update etcd config to point to restored data dir
# Restart etcd and API server
```

### Maintenance

```bash
# Defragmentation (reclaim space after compaction)
etcdctl defrag --cluster

# Compaction (remove old revisions)
etcdctl compact $(etcdctl endpoint status --write-out=json | jq '.[0].Status.header.revision')

# Endpoint health
etcdctl endpoint health --cluster --write-out=table

# Member list
etcdctl member list --write-out=table
```

---

## 4. Certificate Management

```
Kubernetes PKI Structure:
├── ca.crt / ca.key                    # Cluster CA
├── apiserver.crt / apiserver.key      # API server TLS
├── apiserver-kubelet-client.crt       # API server → kubelet
├── front-proxy-ca.crt                 # Aggregation layer CA
├── sa.key / sa.pub                    # Service account signing
├── etcd/
│   ├── ca.crt / ca.key               # etcd CA
│   ├── server.crt                     # etcd server TLS
│   └── peer.crt                       # etcd peer TLS
```

```bash
# Check certificate expiry
kubeadm certs check-expiration

# Renew all certificates
kubeadm certs renew all

# kubelet certificate auto-rotation (enabled by default)
# Via kubelet flag: --rotate-certificates
```

---

## 5. HA Control Plane

### Stacked etcd Topology

```
Each control plane node runs:
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler
  - etcd (local)

Pros: Fewer nodes, simpler setup
Cons: etcd failure = control plane failure on that node
Min: 3 nodes for HA (etcd quorum = 2)
```

### External etcd Topology

```
Control plane nodes (3):
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler

Separate etcd cluster (3-5 nodes):
  - Dedicated resources
  - Independent scaling
  - Can tolerate more failures

Better for: Large clusters, strict performance requirements
```

---

## 6. Disaster Recovery with Velero

```bash
# Install Velero
velero install \
  --provider aws \
  --bucket k8s-backups \
  --secret-file ./credentials \
  --use-volume-snapshots=true \
  --plugins velero/velero-plugin-for-aws:v1.9

# Backup entire namespace
velero backup create storage-backup \
  --include-namespaces dell-storage \
  --include-resources '*' \
  --snapshot-volumes=true

# Schedule daily backup
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces dell-storage,production \
  --ttl 720h

# Restore
velero restore create --from-backup storage-backup \
  --namespace-mappings dell-storage:dell-storage-restored
```

---

## 7. Node Problem Detector

Detects node issues and reports as conditions/events.

```
Monitors:
├── Kernel issues (OOM, hung task, kernel panic)
├── Filesystem corruption
├── Container runtime issues (containerd unresponsive)
├── NTP drift
├── Hardware issues (memory errors, disk failures)
└── Custom health checks (scripts)
```

```yaml
# Node conditions added by NPD
status:
  conditions:
    - type: KernelDeadlock
      status: "False"
    - type: ReadonlyFilesystem
      status: "False"
    - type: CorruptDockerOverlay2
      status: "False"
    - type: FrequentKubeletRestart
      status: "False"
```

---

## 8. Capacity Planning

```bash
# Current resource utilization
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# worker-1   1200m        15%    8192Mi          25%
# worker-2   3500m        43%    16384Mi         50%

# Resource requests/limits summary
kubectl describe nodes worker-1 | grep -A 5 "Allocated resources"
# Allocated resources:
#   Resource           Requests     Limits
#   --------           --------     ------
#   cpu                4100m (51%)  8000m (100%)
#   memory             12Gi (37%)   24Gi (75%)
#   ephemeral-storage  0 (0%)       0 (0%)

# Overcommit ratio
# If sum(requests) < node capacity → room to schedule
# If sum(limits) > node capacity → overcommitted (risk of OOM)
```

### Right-Sizing Recommendations

```promql
# Find pods requesting more CPU than they use
(
  kube_pod_container_resource_requests{resource="cpu"}
  - 
  rate(container_cpu_usage_seconds_total[7d])
) > 0.5
# Pods requesting 500m+ more CPU than they actually use
```

---

## 9. Cluster API (CAPI)

Declarative cluster lifecycle management.

```yaml
# Declare a cluster — CAPI creates it
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-cp
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production

# Machine Deployments (like Deployments but for nodes)
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: production-workers
spec:
  replicas: 5
  template:
    spec:
      version: v1.30.0
      bootstrap:
        configRef:
          kind: KubeadmConfigTemplate
      infrastructureRef:
        kind: AWSMachineTemplate
```

---

## Interview Questions

1. **How do you upgrade a Kubernetes cluster?**
   - Control plane first (one minor version at a time), then worker nodes rolling. Respect version skew policy (kubelet can be N-2 from API server). Drain nodes before upgrading kubelet. Respect PDBs during drain.

2. **How do you handle etcd backup and disaster recovery?**
   - Regular etcd snapshots (etcdctl snapshot save). Store off-cluster. Restore: stop cluster, restore snapshot to new data dir, restart. Also use Velero for application-level backup including PVs.

3. **What happens during kubectl drain?**
   - Node marked unschedulable (cordon). Pods are evicted (graceful termination). Respects PDB (waits if eviction would violate). DaemonSet pods ignored. Standalone pods need --force. emptyDir data lost.

4. **How do certificates work in Kubernetes?**
   - PKI hierarchy: cluster CA signs component certificates. API server, kubelet, etcd each have TLS certs. Kubelet auto-rotates its client cert. kubeadm certificates expire in 1 year by default — `kubeadm certs renew all`.

---

*Next: [Chapter 12 — Service Mesh & Advanced Networking](ServiceMeshAndAdvancedNetworking.md)*
