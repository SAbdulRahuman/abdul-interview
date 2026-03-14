# Chapter 15 — Interview Questions

> Comprehensive Kubernetes interview questions organized by topic and difficulty. Focused on storage infrastructure and platform engineering roles at Dell.

---

## Architecture & Internals

### Q1: What happens when you run `kubectl apply -f deployment.yaml`?

**Answer:**
1. kubectl reads the YAML, identifies API group/version (apps/v1 Deployment)
2. Sends HTTPS POST to API server
3. **Authentication**: validates client cert/token
4. **Authorization**: RBAC check — can user create Deployment in this namespace?
5. **Mutating admission**: webhooks inject defaults (service account, sidecar if Istio)
6. **Schema validation**: validate against OpenAPI schema
7. **Validating admission**: policy webhooks (OPA/Gatekeeper check resource limits present)
8. **Persist to etcd**: API server writes Deployment, etcd replicates via Raft
9. **Deployment controller**: notified via watch, creates ReplicaSet
10. **ReplicaSet controller**: creates Pod objects (nodeName="")
11. **Scheduler**: filter → score → bind, sets pod.spec.nodeName
12. **kubelet on target node**: pulls image (CRI → containerd), sets up network (CNI), mounts volumes (CSI), starts container, runs probes
13. **Endpoint controller**: adds pod IP to Service Endpoints
14. **kube-proxy**: updates iptables/IPVS rules for service routing

---

### Q2: Explain the reconciliation pattern. Why is it central to Kubernetes?

**Answer:**
Controllers use a **level-triggered** observe-diff-act loop:
- **Observe**: read current cluster state from informer cache
- **Diff**: compare with desired state (.spec)
- **Act**: create/update/delete resources to converge
- **Repeat**: re-sync periodically even without events

Why it's central:
- **Self-healing**: if controller crashes and restarts, it catches up from current state
- **Idempotent**: running reconcile multiple times produces same result
- **Composable**: multiple controllers can manage different aspects independently
- **Resilient**: doesn't rely on event ordering or delivery guarantees

---

### Q3: How does etcd store cluster state? What happens if etcd fails?

**Answer:**
- etcd stores all K8s objects as key-value pairs under `/registry/` prefix
- Uses Raft consensus for replication (leader + followers)
- Writes require majority quorum: 3-node cluster tolerates 1 failure
- If etcd loses quorum: API server becomes read-only (eventually errors), no new changes, running pods continue unaffected
- Recovery: restore from snapshot, rejoin members, or rebuild cluster

---

## Workloads

### Q4: Deployment vs StatefulSet — when to use each?

| Aspect | Deployment | StatefulSet |
|--------|------------|-------------|
| **Pod identity** | Random names, interchangeable | Stable names (app-0, app-1) |
| **Storage** | Shared or no persistent storage | Per-pod PVCs (volumeClaimTemplates) |
| **Ordering** | Parallel creation/deletion | Sequential (configurable) |
| **Network** | Random pod IPs via Service | Stable DNS per pod (headless service) |
| **Use case** | Stateless APIs, frontends | Databases, etcd, Kafka, ZooKeeper |
| **Scaling** | Any order | Ordered (0, 1, 2...) |

---

### Q5: What is the difference between liveness, readiness, and startup probes?

| Probe | Purpose | Failure Action | When |
|-------|---------|----------------|------|
| **Startup** | Is the app done starting? | Kill & restart | Before other probes |
| **Liveness** | Is the app alive? | Kill & restart | After startup succeeds |
| **Readiness** | Can it serve traffic? | Remove from Service endpoints | After startup succeeds |

Key insight: A pod can be alive (liveness passes) but not ready (readiness fails) — e.g., warming cache. It stays in the pod list but receives no traffic.

---

### Q6: How do you handle graceful shutdown in Kubernetes?

1. `deletionTimestamp` set on pod
2. Pod removed from Service endpoints (parallel with step 3)
3. `preStop` hook runs (if defined)
4. `SIGTERM` sent to PID 1 in all containers
5. Wait `terminationGracePeriodSeconds` (default 30)
6. `SIGKILL` sent

**Race condition**: endpoint removal takes time to propagate. Solution: add `preStop: sleep 10` to wait for kube-proxy updates before shutting down.

App should:
- Handle SIGTERM signal
- Stop accepting new requests
- Finish in-flight requests
- Close connections, flush buffers
- Exit cleanly

---

## Storage (Critical for Dell)

### Q7: Explain the CSI architecture.

**Controller Plugin** (Deployment):
- Sidecars: external-provisioner, external-attacher, external-snapshotter, external-resizer
- Handles: CreateVolume, DeleteVolume, ControllerPublishVolume (attach LUN to node), snapshots
- Communicates with storage array REST API

**Node Plugin** (DaemonSet):
- Sidecar: node-driver-registrar
- Handles: NodeStageVolume (mount to staging dir, format), NodePublishVolume (bind mount to pod path)
- Runs privileged (needs mount, device access, iSCSI login)

Communication: gRPC over Unix domain socket between sidecars and driver.

---

### Q8: What is the difference between RWO, RWX, ROX, and RWOP?

| Mode | Nodes | Access | Typical Storage |
|------|-------|--------|----------------|
| RWO | 1 | Read-Write | Block (iSCSI, FC, NVMe) |
| ROX | Many | Read-Only | Any (read-only share) |
| RWX | Many | Read-Write | File (NFS, Dell PowerScale) |
| RWOP | 1 pod | Read-Write | Block (strictest isolation) |

RWO allows multiple **pods on the same node** to access the volume. RWOP restricts to a single pod — even other pods on the same node can't access it.

---

### Q9: How does dynamic provisioning work?

1. PVC created referencing StorageClass
2. `external-provisioner` sidecar watches PVCs
3. Matches PVC to StorageClass, calls CSI `CreateVolume()` with SC parameters
4. CSI driver calls storage array API to create LUN/volume
5. PV created automatically with CSI volume source
6. PV bound to PVC (1:1 relationship)
7. When PVC deleted: reclaimPolicy determines fate (Delete → CSI DeleteVolume, Retain → PV stays)

---

### Q10: Explain topology-aware provisioning (WaitForFirstConsumer).

**Problem**: with `volumeBindingMode: Immediate`, PV is provisioned immediately when PVC is created. If volume is created in zone-a but pod gets scheduled to zone-b, the mount fails (block storage is zonal).

**Solution**: `WaitForFirstConsumer`
1. PVC stays Pending (no immediate provisioning)
2. Pod referencing PVC is submitted to scheduler
3. Scheduler considers volume topology during node scoring
4. Node selected (e.g., zone-b)
5. Provisioner creates volume in zone-b
6. PV created with nodeAffinity for zone-b
7. PVC bound, pod starts

This is essential for Dell block storage (PowerStore iSCSI/FC) where volumes are zonal.

---

### Q11: How do volume snapshots work in Kubernetes?

1. User creates `VolumeSnapshot` referencing source PVC and `VolumeSnapshotClass`
2. `external-snapshotter` watches VolumeSnapshot objects
3. Calls CSI `CreateSnapshot()` on the driver
4. Driver calls storage array to create point-in-time snapshot
5. `VolumeSnapshotContent` created (cluster-scoped, references actual storage snapshot)
6. VolumeSnapshot status becomes `readyToUse: true`

**Restore**: create new PVC with `dataSource` referencing VolumeSnapshot → provisioner creates new volume from snapshot.

---

### Q12: How would you design a CSI driver for Dell PowerStore?

**Architecture:**
- Controller: Go binary implementing CSI Controller service. REST client for PowerStore API. External sidecars for K8s integration.
- Node: Go binary implementing CSI Node service. Runs privileged. Handles iSCSI/FC login, multipath, format, mount.

**Volume lifecycle:**
- CreateVolume: call PowerStore API to create LUN, specify size/pool/protection
- ControllerPublishVolume: map LUN to host (iSCSI target → initiator mapping)
- NodeStageVolume: discover device (iSCSI rescan), format (mkfs.ext4), mount to staging
- NodePublishVolume: bind mount to pod directory

**Features:**
- Multi-protocol: iSCSI, FC, NFS, NVMe-TCP via parameters
- Topology: report zone/rack via NodeGetInfo, WaitForFirstConsumer
- Snapshots: map to PowerStore snapshots
- Expansion: PowerStore volume extend API + online resize2fs/xfs_growfs
- Replication: DR via Dell Replication CRDs
- Health monitoring: periodic volume health checks

---

### Q13: How does a CSI driver handle node failure and volume failover?

1. Node goes NotReady (kubelet stops reporting)
2. After `pod-eviction-timeout` (default 5 min), pods are evicted
3. For StatefulSet: pod is recreated on another node
4. Problem: old node still has volume attached (force-detach needed)
5. `VolumeAttachment` object has `attached: true` for old node
6. `external-attacher` waits for old attachment to be removed
7. `force-detach` needed: admin or automation removes stale VolumeAttachment
8. New node gets `ControllerPublishVolume` (re-attach to new node)
9. NodeStageVolume → NodePublishVolume on new node

**Key considerations:**
- fencing: ensure old node can't write to volume (iSCSI reservation, SCSI-3 PR)
- Data integrity: dirty unmount may require fsck
- Timeout tuning: balance between fast failover and false positives

---

## Networking

### Q14: How does kube-proxy implement service load balancing?

**iptables mode** (default):
- Creates DNAT rules in NAT table
- Each service gets a chain with rules per endpoint
- Random probability-based distribution (statistic mode)
- O(n) rule matching in kernel

**IPVS mode:**
- Uses Linux Virtual Server (kernel module)
- Hash table lookup: O(1)
- Supports algorithms: rr, lc, dh, sh, sed, nq
- Better for large clusters (10K+ services)

When switching between modes, kube-proxy cleans up old rules.

---

### Q15: How does DNS work in Kubernetes?

CoreDNS runs as deployment in kube-system namespace.

**Records:**
- Service: `<svc>.<ns>.svc.cluster.local` → ClusterIP
- Headless: `<svc>.<ns>.svc.cluster.local` → all pod IPs
- StatefulSet: `<pod>.<svc>.<ns>.svc.cluster.local` → specific pod IP
- SRV: `_<port>._tcp.<svc>.<ns>.svc.cluster.local` → port + hostname

**ndots issue**: default ndots=5 means external names get search domain suffixes appended, causing 4+ DNS queries. Fix: use FQDNs with trailing dot or reduce ndots.

---

## Security

### Q16: Explain RBAC in Kubernetes.

Four objects:
- **Role**: permissions within a namespace (rules: apiGroups + resources + verbs)
- **ClusterRole**: cluster-wide permissions (or reusable across namespaces)
- **RoleBinding**: grants Role/ClusterRole to subjects within a namespace
- **ClusterRoleBinding**: grants ClusterRole to subjects cluster-wide

Subjects: User, Group, ServiceAccount

Principle of least privilege: start with zero permissions, add only what's needed.

Example: CSI controller needs ClusterRole with access to PV, PVC, StorageClass, VolumeAttachment, Node, Events + ClusterRoleBinding to CSI ServiceAccount.

---

### Q17: How do you manage secrets securely?

Layered approach:
1. **etcd encryption at rest**: EncryptionConfiguration with aescbc/aesgcm
2. **RBAC**: restrict who can read secrets (minimize "get secrets" permissions)
3. **External secret management**: Vault, AWS Secrets Manager with External Secrets Operator
4. **Sealed Secrets**: encrypted in Git, decrypted only in cluster
5. **Short-lived tokens**: bound service account tokens (auto-rotated, audience-scoped)
6. **Audit logging**: track all secret access
7. **Avoid env vars**: prefer volume mounts (env vars show in `kubectl describe`)

---

## Operations

### Q18: Walk through a Kubernetes cluster upgrade strategy.

1. **Read release notes** for deprecations and breaking changes
2. **Backup etcd** before upgrade
3. **Upgrade control plane** (one minor version at a time):
   - Update kubeadm, run `kubeadm upgrade apply`
   - Upgrade kubelet on control plane nodes
4. **Rolling upgrade workers**:
   - For each node: cordon → drain (respects PDB) → upgrade kubelet → uncordon
5. **Post-upgrade**: verify node versions, test workloads, check deprecation warnings
6. **Version skew**: kubelet can be N-2 from API server, so rolling upgrade is safe

---

### Q19: How does Horizontal Pod Autoscaler work?

Control loop (every 15s):
1. Query metrics (metrics-server for CPU/memory, Prometheus adapter for custom)
2. Calculate desired replicas: `ceil(current × (currentMetric / targetMetric))`
3. Apply stabilization window (prevent flapping)
4. Scale target Deployment/StatefulSet

Behavior config:
- Scale-up: stabilization window (default 0), policies (percent/count + period)
- Scale-down: stabilization window (default 5 min), conservative policies

For storage: scale on custom metrics like queue depth, connection count, IOPS — not just CPU.

---

## Operators

### Q20: What is the Operator pattern? When would you build one?

**Pattern**: CRD + custom controller that encodes operational knowledge.

**Build when:**
- Day 2 operations are complex (backup, restore, upgrade, failover)
- Managing external resources (storage arrays, cloud resources)
- Need domain-specific automation (not just deploying YAML)
- Standard K8s resources are insufficient

**Example**: Dell CSI operator manages CSI driver lifecycle — installs driver, configures storage classes, handles driver upgrades, monitors health, manages credentials rotation.

**Maturity levels**: Basic Install → Upgrades → Full Lifecycle → Deep Insights → Auto Pilot

---

### Q21: Explain finalizers. What goes wrong if a finalizer isn't removed?

Finalizers are strings in `metadata.finalizers` that prevent deletion. When DELETE is called:
1. `deletionTimestamp` is set (object still exists)
2. Controller sees timestamp, performs cleanup
3. Controller removes finalizer
4. When all finalizers removed → object deleted from etcd

**If not removed**: object stuck in "Terminating" state forever. Namespace with stuck finalizer = namespace can't be deleted. Fix: manual patch to remove finalizer (risky — external resource may leak).

---

### Q22: How do admission webhooks work?

During API request processing (after auth, before persistence):
1. **Mutating webhooks**: modify the request (inject sidecar, set defaults, add labels)
2. **Object validation**: schema check
3. **Validating webhooks**: approve/reject (enforce policies)

Configuration: webhook URL, CA bundle, rules (which resources/operations trigger it), failurePolicy (Fail/Ignore), timeout (max 10s recommended).

Use cases:
- Istio sidecar injection (mutating)
- OPA Gatekeeper policy enforcement (validating)
- Default resource limits injection (mutating)
- Image registry whitelisting (validating)

---

## Dell-Specific Scenarios

### Q23: How would you design a storage operator for Dell?

**CRDs:**
- `DellCSIDriver`: manage driver installation and lifecycle
- `StorageArray`: represent Dell arrays (PowerStore, PowerScale)
- `ReplicationGroup`: manage DR replication between arrays

**Controller logic:**
- Reconcile DellCSIDriver: deploy controller Deployment + node DaemonSet, configure RBAC, create StorageClasses
- Watch StorageArray: monitor array health, capacity, firmware version
- Handle ReplicationGroup: setup replication links between source/target arrays

**Key considerations:**
- Multi-array support (same operator manages multiple arrays)
- Credential rotation (rotate storage admin passwords)
- Health monitoring (alert on array issues)
- Upgrade strategy (rolling update of CSI driver pods)
- Topology awareness (publish zone/rack info from array)

---

### Q24: How does a storage platform handle multi-tenancy in Kubernetes?

**Isolation layers:**
1. **Namespaces**: per-tenant isolation
2. **RBAC**: restrict storage operations per namespace
3. **ResourceQuotas**: limit storage consumption per tenant
4. **StorageClasses**: tenant-specific classes with different backends/QoS
5. **Network Policies**: isolate management traffic between tenants
6. **Volume encryption**: per-tenant encryption keys
7. **CSI parameters**: separate storage pools per tenant on array

**Implementation:**
```yaml
# Tenant-specific StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tenant-a-premium
provisioner: csi-powerstore.dellemc.com
parameters:
  storagePool: tenant-a-pool
  csi.storage.k8s.io/provisioner-secret-name: tenant-a-creds
```

---

## Behavioral/Design Questions

### Q25: How would you troubleshoot a production storage issue?

**Systematic approach:**
1. **Identify scope**: single pod, namespace, node, or cluster-wide?
2. **Check pod events**: `kubectl describe pod` → mount errors, attach failures
3. **Check PVC/PV status**: bound? volume attachment exists?
4. **CSI driver logs**: controller and node pod logs
5. **Node-level**: iSCSI sessions, multipath status, device presence
6. **Storage array**: volume exists? mapped to correct host? offline?
7. **Network**: can node reach array? firewall? DNS?
8. **Correlate**: timestamps across pod events, CSI logs, array logs

**Communication**: update incident channel, clear timeline, escalation path if needed.

---

### Q26: How do you approach upgrading a CSI driver in production?

1. **Pre-flight**: read release notes, check K8s compatibility, backup etcd
2. **Staging**: test upgrade in staging cluster with same storage config
3. **Canary**: upgrade node DaemonSet on one node first (partition)
4. **Monitor**: watch CSI metrics, volume operations, error rates
5. **Rolling**: upgrade remaining nodes, then controller
6. **Validation**: test volume create/delete/snapshot/expand after upgrade
7. **Rollback plan**: keep previous version image, revert if issues

Key risk: node DaemonSet upgrade briefly interrupts volume operations on that node. Ensure no active volume operations during pod restart.

---

*End of Kubernetes Interview Preparation Guide*
