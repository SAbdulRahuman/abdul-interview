# Design a Container Orchestration System

Examples: Kubernetes, Docker Swarm, Nomad

---

## 1. Requirements

### Functional
- Deploy containerised applications (pods/tasks)
- Auto-scale based on CPU/memory/custom metrics
- Service discovery and internal DNS
- Rolling updates and rollback
- Health checks and self-healing (restart failed containers)
- Resource quotas and namespace isolation

### Non-Functional
- Schedule thousands of containers across hundreds of nodes
- Sub-second scheduling latency
- High availability of control plane
- Extensible (custom controllers, operators)
- Multi-tenancy support

---

## 2. High-Level Architecture

```
  ┌────────────────────────────────────────────────────────┐
  │                    Control Plane                        │
  │                                                        │
  │  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐ │
  │  │ API      │  │  Scheduler   │  │  Controller      │ │
  │  │ Server   │  │              │  │  Manager         │ │
  │  │ (REST +  │  │  (bin-pack   │  │  (reconciliation │ │
  │  │  watch)  │  │   pods to    │  │   loops)         │ │
  │  │          │  │   nodes)     │  │  • ReplicaSet    │ │
  │  └────┬─────┘  └──────┬───────┘  │  • Deployment    │ │
  │       │               │          │  • Service        │ │
  │       │               │          │  • HPA            │ │
  │  ┌────▼───────────────▼──────────┴──────────────────┐ │
  │  │              etcd (Raft consensus)                │ │
  │  │     Cluster state: desired + actual state         │ │
  │  └──────────────────────────────────────────────────┘ │
  └────────────────────────────────────────────────────────┘
          │                    │                    │
  ┌───────▼───────┐   ┌───────▼───────┐   ┌───────▼───────┐
  │   Worker Node  │   │   Worker Node  │   │   Worker Node  │
  │                │   │                │   │                │
  │  ┌──────────┐  │   │  ┌──────────┐  │   │  ┌──────────┐  │
  │  │  Kubelet  │  │   │  │  Kubelet  │  │   │  │  Kubelet  │  │
  │  │ (node    │  │   │  │          │  │   │  │          │  │
  │  │  agent)  │  │   │  └──────────┘  │   │  └──────────┘  │
  │  └──────────┘  │   │  ┌──────────┐  │   │  ┌──────────┐  │
  │  ┌──────────┐  │   │  │ kube-    │  │   │  │ kube-    │  │
  │  │ kube-    │  │   │  │ proxy    │  │   │  │ proxy    │  │
  │  │ proxy    │  │   │  └──────────┘  │   │  └──────────┘  │
  │  └──────────┘  │   │  ┌──────────┐  │   │  ┌──────────┐  │
  │  ┌──────────┐  │   │  │Container │  │   │  │Container │  │
  │  │Container │  │   │  │ Runtime  │  │   │  │ Runtime  │  │
  │  │ Runtime  │  │   │  │(containerd)│  │  │(containerd)│  │
  │  └──────────┘  │   │  └──────────┘  │   │  └──────────┘  │
  │                │   │                │   │                │
  │  [Pod A][Pod B]│   │  [Pod C][Pod D]│   │  [Pod E][Pod F]│
  └────────────────┘   └────────────────┘   └────────────────┘
```

---

## 3. Scheduling Algorithm

```
  Pod scheduling pipeline:

  ┌─────────────────────────────────────────────────────┐
  │  1. FILTERING (find feasible nodes)                 │
  │                                                     │
  │  Pod needs: 2 CPU, 4 GB RAM, GPU, zone=us-east     │
  │                                                     │
  │  Node A: 8 CPU, 32 GB, GPU, us-east    ✓ feasible  │
  │  Node B: 4 CPU, 8 GB, no GPU           ✗ filtered  │
  │  Node C: 1 CPU, 16 GB, GPU             ✗ filtered  │
  │  Node D: 4 CPU, 16 GB, GPU, us-east    ✓ feasible  │
  ├─────────────────────────────────────────────────────┤
  │  2. SCORING (rank feasible nodes)                   │
  │                                                     │
  │  Node A score: resource_balance=70 + spread=80 = 150│
  │  Node D score: resource_balance=85 + spread=60 = 145│
  │                                                     │
  │  Scoring factors:                                   │
  │  • Least requested resources (spread load)          │
  │  • Pod anti-affinity (spread replicas)              │
  │  • Data locality (near persistent volumes)          │
  │  • Topology spread constraints                      │
  ├─────────────────────────────────────────────────────┤
  │  3. BIND: Assign pod to Node A (highest score)      │
  └─────────────────────────────────────────────────────┘
```

---

## 4. Reconciliation Loop (Controllers)

```
  Declarative model: desired state vs actual state

  User declares: "I want 3 replicas of nginx"

  ┌──────────────────────────────────────────────┐
  │  Controller loop (every few seconds):        │
  │                                              │
  │  desired_replicas = 3                        │
  │  actual_replicas = count(running nginx pods) │
  │                                              │
  │  if actual < desired:                        │
  │      create (desired - actual) pods          │
  │  if actual > desired:                        │
  │      delete (actual - desired) pods          │
  │  if actual == desired:                       │
  │      no action (converged)                   │
  │                                              │
  │  Self-healing:                               │
  │  Pod crashes → kubelet reports → actual=2    │
  │  Controller notices 2 < 3 → creates new pod │
  └──────────────────────────────────────────────┘
```

---

## 5. Networking Model

```
  ┌─────────────────────────────────────────────────────┐
  │  Three networking requirements:                      │
  │                                                     │
  │  1. Pod-to-Pod (any pod can reach any pod)          │
  │     ┌──────┐        ┌──────┐                        │
  │     │Pod A │───────▶│Pod B │  (flat network,        │
  │     │10.1.1│        │10.1.2│   no NAT)              │
  │     └──────┘        └──────┘                        │
  │                                                     │
  │  2. Service (stable IP for pod group)               │
  │     ┌──────────────┐                                │
  │     │  Service     │  ClusterIP: 10.96.0.1          │
  │     │  "nginx-svc" │──▶ Pod A (10.1.1.5)           │
  │     │              │──▶ Pod B (10.1.1.6)           │
  │     │              │──▶ Pod C (10.1.2.3)           │
  │     └──────────────┘  kube-proxy → iptables/IPVS   │
  │                                                     │
  │  3. Ingress (external → service)                    │
  │     External LB → Ingress Controller → Service      │
  │     Route by host/path: api.example.com/v1 → svc   │
  └─────────────────────────────────────────────────────┘
```

---

## 6. Rolling Update

```
  Deployment: nginx, replicas=3, maxSurge=1, maxUnavailable=0

  Step 1: Create 1 new pod (v2)
  [v1] [v1] [v1] [v2-starting]

  Step 2: v2 passes health check → terminate 1 v1
  [v1] [v1] [v2] [v2-starting]

  Step 3: Continue...
  [v1] [v2] [v2] [v2-starting]

  Step 4: Complete
  [v2] [v2] [v2]

  Rollback: kubectl rollout undo → reverts to previous ReplicaSet
  History preserved for N revisions
```

---

## 7. Auto-Scaling

```
  ┌────────────────────────────────────────────────────┐
  │  Horizontal Pod Autoscaler (HPA):                   │
  │                                                    │
  │  Target: CPU utilisation = 70%                      │
  │  Current: 3 pods × 90% CPU = 270% total            │
  │  Desired: 270% / 70% = 3.86 → ceil → 4 pods       │
  │                                                    │
  │  Scale-up: immediate                               │
  │  Scale-down: wait 5 min (cooldown to avoid flapping)│
  ├────────────────────────────────────────────────────┤
  │  Cluster Autoscaler:                                │
  │                                                    │
  │  Pending pods (unschedulable) → add node            │
  │  Underutilised nodes (< 50%) → drain and remove   │
  │  Respects PodDisruptionBudgets                      │
  └────────────────────────────────────────────────────┘
```

---

## 8. Fault Tolerance

- **etcd:** 3 or 5 node Raft cluster → survives minority failures
- **API server:** Stateless, run multiple behind load balancer
- **Scheduler/Controller:** Leader election, only one active
- **Node failure:** Pods rescheduled after `node-not-ready` timeout (5 min default)
- **Pod failure:** Kubelet restarts (CrashLoopBackoff); controller creates replacement

---

---

## 9. Low-Level Design (LLD)

### API Contract (Kubernetes API Server)
```
  Core Resources:
  POST   /api/v1/namespaces/{ns}/pods            Create pod
  GET    /api/v1/namespaces/{ns}/pods/{name}      Get pod
  PUT    /api/v1/namespaces/{ns}/pods/{name}      Update pod
  DELETE /api/v1/namespaces/{ns}/pods/{name}      Delete pod
  GET    /api/v1/namespaces/{ns}/pods?watch=true  Watch for changes (SSE)
  PATCH  /api/v1/namespaces/{ns}/pods/{name}      Strategic merge patch

  Workloads:
  POST   /apis/apps/v1/namespaces/{ns}/deployments
  POST   /apis/apps/v1/namespaces/{ns}/statefulsets
  POST   /apis/batch/v1/namespaces/{ns}/jobs

  Custom Resources:
  POST   /apis/{group}/{version}/namespaces/{ns}/{resource}
```

### etcd Data Model
```
  Key Structure:
  /registry/pods/{namespace}/{name}             → Pod spec + status JSON
  /registry/deployments/{namespace}/{name}      → Deployment spec
  /registry/services/{namespace}/{name}         → Service spec
  /registry/secrets/{namespace}/{name}          → Encrypted secret data
  /registry/events/{namespace}/{name}           → Event object

  Watch Mechanism:
  ┌──────────────────────────────────────────────────────────┐
  │  Client → API Server: GET /api/v1/pods?watch=true       │
  │  API Server → etcd: Watch on /registry/pods/            │
  │                                                          │
  │  etcd revision: monotonically increasing int64           │
  │  Each mutation: revision++, store old+new value          │
  │  Watch: stream all changes since revision X              │
  │  Bookmarks: periodic checkpoint of current revision      │
  └──────────────────────────────────────────────────────────┘
```

### Scheduler Decision Pipeline
```
  ┌────────────────────────────────────────────────────────┐
  │  1. Informer cache: watch pods with spec.nodeName=""   │
  │  2. Queue: priority queue sorted by pod priority       │
  │  3. Filter phase:                                      │
  │     - NodeResourcesFit (CPU/memory capacity)           │
  │     - PodTopologySpread (anti-affinity)                │
  │     - NodeAffinity (required/preferred)                │
  │     - TaintToleration                                  │
  │  4. Score phase:                                       │
  │     - LeastRequestedPriority (spread)                  │
  │     - MostRequestedPriority (bin-pack)                 │
  │     - BalancedResourceAllocation                       │
  │     - ImageLocality (prefer nodes with image cached)   │
  │  5. Bind: PATCH pod with spec.nodeName = selected node │
  │  6. kubelet: watch sees binding, pulls image, starts   │
  └────────────────────────────────────────────────────────┘
```

---

## 10. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Control Plane Scaling:                                 │
  │  • API Server: stateless, scale horizontally (3-7)      │
  │  • etcd: 3 or 5 members (Raft quorum)                   │
  │  • etcd limit: ~100K objects, ~8 GB data recommended    │
  │  • API priority & fairness: queue-based request limits  │
  │                                                         │
  │  Worker Node Scaling:                                   │
  │  • Cluster Autoscaler: 0 to 15,000 nodes               │
  │  • Scale-up: pending pods → add nodes (60-90s for cloud)│
  │  • Scale-down: utilization < 50% for 10 min → drain     │
  │  • Node pools: GPU, high-mem, ARM, spot instances       │
  │                                                         │
  │  Large Cluster Optimizations (5000+ nodes):             │
  │  • Watch cache in API server (avoid etcd overload)      │
  │  • etcd compaction every 5 min, defrag weekly           │
  │  • Namespace-scoped watches (avoid cluster-wide)        │
  │  • Limit LIST page size (--chunk-size=500)              │
  │  • Separate etcd clusters: events vs core objects       │
  │                                                         │
  │  Pod Scaling:                                           │
  │  • HPA: scale by CPU/memory/custom metrics              │
  │  • VPA: right-size requests/limits                      │
  │  • KEDA: event-driven scaling (queue depth, etc.)       │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Cluster State (etcd):                                  │
  │  • Raft consensus: committed only when majority ACK     │
  │  • WAL: every mutation written to WAL before apply      │
  │  • Snapshots: periodic full state snapshot (every 10K)  │
  │  • Backup: etcdctl snapshot save → stored in S3         │
  │  • Restore: rebuild cluster from snapshot               │
  │                                                         │
  │  Application Data:                                      │
  │  • PersistentVolumes: data survives pod restarts        │
  │  • StorageClass with reclaimPolicy=Retain               │
  │  • VolumeSnapshots: CSI snapshot before upgrades        │
  │  • StatefulSet: ordered graceful shutdown                │
  │                                                         │
  │  Secret Management:                                     │
  │  • etcd encryption at rest (EncryptionConfiguration)    │
  │  • External secrets sync (Vault, AWS SM)                │
  │  • Sealed Secrets: encrypted in Git, decrypted in-cluster│
  │                                                         │
  │  Config Changes:                                        │
  │  • GitOps: all manifests in Git (single source of truth)│
  │  • Argo CD / Flux: auto-sync from Git to cluster        │
  │  • kubectl diff before apply                            │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ API Server GET (single)      │ 5 ms     │ 50 ms   │
  │ API Server LIST (1K objects) │ 50 ms    │ 500 ms  │
  │ Pod scheduling (pending→bind)│ 100 ms   │ 5 s     │
  │ Pod startup (bind→running)   │ 5 s      │ 30 s    │
  │ Service DNS resolution       │ 1 ms     │ 5 ms    │
  │ HPA reaction time            │ 15 s     │ 60 s    │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Watch cache: avoid expensive etcd reads              │
  │  • Image pre-pulling (DaemonSet or warm pool)           │
  │  • Topology-aware routing: same-zone service calls      │
  │  • CoreDNS cache: reduce DNS lookup latency             │
  │  • Preemptible scheduling: evict low-priority pods      │
  │  • Ephemeral containers: fast debug without restart     │
  │  • Node-local DNS cache (NodeLocal DNSCache)            │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Reliability

```
  Failure Modes & Mitigation:
  ┌─────────────────────────┬────────────────────────────────┐
  │ Failure                 │ Mitigation                     │
  ├─────────────────────────┼────────────────────────────────┤
  │ Worker node crash       │ Node controller: mark NotReady │
  │                         │ after 40s; reschedule pods     │
  │                         │ after 5m (pod-eviction-timeout)│
  ├─────────────────────────┼────────────────────────────────┤
  │ API server crash        │ Multiple replicas behind LB;   │
  │                         │ kubelets cache last state      │
  ├─────────────────────────┼────────────────────────────────┤
  │ etcd member loss        │ Raft continues with majority;  │
  │                         │ replace member, learner mode   │
  ├─────────────────────────┼────────────────────────────────┤
  │ Network partition       │ Split-brain: isolated nodes    │
  │                         │ keep running pods but can't    │
  │                         │ receive updates; lease expiry  │
  ├─────────────────────────┼────────────────────────────────┤
  │ OOM kill (container)    │ Restart policy: Always;        │
  │                         │ resource limits prevent spread │
  ├─────────────────────────┼────────────────────────────────┤
  │ Bad deployment          │ Readiness probe fails → no     │
  │                         │ traffic; rollback with         │
  │                         │ kubectl rollout undo           │
  └─────────────────────────┴────────────────────────────────┘
```

---

## 14. Availability

```
  Target: 99.99% for control plane; 99.999% for running workloads

  ┌─────────────────────────────────────────────────────────┐
  │  Control Plane HA:                                      │
  │  • 3 API servers behind load balancer                    │
  │  • 3 or 5 etcd members across AZs                       │
  │  • 2 scheduler + 2 controller-manager (leader election) │
  │  • Managed K8s (EKS/GKE): 99.95% SLA for control plane │
  │                                                         │
  │  Workload Availability:                                 │
  │  • PodDisruptionBudgets: minAvailable during drain      │
  │  • Pod anti-affinity: spread replicas across nodes/AZs  │
  │  • Topology spread constraints: even distribution       │
  │  • Graceful shutdown: preStop hook + terminationGrace   │
  │                                                         │
  │  Multi-Cluster HA:                                      │
  │  • Federation or multi-cluster service mesh             │
  │  • DNS-based failover between clusters                   │
  │  • Cluster API: declarative cluster lifecycle            │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication:                                        │
  │  • X.509 client certificates (kubeconfig)               │
  │  • OIDC integration (corporate SSO)                     │
  │  • Service account tokens (bound, projected)            │
  │                                                         │
  │  Authorization:                                         │
  │  • RBAC: Role/ClusterRole → RoleBinding                 │
  │  • Principle of least privilege per namespace            │
  │  • Audit logging: all API requests logged               │
  │                                                         │
  │  Pod Security:                                          │
  │  • Pod Security Standards (restricted/baseline)         │
  │  • SecurityContext: runAsNonRoot, readOnlyRootFS        │
  │  • Seccomp profiles, AppArmor                           │
  │  • No privileged containers in production               │
  │                                                         │
  │  Network Security:                                      │
  │  • NetworkPolicies: default deny, allow-list            │
  │  • mTLS via service mesh for pod-to-pod                 │
  │  • Private API server endpoint (no public access)       │
  │                                                         │
  │  Supply Chain:                                          │
  │  • Image signing (cosign/Notary)                        │
  │  • Admission webhook: reject unsigned images            │
  │  • Image scanning in CI/CD (Trivy, Snyk)                │
  │  • Private registry with RBAC                           │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Compute (primary cost):                                │
  │  • Control plane: managed K8s ≈ $70-$200/month (EKS)    │
  │  • Worker nodes: EC2/GCE cost for N × instance type     │
  │  • Spot/Preemptible nodes: 60-70% savings               │
  │    - Use for stateless workloads, batch jobs             │
  │    - Fallback to on-demand for critical workloads        │
  │                                                         │
  │  Optimization Strategies:                               │
  │  • VPA: right-size resource requests (avoid over-prov)  │
  │  • Cluster autoscaler: scale nodes to 0 in dev/staging  │
  │  • Karpenter: per-pod instance selection, better packing│
  │  • Namespace resource quotas: prevent runaway costs      │
  │  • Cost allocation: labels + Kubecost per team/service  │
  │                                                         │
  │  Cost Estimate (200-node production cluster):           │
  │  • Control plane (EKS): $150/month                      │
  │  • 150 on-demand m5.2xl: ~$35K/month                    │
  │  • 50 spot m5.2xl: ~$4K/month                           │
  │  • Load balancers + NAT: ~$1K/month                     │
  │  • Total: ~$40K/month                                   │
  │  • With right-sizing + spot: ~$25K/month (37% saving)  │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How does the scheduler handle resource fragmentation?** — Bin-packing scoring, pod priorities/preemption, cluster autoscaler
2. **How do you achieve zero-downtime deployments?** — Rolling updates + readiness probes + PodDisruptionBudgets
3. **How does service discovery work?** — CoreDNS resolves `svc-name.namespace.svc.cluster.local` → ClusterIP → kube-proxy → pod IP
4. **How do you handle stateful workloads?** — StatefulSets (stable identity), persistent volumes, headless services
5. **etcd scalability?** — etcd is the bottleneck at large scale (>5000 nodes); watch mechanics, compaction, defragmentation
