# Chapter 7 — Scheduling & Resource Management

## 1. Scheduling Framework

```
    Unscheduled Pod (nodeName = "")
              │
              ▼
┌──────────────────────────────────┐
│     SCHEDULING CYCLE             │
│                                  │
│  1. Sort (QueueSort)             │
│     Priority-based pod ordering  │
│                                  │
│  2. Filter (PreFilter → Filter)  │
│     Remove infeasible nodes      │
│     • PodFitsResources           │
│     • PodFitsHostPorts           │
│     • NodeSelector/Affinity      │
│     • TaintToleration            │
│     • VolumeBinding              │
│     • TopologySpreadConstraints  │
│                                  │
│  3. PostFilter                   │
│     (if no node found → try      │
│      preempting lower-priority)  │
│                                  │
│  4. Score (PreScore → Score)     │
│     Rank feasible nodes 0-100   │
│     • LeastRequestedPriority    │
│     • BalancedResourceAlloc     │
│     • InterPodAffinity          │
│     • ImageLocality             │
│     • NodeAffinity preference   │
│                                  │
│  5. Reserve                      │
│     Temporarily claim resources  │
│                                  │
│  6. Permit                       │
│     Approve/deny/wait           │
│                                  │
│  7. Bind (PreBind → Bind)       │
│     Set pod.spec.nodeName       │
└──────────────────────────────────┘
```

---

## 2. nodeSelector

Simplest constraint: match node labels exactly.

```yaml
spec:
  nodeSelector:
    storage-access: "true"            # Node must have this label
    topology.kubernetes.io/zone: us-east-1a
```

```bash
# Label a node
kubectl label node worker-3 storage-access=true

# List node labels
kubectl get nodes --show-labels
```

---

## 3. Node Affinity

More expressive than nodeSelector — supports operators and soft preferences.

```yaml
spec:
  affinity:
    nodeAffinity:
      # HARD requirement: must be zone a or b
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a", "us-east-1b"]
              - key: node.kubernetes.io/instance-type
                operator: NotIn
                values: ["t3.small"]
      
      # SOFT preference: prefer nodes with SSD
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80    # 1-100 — higher = stronger preference
          preference:
            matchExpressions:
              - key: disk-type
                operator: In
                values: ["ssd", "nvme"]
        - weight: 20
          preference:
            matchExpressions:
              - key: cpu-gen
                operator: In
                values: ["latest"]
```

### Operators

| Operator | Match |
|----------|-------|
| `In` | Value is in list |
| `NotIn` | Value not in list |
| `Exists` | Key exists (any value) |
| `DoesNotExist` | Key doesn't exist |
| `Gt` | Value > (numeric) |
| `Lt` | Value < (numeric) |

---

## 4. Pod Affinity & Anti-Affinity

Place pods relative to **other pods**, not just nodes.

```yaml
spec:
  affinity:
    # Pod Affinity: co-locate with related pods
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["cache"]
          topologyKey: kubernetes.io/hostname
          # Same node as cache pods
    
    # Pod Anti-Affinity: spread apart
    podAntiAffinity:
      # Hard: never on same node
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["etcd"]
          topologyKey: kubernetes.io/hostname
          # No two etcd pods on same node
      
      # Soft: prefer different zones
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values: ["etcd"]
            topologyKey: topology.kubernetes.io/zone
```

### topologyKey

| topologyKey | Scope |
|-------------|-------|
| `kubernetes.io/hostname` | Per node |
| `topology.kubernetes.io/zone` | Per availability zone |
| `topology.kubernetes.io/region` | Per region |

---

## 5. Taints and Tolerations

**Taints**: nodes repel pods. **Tolerations**: pods can tolerate taints.

```bash
# Taint a node
kubectl taint nodes worker-1 gpu=true:NoSchedule
kubectl taint nodes worker-2 dedicated=storage:NoSchedule
kubectl taint nodes master-1 node-role.kubernetes.io/control-plane:NoSchedule

# Remove taint
kubectl taint nodes worker-1 gpu=true:NoSchedule-
```

### Toleration in Pod

```yaml
spec:
  tolerations:
    # Exact match
    - key: dedicated
      operator: Equal
      value: storage
      effect: NoSchedule
    
    # Key exists (any value)
    - key: gpu
      operator: Exists
      effect: NoSchedule
    
    # Tolerate control plane
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    
    # Tolerate NoExecute with timeout
    - key: node.kubernetes.io/not-ready
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 300    # Evict after 5 min if node not ready
```

### Taint Effects

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | Don't schedule new pods (existing stay) |
| `PreferNoSchedule` | Try not to schedule (soft) |
| `NoExecute` | Evict existing pods + don't schedule new |

### Use Case: Dedicated Nodes for Storage

```bash
# Mark nodes for storage workloads only
kubectl taint nodes storage-1 dedicated=storage:NoSchedule
kubectl taint nodes storage-2 dedicated=storage:NoSchedule
kubectl label nodes storage-1 storage-2 node-type=storage

# Only CSI pods can run on these nodes
spec:
  nodeSelector:
    node-type: storage
  tolerations:
    - key: dedicated
      value: storage
      effect: NoSchedule
```

---

## 6. Topology Spread Constraints

Control how pods are spread across failure domains.

```yaml
spec:
  topologySpreadConstraints:
    # Spread across zones: max 1 difference between zones
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule     # or ScheduleAnyway
      labelSelector:
        matchLabels:
          app: storage-api
      matchLabelKeys: ["pod-template-hash"]  # K8s 1.27+
      # matchLabelKeys ensures spread is per Deployment revision
      
    # Also spread within each zone across nodes
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: storage-api
```

### Spread Behavior

```
3 zones, 6 replicas, maxSkew=1:
  zone-a: 2 pods
  zone-b: 2 pods
  zone-c: 2 pods  ✓ (max difference = 0)

Add 1 more:
  zone-a: 3 pods
  zone-b: 2 pods
  zone-c: 2 pods  ✓ (max difference = 1)

Try to schedule in zone-a again:
  zone-a: 4 pods
  zone-b: 2 pods  ✗ (skew = 2 > maxSkew of 1)
  → Scheduled in zone-b or zone-c instead
```

---

## 7. Pod Priority and Preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: storage-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Storage infrastructure — must not be evicted"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 100
preemptionPolicy: Never    # Don't preempt others
```

### Preemption Flow

```
High-priority Pod (P=1000000) can't be scheduled
    │
    ▼
PostFilter: find nodes where preemption would work
    │
    ▼
Select victim pods (lowest priority first)
    │
    ▼
Nominate node → set pod.status.nominatedNodeName
    │
    ▼
Delete victim pods (graceful termination)
    │
    ▼
Reschedule high-priority pod on freed node
```

---

## 8. Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: storage-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: storage-api
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
    # CPU-based
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Target 70% CPU
    
    # Memory-based
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 500Mi
    
    # Custom metric (from Prometheus)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
    
    # External metric
    - type: External
      external:
        metric:
          name: queue_depth
          selector:
            matchLabels:
              queue: storage-jobs
        target:
          type: Value
          value: "50"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100      # Double pods
          periodSeconds: 60
        - type: Pods
          value: 4        # Or add 4 pods
          periodSeconds: 60
      selectPolicy: Max   # Use whichever allows more scaling
    
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scale-down
      policies:
        - type: Percent
          value: 10       # Remove 10% at a time
          periodSeconds: 60
```

### HPA Algorithm

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))

Example: 
  Current: 5 replicas, CPU utilization = 90%
  Target: 70%
  Desired = ceil(5 × (90/70)) = ceil(6.43) = 7 replicas
```

---

## 9. Vertical Pod Autoscaler (VPA)

Right-size resource requests automatically.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: storage-api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: storage-api
  
  updatePolicy:
    updateMode: Auto       # Off, Initial, Recreate, Auto
    # Off: recommendations only (view with kubectl)
    # Initial: apply only on pod creation
    # Auto: evict and recreate pods with new resources
  
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "4"
          memory: 4Gi
        controlledResources: ["cpu", "memory"]
```

**Note**: VPA and HPA should not target the same resource metric (e.g., both on CPU). Use VPA for resource sizing and HPA for replica count on different metrics.

---

## 10. Cluster Autoscaler

Scale node pools based on pending pods.

```
Pod unschedulable (insufficient resources)
        │
        ▼
Cluster Autoscaler detects pending pods
        │
        ▼
Simulate scheduling on each node group
        │
        ▼
Scale up smallest adequate node group
        │
        ▼
Wait for node to be Ready
        │
        ▼
Scheduler places pod on new node

=== Scale Down ===

Node utilization < 50% for 10 minutes
        │
        ▼
Check if pods can be moved elsewhere
(respect PDB, pod anti-affinity, local storage)
        │
        ▼
Cordon node → drain pods → delete node
```

---

## 11. KEDA (Event-Driven Autoscaling)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: storage-events
        topic: volume-events
        lagThreshold: "100"    # Scale when lag > 100
    
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: storage_api_queue_depth
        threshold: "50"
        query: |
          sum(storage_api_queue_depth{namespace="production"})
```

---

## Interview Questions

1. **How does the Kubernetes scheduler work?**
   - Two phases: Filter (remove infeasible nodes by resource, affinity, taints, topology) → Score (rank remaining nodes by resource balance, affinity, image locality). Highest-scoring node wins. Plugin-based framework allows custom scheduling logic.

2. **Explain taints and tolerations with a use case.**
   - Taints on nodes repel pods (NoSchedule, PreferNoSchedule, NoExecute). Tolerations on pods allow scheduling on tainted nodes. Use case: dedicated storage nodes tainted with `dedicated=storage:NoSchedule` — only CSI pods with matching toleration run there.

3. **How does HPA work? What happens if scaling isn't fast enough?**
   - HPA monitors metrics (CPU, memory, custom) every 15s. Calculates desired replicas using ratio formula. Respects stabilization windows and scaling policies. If scaling isn't fast enough: adjust behavior policies, use KEDA for faster event-driven scaling, or pre-scale with scheduled scaling.

4. **What is topology spread constraint?**
   - Controls pod distribution across failure domains (zones, nodes). maxSkew defines max difference in pod count between domains. Essential for HA — ensures pods survive zone failures.

5. **Pod priority and preemption — how it works?**
   - PriorityClass assigns numeric priority. When high-priority pod can't be scheduled, scheduler identifies lower-priority victim pods to evict. Victims are gracefully terminated, then high-priority pod is placed. PreemptionPolicy: Never prevents a priority class from preempting others.

---

*Next: [Chapter 8 — Operators & Custom Resources](OperatorsAndCRDs.md)*
