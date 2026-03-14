# Chapter 2 — Workload Resources

## 1. Pod — Smallest Deployable Unit

A Pod is a group of one or more containers with shared network and storage.

### Pod Structure

```
┌──────────────────────────────────────────┐
│                   Pod                     │
│                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │  App    │  │ Sidecar │  │  Init   │  │
│  │Container│  │Container│  │Container│  │
│  │ :8080   │  │ (envoy) │  │ (setup) │  │
│  └────┬────┘  └────┬────┘  └─────────┘  │
│       │            │                     │
│  ┌────┴────────────┴────┐               │
│  │  Shared Network NS   │  (localhost)   │
│  │  IP: 10.244.1.5      │               │
│  └──────────────────────┘               │
│                                          │
│  ┌──────────────────────┐               │
│  │  Shared Volumes      │               │
│  │  (emptyDir, PVC)     │               │
│  └──────────────────────┘               │
│                                          │
│  cgroup: limits applied per container    │
└──────────────────────────────────────────┘
```

### Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-agent
  labels:
    app: storage-agent
    tier: data
spec:
  # Init containers run first, sequentially
  initContainers:
    - name: init-config
      image: busybox:1.36
      command: ['sh', '-c', 'cp /defaults/config.yaml /config/']
      volumeMounts:
        - name: config
          mountPath: /config

  containers:
    # Main application container
    - name: agent
      image: dell/storage-agent:v2.1
      ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
      resources:
        requests:
          cpu: 250m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 512Mi
      volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
        - name: data
          mountPath: /data
      
      # Probes
      startupProbe:
        httpGet:
          path: /healthz
          port: http
        failureThreshold: 30
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /healthz
          port: http
        periodSeconds: 15
        timeoutSeconds: 5
      readinessProbe:
        httpGet:
          path: /ready
          port: http
        periodSeconds: 5

    # Sidecar container (log shipper)
    - name: log-shipper
      image: fluent/fluent-bit:2.2
      volumeMounts:
        - name: data
          mountPath: /data
          readOnly: true

  volumes:
    - name: config
      emptyDir: {}
    - name: data
      persistentVolumeClaim:
        claimName: agent-data

  terminationGracePeriodSeconds: 60
  restartPolicy: Always
```

---

## 2. Init Containers

Run **before** application containers, one at a time, in order.

### Use Cases

| Use Case | Example |
|----------|---------|
| Wait for dependency | Wait for database to be ready |
| Configuration setup | Download config from remote source |
| Schema migration | Run database migrations |
| Security setup | Set file permissions on volumes |
| Data seeding | Pre-populate cache from S3 |

```yaml
initContainers:
  # Step 1: Wait for database
  - name: wait-db
    image: busybox:1.36
    command: ['sh', '-c', 
      'until nslookup postgres.default.svc.cluster.local; do sleep 2; done']
  
  # Step 2: Run migrations
  - name: migrate
    image: myapp:v1
    command: ['./migrate', '--up']
    env:
      - name: DB_URL
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: url
```

### Init Container Rules

- Run to completion before any app container starts
- If any init container fails → pod restarts (respecting restartPolicy)
- Run sequentially (step 1 finishes before step 2 starts)
- Don't support lifecycle hooks, liveness, or readiness probes
- Share volumes with app containers but have separate image and command

---

## 3. Sidecar Containers (K8s 1.28+ Native Support)

### Traditional vs Native Sidecar

```yaml
# Traditional sidecar — regular container, no ordering guarantees
containers:
  - name: app
    image: myapp:v1
  - name: envoy
    image: envoyproxy/envoy:v1.28

# Native sidecar (K8s 1.28+) — init container with restartPolicy: Always
initContainers:
  - name: envoy
    image: envoyproxy/envoy:v1.28
    restartPolicy: Always    # ← Key: runs alongside app containers
    ports:
      - containerPort: 15001
```

### Native Sidecar Benefits

- **Startup ordering**: Sidecar starts before app containers
- **Shutdown ordering**: Sidecar stops after app containers
- **Job compatibility**: Sidecar doesn't prevent Job completion
- **Probe support**: readiness/liveness probes on sidecars

### Common Sidecar Patterns

| Pattern | Example |
|---------|---------|
| **Service mesh proxy** | Envoy, Linkerd-proxy |
| **Log collection** | Fluent Bit, Filebeat |
| **Monitoring** | Prometheus exporter |
| **Security** | Vault agent (secret injection) |
| **Adapter** | Protocol translation |

---

## 4. ReplicaSet

Ensures a specified number of pod replicas are running at all times.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: storage-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: storage-worker
  template:
    metadata:
      labels:
        app: storage-worker
    spec:
      containers:
        - name: worker
          image: dell/storage-worker:v2
```

**Note**: You rarely create ReplicaSets directly. Deployments manage ReplicaSets.

### How ReplicaSet Reconciles

```
Desired = 3, Current = ?

Current = 1 → Create 2 more pods
Current = 3 → Do nothing
Current = 5 → Delete 2 pods (using deletion cost annotation)
Pod dies   → Controller notices, creates replacement
```

---

## 5. Deployment

Declarative updates for Pods and ReplicaSets. The **most common** workload resource.

### Deployment Architecture

```
┌──────────────────────────────────────────┐
│              Deployment                   │
│  replicas: 3                             │
│  strategy: RollingUpdate                 │
│                                          │
│  ┌─────────────────────────────────┐     │
│  │     ReplicaSet (current)        │     │
│  │     image: app:v2               │     │
│  │     replicas: 3                 │     │
│  │                                 │     │
│  │  [Pod v2] [Pod v2] [Pod v2]     │     │
│  └─────────────────────────────────┘     │
│                                          │
│  ┌─────────────────────────────────┐     │
│  │     ReplicaSet (old)            │     │
│  │     image: app:v1               │     │
│  │     replicas: 0  (scaled down)  │     │
│  └─────────────────────────────────┘     │
└──────────────────────────────────────────┘
```

### Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-api
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max pods above desired during update
      maxUnavailable: 1  # Max pods unavailable during update
  
  # Rolling update progression (6 replicas):
  # Step 1: 6 old + 2 new = 8 total (maxSurge=2)
  # Step 2: Kill 1 old → 5 old + 2 new
  # Step 3: New ready → 5 old + 2 new, create 1 more new
  # ... continues until all 6 are new
  
  revisionHistoryLimit: 5  # Keep 5 old ReplicaSets for rollback
  minReadySeconds: 30      # Wait 30s after ready before proceeding
  progressDeadlineSeconds: 600  # Fail if stuck for 10 min
```

### Deployment Operations

```bash
# Create deployment
kubectl apply -f deployment.yaml

# Watch rollout progress
kubectl rollout status deployment/storage-api

# Rollout history
kubectl rollout history deployment/storage-api
kubectl rollout history deployment/storage-api --revision=3

# Rollback
kubectl rollout undo deployment/storage-api                    # Previous version
kubectl rollout undo deployment/storage-api --to-revision=2    # Specific version

# Pause/Resume (batch changes)
kubectl rollout pause deployment/storage-api
kubectl set image deployment/storage-api app=app:v3
kubectl set resources deployment/storage-api -c app --limits=cpu=500m
kubectl rollout resume deployment/storage-api   # Triggers single rollout

# Scale
kubectl scale deployment storage-api --replicas=10

# Update image
kubectl set image deployment/storage-api app=dell/storage-api:v3
```

### Recreate Strategy

```yaml
strategy:
  type: Recreate  # Kill all old pods, then create all new pods
  # Use when: app doesn't support running two versions simultaneously
  # Drawback: downtime during update
```

---

## 6. StatefulSet

For **stateful** workloads requiring stable identity, ordered operations, and persistent storage.

### StatefulSet Guarantees

| Guarantee | Description |
|-----------|-------------|
| **Stable network identity** | `pod-0`, `pod-1`, `pod-2` (predictable names) |
| **Stable storage** | Each pod gets its own PVC (survives rescheduling) |
| **Ordered deployment** | Pods created 0 → 1 → 2 (parallel option available) |
| **Ordered deletion** | Pods deleted 2 → 1 → 0 |
| **Ordered rolling update** | Updated 2 → 1 → 0 (reverse order) |

### StatefulSet Architecture

```
┌──────────────────────────────────────────────────────┐
│                    StatefulSet                        │
│   serviceName: "etcd"                                │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ etcd-0   │  │ etcd-1   │  │ etcd-2   │           │
│  │          │  │          │  │          │           │
│  │ DNS:     │  │ DNS:     │  │ DNS:     │           │
│  │ etcd-0.  │  │ etcd-1.  │  │ etcd-2.  │           │
│  │ etcd.ns. │  │ etcd.ns. │  │ etcd.ns. │           │
│  │ svc.     │  │ svc.     │  │ svc.     │           │
│  │ cluster. │  │ cluster. │  │ cluster. │           │
│  │ local    │  │ local    │  │ local    │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │                │
│  ┌────┴─────┐  ┌────┴─────┐  ┌────┴─────┐           │
│  │ PVC-0    │  │ PVC-1    │  │ PVC-2    │           │
│  │ data-    │  │ data-    │  │ data-    │           │
│  │ etcd-0   │  │ etcd-1   │  │ etcd-2   │           │
│  └──────────┘  └──────────┘  └──────────┘           │
└──────────────────────────────────────────────────────┘
```

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
spec:
  serviceName: etcd   # Required: headless service for DNS
  replicas: 3
  podManagementPolicy: OrderedReady  # or Parallel
  
  selector:
    matchLabels:
      app: etcd
  
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
        - name: etcd
          image: quay.io/coreos/etcd:v3.5.11
          ports:
            - containerPort: 2379
              name: client
            - containerPort: 2380
              name: peer
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: INITIAL_CLUSTER
              value: "etcd-0=http://etcd-0.etcd:2380,etcd-1=http://etcd-1.etcd:2380,etcd-2=http://etcd-2.etcd:2380"
          volumeMounts:
            - name: data
              mountPath: /var/lib/etcd

  # volumeClaimTemplates creates a PVC per pod
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi

  # Update strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods (set >0 for canary)
      maxUnavailable: 1  # K8s 1.24+

  # PVC retention (K8s 1.27+)
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain    # Keep PVCs when StatefulSet is deleted
    whenScaled: Delete     # Delete PVCs when scaling down
```

### Headless Service for StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd    # Must match StatefulSet.spec.serviceName
spec:
  clusterIP: None   # Headless — no load-balanced VIP
  selector:
    app: etcd
  ports:
    - port: 2379
      name: client
    - port: 2380
      name: peer

# DNS records created:
# etcd-0.etcd.default.svc.cluster.local → Pod IP
# etcd-1.etcd.default.svc.cluster.local → Pod IP
# etcd-2.etcd.default.svc.cluster.local → Pod IP
```

---

## 7. DaemonSet

Ensures one pod runs on every (or selected) node.

### Use Cases for Dell/Storage

| Use Case | Example |
|----------|---------|
| **CSI node plugin** | Dell CSI driver node component |
| **Storage agent** | Node-level storage monitoring |
| **Log collector** | Fluent Bit / Filebeat |
| **Monitoring** | node-exporter, cAdvisor |
| **Network plugin** | Calico, Cilium agent |
| **Device plugin** | GPU, FPGA, NVMe device plugins |

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-node-driver
  namespace: dell-storage
spec:
  selector:
    matchLabels:
      app: csi-node
  template:
    metadata:
      labels:
        app: csi-node
    spec:
      # Only on nodes with storage access
      nodeSelector:
        storage-access: "true"
      
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      
      containers:
        - name: csi-driver
          image: dell/csi-powerstore:v2.8
          securityContext:
            privileged: true  # Required for mount operations
          volumeMounts:
            - name: device-dir
              mountPath: /dev
            - name: kubelet-dir
              mountPath: /var/lib/kubelet
              mountPropagation: Bidirectional
      
      volumes:
        - name: device-dir
          hostPath:
            path: /dev
        - name: kubelet-dir
          hostPath:
            path: /var/lib/kubelet
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # or percentage "10%"
      maxSurge: 0           # DaemonSets: 0 (kill before create) or 1
```

---

## 8. Job & CronJob

### Job — Run to Completion

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: storage-migration
spec:
  completions: 10        # Total work items to complete
  parallelism: 3         # Max concurrent pods
  backoffLimit: 4        # Max retries before failure
  activeDeadlineSeconds: 3600  # Max total runtime (1 hour)
  ttlSecondsAfterFinished: 86400  # Cleanup after 24h
  
  completionMode: Indexed  # Each pod gets JOB_COMPLETION_INDEX (0-9)
  
  template:
    spec:
      containers:
        - name: migrate
          image: dell/data-migrator:v1
          env:
            - name: SHARD_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: OnFailure  # or Never
```

### Job Completion Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `NonIndexed` | Each pod is interchangeable | Queue-based work |
| `Indexed` | Each pod gets unique index (0 to completions-1) | Sharded processing |

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: storage-backup
spec:
  schedule: "0 2 * * *"    # 2 AM daily
  timeZone: "America/Chicago"  # K8s 1.27+
  
  concurrencyPolicy: Forbid   # Don't start if previous still running
  # Allow — concurrent runs allowed
  # Forbid — skip if previous running
  # Replace — kill previous, start new
  
  startingDeadlineSeconds: 300  # If missed by 5 min, skip
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: dell/backup-agent:v1
              command: ['./backup', '--full']
          restartPolicy: OnFailure
```

---

## 9. Pod Lifecycle

### Pod Phases

```
             ┌──────────┐
         ┌──→│ Pending  │  (scheduling, image pull)
         │   └────┬─────┘
         │        │
         │   ┌────▼─────┐
Created ─┘   │ Running  │  (at least 1 container running)
             └────┬─────┘
                  │
         ┌───────┴───────┐
         │               │
    ┌────▼─────┐    ┌────▼─────┐
    │Succeeded │    │  Failed  │
    │(all exit │    │(at least │
    │ code 0)  │    │ 1 non-0) │
    └──────────┘    └──────────┘
    
   (Jobs/Pods with    (restartPolicy
    restartPolicy:     determines retry)
    Never)
```

### Pod Conditions

| Condition | Description |
|-----------|-------------|
| `PodScheduled` | Pod has been assigned to a node |
| `Initialized` | All init containers completed successfully |
| `ContainersReady` | All containers are ready |
| `Ready` | Pod is ready to serve requests (added to Service endpoints) |

### Container States

```
               ┌───────────┐
  Pull image → │  Waiting  │  (image pull, pending volume mount)
               └─────┬─────┘
                     │
               ┌─────▼─────┐
  Start proc → │  Running  │  (process PID 1 executing)
               └─────┬─────┘
                     │
               ┌─────▼──────┐
  Exit/Signal →│ Terminated │  (exitCode, signal, reason)
               └────────────┘
```

---

## 10. Container Probes

### Probe Types

```
┌────────────────────────────────────────────────────┐
│                    POD STARTUP                      │
│                                                     │
│  ┌─────────────┐                                  │
│  │StartupProbe │  Runs first, disables other probes│
│  │ (gate)      │  until success                    │
│  └──────┬──────┘                                  │
│         │ success                                  │
│         ▼                                          │
│  ┌─────────────┐  ┌──────────────┐                │
│  │LivenessProbe│  │ReadinessProbe│                │
│  │ (is alive?) │  │ (can serve?) │                │
│  └──────┬──────┘  └──────┬───────┘                │
│         │                │                          │
│    fail → KILL      fail → remove from              │
│    & restart        Service endpoints               │
└────────────────────────────────────────────────────┘
```

### Probe Mechanisms

```yaml
# HTTP GET probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: X-Custom-Header
        value: health-check
  initialDelaySeconds: 15
  periodSeconds: 20
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3

# TCP Socket probe
readinessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 10

# Exec probe
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - pg_isready -U postgres
  periodSeconds: 30

# gRPC probe (K8s 1.27+ GA)
readinessProbe:
  grpc:
    port: 50051
    service: "health.v1.Health"  # optional
  periodSeconds: 10
```

### Startup Probe Strategy for Slow-Starting Apps

```yaml
# App takes up to 5 minutes to start
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30      # 30 attempts
  periodSeconds: 10          # Every 10 seconds = 5 min max
# After startup probe succeeds, liveness/readiness probes take over
```

---

## 11. Graceful Shutdown

```
┌──────────────────────────────────────────────────┐
│              Pod Termination Flow                  │
│                                                   │
│  1. pod.deletionTimestamp set                     │
│  2. Pod removed from Service endpoints (parallel) │
│  3. PreStop hook executes (if defined)            │
│  4. SIGTERM sent to PID 1 in all containers      │
│  5. Wait terminationGracePeriodSeconds (default 30)│
│  6. SIGKILL sent (force kill)                     │
└──────────────────────────────────────────────────┘
```

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                # Drain connections
                curl -X POST localhost:8080/drain
                # Wait for in-flight requests
                sleep 15
```

### Common Gotcha: Race Condition on Shutdown

```
Problem: Pod receives traffic AFTER getting SIGTERM because
Endpoints update hasn't propagated to all kube-proxy nodes yet.

Solution: Add a preStop sleep (5-10 seconds) to allow endpoint
removal to propagate before shutting down.

lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]  # Wait for endpoint removal
```

---

## 12. Pod Disruption Budgets (PDB)

Limits voluntary disruptions during node maintenance, upgrades, or autoscaling.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: etcd-pdb
spec:
  # Use ONE of minAvailable or maxUnavailable
  minAvailable: 2          # At least 2 pods must remain
  # maxUnavailable: 1      # At most 1 pod can be down
  
  selector:
    matchLabels:
      app: etcd

  # unhealthyPodEvictionPolicy: AlwaysAllow (K8s 1.31+)
  # Allows evicting unhealthy pods even if PDB is violated
```

### PDB Behavior

```
etcd StatefulSet: 3 replicas, PDB minAvailable=2

Drain node-1 (has etcd-0):
  → Check PDB: 3 running, min=2 → can evict 1 → ALLOWED
  → etcd-0 evicted, rescheduled to node-3

Drain node-2 (has etcd-1):
  → Check PDB: 2 running (etcd-1, etcd-2), min=2
  → Would leave 1 running → BLOCKED until etcd-0 is Ready
```

---

## 13. Restart Policies

| Policy | Behavior | Used By |
|--------|----------|---------|
| `Always` | Always restart (default for Pods) | Deployments, StatefulSets, DaemonSets |
| `OnFailure` | Restart only on non-zero exit code | Jobs (retry on failure) |
| `Never` | Never restart | Jobs (one attempt only) |

### Backoff Policy

When a container repeatedly crashes:
```
1st restart: 10s delay
2nd restart: 20s delay
3rd restart: 40s delay
... exponential backoff ...
Max: 5 minutes (300s cap)

Container enters CrashLoopBackOff state
```

---

## Interview Questions

1. **Deployment vs StatefulSet — when to use each?**
   - Deployment: stateless apps, interchangeable pods, rolling updates
   - StatefulSet: databases, message queues — need stable identity, ordered ops, persistent volumes

2. **How does a rolling update work? What if a new version fails probes?**
   - Deployment creates new ReplicaSet, gradually scales up new while scaling down old. If new pods fail readiness probes, rollout stalls (doesn't exceed maxUnavailable). You can rollback with `kubectl rollout undo`.

3. **What's the difference between liveness and readiness probes?**
   - Liveness: is the container alive? Failure → kubelet kills and restarts container
   - Readiness: can it serve traffic? Failure → removed from Service endpoints (not killed)

4. **How does Kubernetes handle graceful shutdown?**
   - PreStop hook → SIGTERM → grace period → SIGKILL. Application must handle SIGTERM to drain connections. Use preStop sleep to handle endpoint propagation delay.

5. **What is CrashLoopBackOff?**
   - Container keeps crashing and restarting. Backoff delay increases exponentially (10s, 20s, 40s... up to 5m). Debug with `kubectl logs --previous` and `kubectl describe pod`.

6. **Explain Pod Disruption Budgets. Why are they important for storage workloads?**
   - PDBs prevent voluntary disruptions from violating availability. For storage (e.g., etcd, distributed DB), losing too many replicas simultaneously risks data loss. PDB ensures quorum is maintained during maintenance.

---

*Next: [Chapter 3 — Configuration & Secrets](ConfigurationAndSecrets.md)*
