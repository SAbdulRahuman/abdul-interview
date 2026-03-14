# Chapter 3 — Configuration & Secrets

## 1. ConfigMaps

Store non-confidential configuration data as key-value pairs.

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# From file
kubectl create configmap nginx-config --from-file=nginx.conf

# From directory
kubectl create configmap configs --from-file=config-dir/

# From env file
kubectl create configmap env-config --from-env-file=app.env
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: storage-config
data:
  # Simple key-value
  LOG_LEVEL: "info"
  STORAGE_BACKEND: "powerstore"
  MAX_RETRIES: "3"
  
  # Multi-line config file
  config.yaml: |
    server:
      port: 8080
      readTimeout: 30s
    storage:
      endpoint: https://powerstore.internal:443
      pool: default-pool
      protocol: iSCSI
```

### Using ConfigMaps

```yaml
spec:
  containers:
    - name: app
      image: myapp:v1
      
      # Method 1: Environment variables
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: storage-config
              key: LOG_LEVEL
      
      # Method 2: All keys as env vars
      envFrom:
        - configMapRef:
            name: storage-config
          prefix: STORAGE_  # Optional prefix: STORAGE_LOG_LEVEL
      
      # Method 3: Volume mount (files)
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  
  volumes:
    - name: config-volume
      configMap:
        name: storage-config
        items:
          - key: config.yaml
            path: config.yaml    # Mounted at /etc/config/config.yaml
```

### ConfigMap Updates

| Method | Automatic Update | Restart Required |
|--------|-----------------|-----------------|
| **Volume mount** | Yes (kubelet sync ~1 min) | App must watch for file changes |
| **Env variable** | No | Yes (pod restart) |
| **envFrom** | No | Yes (pod restart) |

### Immutable ConfigMaps (K8s 1.21+)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: static-config
immutable: true   # Cannot be updated after creation
data:
  VERSION: "2.0"
# Benefits: Protects against accidental changes, reduces API server load
# (no watches needed for immutable ConfigMaps)
```

---

## 2. Secrets

Store sensitive data (passwords, tokens, certificates).

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Default — arbitrary key-value data |
| `kubernetes.io/tls` | TLS certificate + key |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/service-account-token` | Legacy SA token |
| `kubernetes.io/basic-auth` | Username + password |
| `kubernetes.io/ssh-auth` | SSH private key |

### Creating Secrets

```bash
# From literal
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='S3cr3t!'

# TLS secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Docker registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.dell.com \
  --docker-username=user \
  --docker-password=pass
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=              # base64 encoded "admin"
  password: UzNjcjN0IQ==          # base64 encoded "S3cr3t!"
---
# Using stringData (plain text — encoded on creation)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: S3cr3t!
```

### Using Secrets

```yaml
spec:
  containers:
    - name: app
      # As environment variable
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      
      # As volume (files)
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
  
  volumes:
    - name: secrets
      secret:
        secretName: db-credentials
        defaultMode: 0400  # Read-only by owner
```

### Secret Security Considerations

```
⚠️ Base64 ≠ encryption
   - Secrets stored in etcd are base64-encoded by default
   - Enable etcd encryption at rest:
     --encryption-provider-config=encryption-config.yaml

⚠️ RBAC controls
   - Restrict "get" verb on secrets to specific ServiceAccounts
   - Audit who can read secrets

⚠️ External secret management (preferred for production)
   - HashiCorp Vault + Vault Agent Injector
   - AWS Secrets Manager + External Secrets Operator
   - Azure Key Vault + CSI secrets store driver
   - Sealed Secrets (Bitnami) — encrypted in Git
```

### Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}  # Fallback: plaintext (for reading old secrets)
```

---

## 3. Downward API

Expose pod and container metadata to containers without coupling to the Kubernetes API.

```yaml
spec:
  containers:
    - name: app
      env:
        # Pod-level fields
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
        
        # Container-level fields
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: limits.cpu
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: limits.memory

      # As files in a volume
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
          - path: annotations
            fieldRef:
              fieldPath: metadata.annotations
```

### Available Fields

| Category | Fields |
|----------|--------|
| **Pod metadata** | name, namespace, uid, labels, annotations |
| **Pod spec** | nodeName, serviceAccountName |
| **Pod status** | podIP, hostIP |
| **Container resources** | requests.cpu, limits.cpu, requests.memory, limits.memory |

---

## 4. Resource Management

### Requests and Limits

```yaml
containers:
  - name: app
    resources:
      requests:          # Scheduling guarantee
        cpu: 250m        # 0.25 CPU cores (millicores)
        memory: 256Mi    # 256 MiB
        ephemeral-storage: 1Gi
      limits:            # Hard ceiling
        cpu: "1"         # 1 CPU core
        memory: 512Mi    # 512 MiB
        ephemeral-storage: 2Gi
```

### CPU vs Memory Behavior

| Resource | When limit exceeded | Impact |
|----------|-------------------|--------|
| **CPU** | Throttled (CFS bandwidth) | Pod runs slower but stays alive |
| **Memory** | OOMKilled | Container is killed and restarted |
| **Ephemeral storage** | Evicted | Pod is evicted from node |

### CPU Units

```
1 CPU = 1000 millicores (m)
500m = 0.5 CPU = 50% of one core
100m = 0.1 CPU = 10% of one core
"2" = 2000m = 2 full cores
```

### Memory Units

```
Ki = KiB = 1024 bytes      (kibibyte)
Mi = MiB = 1048576 bytes   (mebibyte)
Gi = GiB = 1073741824 bytes (gibibyte)
K = KB = 1000 bytes        (kilobyte — decimal)
M = MB = 1000000 bytes     (megabyte — decimal)
```

---

## 5. QoS Classes

Kubernetes assigns QoS classes automatically based on resource config.

```
┌────────────────────────────────────────────────────┐
│              QoS Class Rules                       │
│                                                     │
│  Guaranteed:  Every container has                   │
│               requests == limits                    │
│               for BOTH cpu and memory              │
│                                                     │
│  Burstable:   At least one container has           │
│               requests < limits or                 │
│               only requests (no limits)            │
│                                                     │
│  BestEffort:  No requests or limits set            │
│               on any container                      │
│                                                     │
│  Eviction priority (under pressure):               │
│  BestEffort → Burstable → Guaranteed (last)        │
└────────────────────────────────────────────────────┘
```

```yaml
# Guaranteed (requests == limits)
resources:
  requests:
    cpu: "1"
    memory: 1Gi
  limits:
    cpu: "1"
    memory: 1Gi

# Burstable (requests < limits)
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi

# BestEffort (none specified)
# resources: {}
```

### QoS Impact on OOM Score

| QoS Class | OOM Score Adjustment | Kill Priority |
|-----------|---------------------|---------------|
| BestEffort | 1000 | First to die |
| Burstable | 2-999 (based on usage) | Middle |
| Guaranteed | -997 | Last to die |

---

## 6. LimitRanges

Set default and min/max resource constraints per namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: storage-team
spec:
  limits:
    # Per container defaults
    - type: Container
      default:        # Default limits if not specified
        cpu: 500m
        memory: 256Mi
      defaultRequest: # Default requests if not specified
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
    
    # Per pod limits
    - type: Pod
      max:
        cpu: "8"
        memory: 8Gi
    
    # PVC limits
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
```

---

## 7. ResourceQuotas

Limit aggregate resource consumption per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: storage-team
spec:
  hard:
    # Compute resources
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    
    # Object counts
    pods: "50"
    services: "10"
    secrets: "20"
    configmaps: "20"
    persistentvolumeclaims: "30"
    
    # Storage
    requests.storage: 500Gi
    
    # Per storage class
    fast-ssd.storageclass.storage.k8s.io/requests.storage: 200Gi
    fast-ssd.storageclass.storage.k8s.io/persistentvolumeclaims: "10"
    
    # Scoped to BestEffort QoS
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - high-priority
```

```bash
# Check quota usage
kubectl describe resourcequota team-quota -n storage-team
# Output:
# Resource                  Used   Hard
# --------                  ----   ----
# requests.cpu              12     20
# requests.memory           24Gi   40Gi
# pods                      15     50
```

---

## 8. PriorityClasses

Control pod scheduling priority and preemption behavior.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: storage-critical
value: 1000000           # Higher = more important
globalDefault: false
preemptionPolicy: PreemptLowerPriority  # or Never
description: "Critical storage infrastructure pods"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 100
globalDefault: false
preemptionPolicy: Never   # Don't preempt other pods
description: "Low priority batch jobs"
```

```yaml
# Usage in pod spec
spec:
  priorityClassName: storage-critical
  containers:
    - name: csi-controller
      image: dell/csi-powerstore:v2
```

### System Priority Classes

| Class | Value | Purpose |
|-------|-------|---------|
| `system-cluster-critical` | 2000000000 | Essential cluster components |
| `system-node-critical` | 2000001000 | Essential node components |

---

## 9. RuntimeClasses

Select different container runtimes per workload.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc    # CRI handler name
scheduling:
  nodeSelector:
    runtime: gvisor
overhead:
  podFixed:
    cpu: 100m
    memory: 64Mi
---
# Use in pod
spec:
  runtimeClassName: gvisor  # Use gVisor sandbox runtime
  containers:
    - name: untrusted-workload
      image: user-code:latest
```

| Runtime | Isolation Level | Performance | Use Case |
|---------|----------------|-------------|----------|
| **runc** | Container (default) | Native | Standard workloads |
| **gVisor** | User-space kernel | ~15% overhead | Untrusted code |
| **Kata** | Lightweight VM | VM overhead | Strong isolation |

---

## Interview Questions

1. **ConfigMap vs Secret — when to use each?**
   - ConfigMap: non-sensitive configuration (log levels, endpoints, feature flags)
   - Secret: sensitive data (passwords, tokens, keys). Secrets have RBAC controls, can be encrypted at rest, and are base64-encoded.

2. **How do ConfigMap updates propagate?**
   - Volume mounts: kubelet syncs within ~1 minute (configurable). App must watch for file changes.
   - Env vars: NOT updated. Pod must be restarted.
   - Immutable ConfigMaps: cannot be updated at all.

3. **Explain QoS classes and their eviction behavior.**
   - Guaranteed (requests=limits): last evicted, used for critical workloads.
   - Burstable (requests<limits): evicted based on usage relative to request.
   - BestEffort (no resources): first evicted. Never use for production.

4. **What happens when a pod exceeds its CPU limit vs memory limit?**
   - CPU: throttled via CFS bandwidth control. Pod is slower but alive.
   - Memory: OOMKilled by the kernel. Container restarted per restartPolicy.

5. **How would you enforce resource policies across teams in a multi-tenant cluster?**
   - ResourceQuotas per namespace (aggregate limits)
   - LimitRanges per namespace (per-container defaults and bounds)
   - PriorityClasses for scheduling priority
   - Admission webhooks for custom policy enforcement (OPA Gatekeeper)

---

*Next: [Chapter 4 — Storage in Kubernetes](StorageInKubernetes.md)*
