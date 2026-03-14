# Chapter 1 — Architecture & Components

## 1. Kubernetes Cluster Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              CONTROL PLANE                   │
                    │                                             │
                    │  ┌──────────┐  ┌───────────┐  ┌──────────┐│
                    │  │kube-api  │  │  etcd     │  │scheduler ││
                    │  │server    │←→│ (Raft)    │  │          ││
                    │  └────┬─────┘  └───────────┘  └──────────┘│
                    │       │                                     │
                    │  ┌────┴──────────────┐  ┌───────────────┐  │
                    │  │controller-manager │  │cloud-controller│  │
                    │  └───────────────────┘  └───────────────┘  │
                    └─────────────┬───────────────────────────────┘
                                  │  (API calls)
                    ┌─────────────┼───────────────────────────────┐
                    │             ▼         WORKER NODE            │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
                    │  │ kubelet  │  │kube-proxy│  │container │  │
                    │  │          │  │          │  │ runtime  │  │
                    │  └────┬─────┘  └──────────┘  └──────────┘  │
                    │       │                                      │
                    │  ┌────┴─────────────────────────────────┐   │
                    │  │  Pods  [C1] [C2]    [C3] [C4]        │   │
                    │  └──────────────────────────────────────┘   │
                    └─────────────────────────────────────────────┘
```

### Design Principles

| Principle | Description |
|-----------|-------------|
| **Declarative** | Users declare desired state; system reconciles to match |
| **Control Loops** | Controllers watch → diff → act continuously |
| **API-Centric** | All interaction through kube-apiserver REST API |
| **Extensible** | CRDs, admission webhooks, custom controllers extend API |
| **Loosely Coupled** | Components communicate via API server, not directly |
| **Self-Healing** | Automatic restart, reschedule, replicate, and kill unhealthy pods |

---

## 2. kube-apiserver

The API server is the **central hub** of the Kubernetes control plane. Every component communicates through it.

### Responsibilities

1. **REST API endpoint** — serves the Kubernetes API (CRUD on resources)
2. **Authentication** — X.509 certs, bearer tokens, OIDC, webhook
3. **Authorization** — RBAC, ABAC, Node authorizer, webhook
4. **Admission Control** — mutating + validating admission webhooks
5. **etcd gateway** — only component that reads/writes etcd directly
6. **Watch mechanism** — long-poll watches for resource changes

### Request Processing Pipeline

```
kubectl apply -f deploy.yaml
        │
        ▼
┌─────────────────────┐
│   Authentication    │  Who is this? (certs, tokens, OIDC)
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│   Authorization     │  Are they allowed? (RBAC check)
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ Mutating Admission  │  Modify request (inject sidecar, defaults)
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ Object Validation   │  Schema validation
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ Validating Admission│  Final validation (policy checks)
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│   etcd Write        │  Persist resource
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│   Watch Notification│  Notify controllers via watch streams
└─────────────────────┘
```

### Key Flags

```bash
# HA setup with multiple instances
--etcd-servers=https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
--service-cluster-ip-range=10.96.0.0/12
--service-node-port-range=30000-32767
--enable-admission-plugins=NodeRestriction,PodSecurity
--authorization-mode=Node,RBAC
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

### API Discovery

```bash
# List all API groups
kubectl api-versions

# List all resources
kubectl api-resources

# Explain a resource's fields
kubectl explain pod.spec.containers

# Raw API access
kubectl get --raw /api/v1/namespaces/default/pods
```

---

## 3. etcd

Distributed key-value store that serves as Kubernetes' **single source of truth**.

### Characteristics

| Property | Detail |
|----------|--------|
| **Consensus** | Raft protocol — leader election, log replication |
| **Consistency** | Linearizable reads (via leader) or serializable (stale) |
| **Storage** | bbolt (B+tree) — key-value on disk |
| **Cluster Size** | 3 or 5 members typical (odd number for quorum) |
| **Quorum** | (n/2) + 1 members must agree for writes |

### Data Model

```
/registry/deployments/default/nginx    → Deployment object JSON
/registry/pods/default/nginx-abc123    → Pod object JSON  
/registry/services/default/my-svc      → Service object JSON
/registry/secrets/default/my-secret    → Secret object JSON
```

### Raft Consensus

```
                  ┌─────────┐
                  │ Leader  │  ← Handles all writes
                  │ etcd-1  │
                  └────┬────┘
                       │  Log replication
              ┌────────┼────────┐
              ▼        ▼        ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │Follower│ │Follower│ │Follower│
         │etcd-2  │ │etcd-3  │ │etcd-4  │
         └────────┘ └────────┘ └────────┘
         
Write committed when majority (quorum) acknowledges
5-node cluster: quorum = 3 → tolerates 2 failures
3-node cluster: quorum = 2 → tolerates 1 failure
```

### Operational Commands

```bash
# Check cluster health
etcdctl endpoint health --cluster

# List all keys (for debugging)
etcdctl get / --prefix --keys-only | head -20

# Snapshot for backup
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db

# Restore from snapshot
etcdctl snapshot restore /backup/etcd-20240101.db \
  --data-dir=/var/lib/etcd-restored

# Defragmentation (reclaim space)
etcdctl defrag --cluster

# Check database size
etcdctl endpoint status --write-out=table
```

### Performance Considerations

- **Storage backend**: SSD strongly recommended (write-heavy)
- **Default DB size limit**: 2 GB (configurable via `--quota-backend-bytes`)
- **Compaction**: Periodic history compaction prevents unbounded growth
- **Network latency**: etcd members should be within 1ms RTT of each other
- **Event rate**: High watch/event rates can strain etcd → use watch caching in apiserver

---

## 4. kube-scheduler

Assigns pods to nodes based on resource requirements, constraints, and policies.

### Scheduling Workflow

```
       Unscheduled Pod (nodeName = "")
                │
                ▼
    ┌───────────────────────┐
    │   SCHEDULING QUEUE    │  Priority-sorted queue
    └───────────┬───────────┘
                ▼
    ┌───────────────────────┐
    │   FILTERING PHASE     │  Remove nodes that can't run the pod
    │                       │
    │ • PodFitsResources    │  CPU/memory capacity
    │ • PodFitsHostPorts    │  Port availability
    │ • NodeAffinity        │  Label matching
    │ • TaintToleration     │  Taint checks
    │ • VolumeBinding       │  Storage availability
    │ • TopologySpread      │  Distribution constraints
    └───────────┬───────────┘
                ▼
    ┌───────────────────────┐
    │   SCORING PHASE       │  Rank remaining nodes 0-100
    │                       │
    │ • LeastRequestedPriority  │  Prefer less loaded nodes
    │ • BalancedResourceAllocation │  CPU/mem balance
    │ • InterPodAffinity    │  Co-location scoring
    │ • NodeAffinity        │  Preference matching
    │ • ImageLocality       │  Prefer cached images
    └───────────┬───────────┘
                ▼
    ┌───────────────────────┐
    │   BINDING             │  Assign pod to highest-scoring node
    │   pod.spec.nodeName   │  API server write
    └───────────────────────┘
```

### Scheduling Framework (Plugin-Based since 1.19)

Extension points in the scheduling cycle:

| Extension Point | Purpose |
|----------------|---------|
| `QueueSort` | Order pods in scheduling queue |
| `PreFilter` | Pre-process/check pod info |
| `Filter` | Filter out infeasible nodes |
| `PostFilter` | Handle when no node is feasible (preemption) |
| `PreScore` | Pre-process info for scoring |
| `Score` | Rank feasible nodes |
| `Reserve` | Reserve resources on chosen node |
| `Permit` | Approve/deny/wait for binding |
| `PreBind` | Pre-binding tasks (volume provisioning) |
| `Bind` | Bind pod to node |
| `PostBind` | Post-binding cleanup |

### Custom Scheduling

```yaml
# Scheduler profile with custom plugins
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: storage-aware-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesFit
            weight: 2
          - name: VolumeBinding
            weight: 5  # Prioritize storage locality
```

---

## 5. kube-controller-manager

Runs core **control loops** (controllers) that watch cluster state via the API server and drive actual state toward desired state.

### Key Controllers

| Controller | What It Reconciles |
|-----------|-------------------|
| **Deployment** | Manages ReplicaSets for rolling updates |
| **ReplicaSet** | Ensures desired number of pod replicas |
| **StatefulSet** | Ordered pod creation/deletion with stable identity |
| **DaemonSet** | One pod per (matching) node |
| **Job** | Run-to-completion pods |
| **CronJob** | Scheduled job creation |
| **Node** | Monitors node health, sets conditions, evicts pods |
| **Endpoint** | Populates Endpoints for Services |
| **EndpointSlice** | Scalable endpoint management |
| **ServiceAccount** | Creates default SA in each namespace |
| **Namespace** | Cleans up all resources when namespace is deleted |
| **Garbage Collector** | Deletes orphaned objects via ownerReferences |
| **PV/PVC Binder** | Binds PersistentVolumeClaims to PersistentVolumes |

### Reconciliation Loop Pattern

```go
// Simplified controller reconciliation
func (c *Controller) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. OBSERVE — Get current state
    obj := &appsv1.Deployment{}
    if err := c.Get(ctx, req.NamespacedName, obj); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil // Object deleted, nothing to do
        }
        return ctrl.Result{}, err
    }

    // 2. DIFF — Compare desired vs actual
    desired := obj.Spec.Replicas
    actual := countRunningPods(obj)

    // 3. ACT — Take action to converge
    if actual < *desired {
        // Scale up — create more pods
        return ctrl.Result{}, c.scaleUp(ctx, obj, *desired-actual)
    } else if actual > *desired {
        // Scale down — delete excess pods
        return ctrl.Result{}, c.scaleDown(ctx, obj, actual-*desired)
    }

    return ctrl.Result{}, nil // Desired state reached
}
```

### Leader Election

When running multiple controller-manager replicas for HA, only one is active:

```yaml
# Leader election via Lease objects
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: master-1
  leaseDurationSeconds: 15
  renewTime: "2024-01-15T10:30:00Z"
```

---

## 6. cloud-controller-manager

Separates cloud-specific logic from core Kubernetes controllers.

### Responsibilities

| Controller | Cloud Function |
|-----------|---------------|
| **Node** | Initialize node with cloud metadata (zone, instance type) |
| **Route** | Configure cloud network routes for pod CIDR |
| **Service** | Provision cloud load balancers for type LoadBalancer |

### Why Separate?

- Core K8s doesn't need cloud SDK dependencies
- Cloud providers maintain their own controller code
- Enables faster release cycles for cloud integrations
- On-premises deployments don't run cloud-controller-manager

---

## 7. kubelet

Node agent running on every worker node. Ensures containers described in PodSpecs are running and healthy.

### Responsibilities

```
┌─────────────────────────────────────────────────┐
│                   kubelet                        │
│                                                  │
│  ┌─────────────┐  ┌──────────────┐              │
│  │ Pod Manager │  │ Volume Mgr   │              │
│  │ (PLEG)      │  │ (mount/umount)│             │
│  └──────┬──────┘  └──────┬───────┘              │
│         │                │                       │
│  ┌──────┴──────┐  ┌──────┴───────┐              │
│  │ Container   │  │ CSI Node     │              │
│  │ Runtime     │  │ (staging,    │              │
│  │ Interface   │  │  publishing) │              │
│  │ (CRI)       │  └──────────────┘              │
│  └──────┬──────┘                                 │
│         │ gRPC                                   │
│  ┌──────┴──────┐                                 │
│  │ containerd  │  or CRI-O                       │
│  └──────┬──────┘                                 │
│         │                                        │
│  ┌──────┴──────┐                                 │
│  │   runc      │  Low-level runtime (OCI)        │
│  └─────────────┘                                 │
└─────────────────────────────────────────────────┘
```

### Pod Lifecycle Event Generator (PLEG)

```
PLEG syncs every 1 second:
1. List all containers on node (via CRI)
2. Compare with last known state
3. Generate events: ContainerStarted, ContainerDied, etc.
4. Update pod status in API server
```

### Node Status Reporting

```yaml
# kubelet reports node status periodically
status:
  conditions:
    - type: Ready
      status: "True"
    - type: MemoryPressure
      status: "False"
    - type: DiskPressure
      status: "False"
    - type: PIDPressure
      status: "False"
  capacity:
    cpu: "8"
    memory: 32Gi
    pods: "110"
  allocatable:    # capacity minus reservations
    cpu: "7500m"
    memory: 30Gi
    pods: "110"
```

### Eviction Thresholds

```bash
# Default soft eviction thresholds
--eviction-soft=memory.available<500Mi,nodefs.available<10%
--eviction-soft-grace-period=memory.available=1m30s
# Hard eviction (immediate)
--eviction-hard=memory.available<100Mi,nodefs.available<5%
```

Eviction order: BestEffort → Burstable → Guaranteed (by QoS class)

---

## 8. kube-proxy

Implements Kubernetes **Service** networking on each node.

### Modes

| Mode | Mechanism | Performance | Use Case |
|------|-----------|-------------|----------|
| **iptables** (default) | NAT rules per service/endpoint | O(n) rule matching | Most clusters |
| **IPVS** | Linux Virtual Server (hash table) | O(1) lookup | Large clusters (10K+ services) |
| **nftables** (K8s 1.29+) | nftables rules | Better than iptables | Future default |

### iptables Mode

```
Client Pod (10.244.1.5) → Service ClusterIP (10.96.0.10:80)
        │
        ▼  (DNAT via iptables)
┌─────────────────────────────────────┐
│ -A KUBE-SERVICES -d 10.96.0.10/32  │
│   -p tcp --dport 80                │
│   -j KUBE-SVC-XXXXX                │
│                                     │
│ -A KUBE-SVC-XXXXX                  │
│   -m statistic --mode random       │
│     --probability 0.333            │
│   -j KUBE-SEP-AAAA  (10.244.1.10) │
│                                     │
│ -A KUBE-SVC-XXXXX                  │
│   -m statistic --mode random       │
│     --probability 0.500            │
│   -j KUBE-SEP-BBBB  (10.244.2.20) │
│                                     │
│ -A KUBE-SVC-XXXXX                  │
│   -j KUBE-SEP-CCCC  (10.244.3.30) │
└─────────────────────────────────────┘
```

### IPVS Mode

```bash
# View IPVS rules
ipvsadm -Ln
# Output:
# TCP  10.96.0.10:80 rr
#   -> 10.244.1.10:8080   Masq    1
#   -> 10.244.2.20:8080   Masq    1
#   -> 10.244.3.30:8080   Masq    1

# IPVS supports multiple LB algorithms:
# rr (round-robin), lc (least connections), 
# dh (destination hashing), sh (source hashing)
```

---

## 9. Container Runtimes

### CRI (Container Runtime Interface)

```
kubelet ──(gRPC)──→ CRI Runtime ──→ OCI Runtime ──→ Container
                    (containerd)     (runc)
```

### Runtime Comparison

| Runtime | Type | Description |
|---------|------|-------------|
| **containerd** | High-level | Industry standard, used by Docker, EKS, AKS, GKE |
| **CRI-O** | High-level | Built specifically for Kubernetes (Red Hat/OpenShift) |
| **runc** | Low-level | OCI reference implementation (Go) |
| **crun** | Low-level | OCI implementation in C (faster startup) |
| **gVisor (runsc)** | Sandbox | User-space kernel for security isolation |
| **Kata Containers** | VM-based | Lightweight VM per pod for strong isolation |

### containerd Architecture

```
              kubelet
                │ CRI gRPC
                ▼
         ┌──────────────┐
         │  containerd   │
         │               │
         │  ┌─────────┐  │    ┌────────────────┐
         │  │snapshotter│──┤──→│ overlay/btrfs   │  (image layers)
         │  └─────────┘  │    └────────────────┘
         │               │
         │  ┌─────────┐  │    ┌────────────────┐
         │  │ task     │──┤──→│ runc/crun       │  (container lifecycle)
         │  └─────────┘  │    └────────────────┘
         │               │
         │  ┌─────────┐  │    ┌────────────────┐
         │  │ content  │──┤──→│ image store     │  (OCI images)
         │  └─────────┘  │    └────────────────┘
         └──────────────┘
```

---

## 10. API Request Flow — Full Trace

**What happens when you run `kubectl apply -f deployment.yaml`?**

```
Step 1: kubectl reads deployment.yaml, determines API group/version
        (apps/v1 Deployment)

Step 2: kubectl sends HTTPS POST/PUT to API server
        POST /apis/apps/v1/namespaces/default/deployments

Step 3: API Server Authentication
        - Validates client certificate / bearer token
        - Identifies user (e.g., system:admin)

Step 4: API Server Authorization
        - RBAC check: Can user create Deployment in default namespace?

Step 5: Mutating Admission Webhooks
        - Inject defaults (e.g., add default service account)
        - Istio sidecar injector adds envoy container

Step 6: Schema Validation
        - Validate against OpenAPI schema

Step 7: Validating Admission Webhooks
        - Policy enforcement (e.g., OPA/Gatekeeper: must have resource limits)

Step 8: Persist to etcd
        - API server writes Deployment object to etcd
        - etcd replicates via Raft to followers

Step 9: Deployment Controller (watching Deployments) gets notified
        - Creates ReplicaSet object

Step 10: ReplicaSet Controller (watching ReplicaSets) gets notified
         - Creates Pod objects (with nodeName = "")

Step 11: Scheduler (watching unscheduled Pods) gets notified
         - Runs filter → score → bind
         - Sets pod.spec.nodeName = "worker-2"

Step 12: kubelet on worker-2 (watching Pods assigned to its node)
         - Pulls image (containerd → registry)
         - Creates container via CRI → runc
         - Sets up networking (CNI plugin)
         - Mounts volumes (CSI plugin)
         - Starts container
         - Runs startup/liveness/readiness probes

Step 13: kubelet reports Pod status back to API server
         - Pod phase: Running
         - Container state: Running

Step 14: Endpoint controller adds pod IP to Service endpoints
         - kube-proxy updates iptables/IPVS rules
```

---

## 11. Control Loops / Reconciliation Pattern

### Level-Triggered vs Edge-Triggered

| Approach | Description | K8s Choice |
|----------|-------------|------------|
| **Edge-triggered** | React only to events (changes) | ✗ |
| **Level-triggered** | Continuously check current state | ✓ |

Kubernetes uses **level-triggered** reconciliation: controllers periodically re-sync even without new events. This makes the system **self-healing** — if a controller crashes and restarts, it catches up by observing current state, not by replaying events.

### Reconciliation Example: ReplicaSet Controller

```go
func reconcileReplicaSet(rs *appsv1.ReplicaSet) error {
    // 1. List current pods matching RS selector
    pods := listPodsBySelector(rs.Spec.Selector)
    activePods := filterActivePods(pods)
    
    // 2. Compare with desired count
    diff := int(*rs.Spec.Replicas) - len(activePods)
    
    // 3. Take action
    if diff > 0 {
        // Need more pods — create them
        for i := 0; i < diff; i++ {
            createPod(rs.Spec.Template, rs)
        }
    } else if diff < 0 {
        // Too many pods — delete excess
        podsToDelete := selectPodsToDelete(activePods, -diff)
        for _, pod := range podsToDelete {
            deletePod(pod)
        }
    }
    
    // 4. Update status
    rs.Status.Replicas = int32(len(activePods))
    rs.Status.ReadyReplicas = countReady(activePods)
    updateStatus(rs)
    
    return nil
}
```

### Watch + Informer Pattern

```
                API Server
                    │
                    │  Watch stream (HTTP/2 chunked)
                    ▼
            ┌───────────────┐
            │   Reflector   │  Makes LIST + WATCH calls
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │ Delta FIFO    │  Queue of Add/Update/Delete events
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │   Indexer     │  In-memory cache (thread-safe store)
            │  (Local Cache)│  Indexed by namespace/name
            └───────┬───────┘
                    │
                    ▼
         ┌────────────────────┐
         │  Event Handlers    │  OnAdd / OnUpdate / OnDelete
         │  (enqueue key)     │
         └────────┬───────────┘
                  │
                  ▼
         ┌────────────────────┐
         │    Work Queue      │  Rate-limited, deduplicating
         └────────┬───────────┘
                  │
                  ▼
         ┌────────────────────┐
         │   Reconcile()      │  Process one key at a time
         │   (Business Logic) │  Reads from cache, writes to API
         └────────────────────┘
```

### SharedInformerFactory

```go
// Efficient resource watching — shares watches across controllers
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)

// Get informer for Pods
podInformer := factory.Core().V1().Pods()
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { enqueue(obj) },
    UpdateFunc: func(old, new interface{}) { enqueue(new) },
    DeleteFunc: func(obj interface{}) { enqueue(obj) },
})

// Start all informers
factory.Start(stopCh)

// Wait for cache sync before processing
factory.WaitForCacheSync(stopCh)
```

---

## 12. Component Communication Matrix

| From | To | Protocol | Purpose |
|------|----|----------|---------|
| kubectl | API server | HTTPS (REST) | User commands |
| API server | etcd | gRPC (TLS) | State persistence |
| Scheduler | API server | HTTPS (watch) | Pod binding |
| Controller manager | API server | HTTPS (watch) | Reconciliation |
| kubelet | API server | HTTPS | Pod status, spec watches |
| kubelet | Container runtime | gRPC (CRI) | Container lifecycle |
| kubelet | CSI driver | gRPC (CSI) | Volume operations |
| kube-proxy | API server | HTTPS (watch) | Service/endpoint updates |
| CoreDNS | API server | HTTPS (watch) | DNS record updates |

---

## 13. High Availability Architecture

```
                    Load Balancer (L4)
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │ API Server │ │ API Server │ │ API Server │
    │  Master-1  │ │  Master-2  │ │  Master-3  │
    ├────────────┤ ├────────────┤ ├────────────┤
    │ Controller │ │ Controller │ │ Controller │
    │ (leader)   │ │ (standby)  │ │ (standby)  │
    ├────────────┤ ├────────────┤ ├────────────┤
    │ Scheduler  │ │ Scheduler  │ │ Scheduler  │
    │ (standby)  │ │ (leader)   │ │ (standby)  │
    └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
          │              │              │
    ┌─────┴──────┐ ┌─────┴──────┐ ┌─────┴──────┐
    │   etcd-1   │ │   etcd-2   │ │   etcd-3   │
    │  (leader)  │ │ (follower) │ │ (follower) │
    └────────────┘ └────────────┘ └────────────┘
    
Stacked etcd topology — etcd co-located with control plane
External etcd topology — dedicated etcd cluster (larger setups)
```

### Split-Brain Prevention

- etcd uses Raft — requires majority quorum
- Controller-manager and scheduler use Lease-based leader election
- API server is stateless — all instances can handle requests
- Load balancer distributes across API server instances

---

## Interview Questions

1. **What happens if the API server goes down?** Running workloads continue (kubelet keeps containers running). No new changes can be applied. Controller reconciliation pauses.

2. **What happens if etcd loses quorum?** API server returns errors for all reads/writes. Existing pods keep running but no new scheduling or changes occur. Recovery requires restoring etcd quorum.

3. **Why does Kubernetes use a declarative model instead of imperative?** Declarative models are idempotent, self-healing, and composable. Controllers can always converge to desired state regardless of intermediate failures.

4. **Explain the informer pattern. Why doesn't every controller make direct API calls?** Informers provide a shared local cache with watch-based updates, reducing API server load. The work queue deduplicates events and provides rate limiting.

5. **How does kubelet handle a situation where the API server is unreachable?** Kubelet continues running existing pods using its local cache. It retries API server connection with exponential backoff. Pod management continues based on last-known state.

6. **Why 3 or 5 etcd nodes? What about 4 or 6?** Odd numbers maximize fault tolerance per node. 3 nodes tolerate 1 failure (quorum=2). 4 nodes also tolerate only 1 failure (quorum=3), wasting a node.

---

*Next: [Chapter 2 — Workload Resources](WorkloadResources.md)*
