# Chapter 8 — Operators & Custom Resources

> **Critical for Dell interview** — Dell builds Kubernetes operators for CSI drivers, storage replication, and platform lifecycle management. Understanding the operator pattern deeply is essential.

---

## 1. Custom Resource Definitions (CRDs)

Extend the Kubernetes API with your own resource types.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: storagepools.storage.dell.com
spec:
  group: storage.dell.com
  names:
    kind: StoragePool
    listKind: StoragePoolList
    singular: storagepool
    plural: storagepools
    shortNames: ["sp"]
    categories: ["dell", "storage"]
  
  scope: Namespaced    # or Cluster
  
  versions:
    - name: v1
      served: true
      storage: true     # Only one version can be storage version
      
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["arrayId", "poolName", "protocol"]
              properties:
                arrayId:
                  type: string
                  description: "Dell storage array ID"
                poolName:
                  type: string
                protocol:
                  type: string
                  enum: ["iSCSI", "FC", "NFS", "NVMe"]
                capacityGB:
                  type: integer
                  minimum: 1
                replicationEnabled:
                  type: boolean
                  default: false
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Provisioning", "Ready", "Error", "Deleting"]
                availableCapacityGB:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                        enum: ["True", "False", "Unknown"]
                      lastTransitionTime:
                        type: string
                        format: date-time
                      reason:
                        type: string
                      message:
                        type: string
      
      # Status subresource: separate RBAC for .status updates
      subresources:
        status: {}
        # scale: {}  # Optional: enable /scale subresource
      
      # Additional columns for kubectl get
      additionalPrinterColumns:
        - name: Array
          type: string
          jsonPath: .spec.arrayId
        - name: Protocol
          type: string
          jsonPath: .spec.protocol
        - name: Capacity
          type: integer
          jsonPath: .spec.capacityGB
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
    
    - name: v1alpha1
      served: true
      storage: false    # Old version, still served but not stored
```

### Custom Resource Instance

```yaml
apiVersion: storage.dell.com/v1
kind: StoragePool
metadata:
  name: powerstore-pool-1
  namespace: dell-storage
spec:
  arrayId: "PS000000001"
  poolName: "premium-ssd"
  protocol: "iSCSI"
  capacityGB: 10000
  replicationEnabled: true
```

```bash
kubectl get storagepools -n dell-storage
# NAME                 ARRAY          PROTOCOL  CAPACITY  STATUS  AGE
# powerstore-pool-1    PS000000001    iSCSI     10000     Ready   5d
```

---

## 2. Operator Pattern

```
┌──────────────────────────────────────────────────────────┐
│                    OPERATOR PATTERN                       │
│                                                          │
│  CRD (Custom Resource Definition)                        │
│    + Custom Controller                                   │
│    = Operator                                            │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │                Controller                        │    │
│  │                                                  │    │
│  │  Watch ──→ Diff ──→ Act ──→ Update Status       │    │
│  │                                                  │    │
│  │  "Observe the current state of the world,       │    │
│  │   determine differences from desired state,      │    │
│  │   and take actions to converge."                 │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  Encodes operational knowledge: install, upgrade,        │
│  backup, failover, scaling, monitoring, recovery         │
└──────────────────────────────────────────────────────────┘
```

### Maturity Model

| Level | Capability | Example |
|-------|-----------|---------|
| 1 | **Basic Install** | Helm chart, basic deployment |
| 2 | **Seamless Upgrades** | Version migrations, config changes |
| 3 | **Full Lifecycle** | Backup, restore, failure recovery |
| 4 | **Deep Insights** | Metrics, alerts, log aggregation |
| 5 | **Auto Pilot** | Auto-scaling, auto-tuning, anomaly detection |

---

## 3. Controller Implementation (Go)

### Project Structure (Kubebuilder)

```
├── cmd/
│   └── main.go              # Entry point
├── api/
│   └── v1/
│       ├── storagepool_types.go   # CRD Go types
│       └── zz_generated_deepcopy.go
├── internal/
│   └── controller/
│       ├── storagepool_controller.go  # Reconciler
│       └── storagepool_controller_test.go
├── config/
│   ├── crd/                  # CRD YAML (auto-generated)
│   ├── rbac/                 # RBAC rules
│   └── manager/              # Deployment YAML
├── Dockerfile
├── Makefile
└── go.mod
```

### CRD Types (api/v1/storagepool_types.go)

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// StoragePoolSpec defines the desired state
type StoragePoolSpec struct {
    // +kubebuilder:validation:Required
    ArrayID string `json:"arrayId"`
    
    // +kubebuilder:validation:Required
    PoolName string `json:"poolName"`
    
    // +kubebuilder:validation:Enum=iSCSI;FC;NFS;NVMe
    Protocol string `json:"protocol"`
    
    // +kubebuilder:validation:Minimum=1
    CapacityGB int64 `json:"capacityGB"`
    
    // +kubebuilder:default=false
    ReplicationEnabled bool `json:"replicationEnabled,omitempty"`
}

// StoragePoolStatus defines the observed state
type StoragePoolStatus struct {
    // +kubebuilder:validation:Enum=Provisioning;Ready;Error;Deleting
    Phase string `json:"phase,omitempty"`
    
    AvailableCapacityGB int64 `json:"availableCapacityGB,omitempty"`
    
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Array",type=string,JSONPath=`.spec.arrayId`
// +kubebuilder:printcolumn:name="Protocol",type=string,JSONPath=`.spec.protocol`
// +kubebuilder:printcolumn:name="Status",type=string,JSONPath=`.status.phase`
type StoragePool struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   StoragePoolSpec   `json:"spec,omitempty"`
    Status StoragePoolStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type StoragePoolList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []StoragePool `json:"items"`
}
```

### Reconciler (internal/controller/storagepool_controller.go)

```go
package controller

import (
    "context"
    "fmt"
    "time"
    
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    storagev1 "github.com/dell/storage-operator/api/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
)

const finalizerName = "storage.dell.com/cleanup"

type StoragePoolReconciler struct {
    client.Client
    Scheme       *runtime.Scheme
    StorageClient StorageBackendClient  // Your storage API client
}

// +kubebuilder:rbac:groups=storage.dell.com,resources=storagepools,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=storage.dell.com,resources=storagepools/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=storage.dell.com,resources=storagepools/finalizers,verbs=update

func (r *StoragePoolReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    // 1. FETCH the resource
    pool := &storagev1.StoragePool{}
    if err := r.Get(ctx, req.NamespacedName, pool); err != nil {
        if apierrors.IsNotFound(err) {
            logger.Info("StoragePool deleted, nothing to do")
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    // 2. Handle DELETION (finalizer pattern)
    if !pool.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, pool)
    }
    
    // 3. Add FINALIZER if not present
    if !controllerutil.ContainsFinalizer(pool, finalizerName) {
        controllerutil.AddFinalizer(pool, finalizerName)
        if err := r.Update(ctx, pool); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 4. RECONCILE — ensure desired state matches actual
    
    // Check if pool exists on storage backend
    exists, backendPool, err := r.StorageClient.GetPool(ctx, pool.Spec.ArrayID, pool.Spec.PoolName)
    if err != nil {
        logger.Error(err, "Failed to connect to storage backend")
        r.setCondition(pool, "BackendConnected", metav1.ConditionFalse, "ConnectionFailed", err.Error())
        pool.Status.Phase = "Error"
        _ = r.Status().Update(ctx, pool)
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }
    
    if !exists {
        // Create pool on storage backend
        logger.Info("Creating storage pool on backend", "array", pool.Spec.ArrayID)
        pool.Status.Phase = "Provisioning"
        _ = r.Status().Update(ctx, pool)
        
        if err := r.StorageClient.CreatePool(ctx, pool.Spec); err != nil {
            pool.Status.Phase = "Error"
            r.setCondition(pool, "Provisioned", metav1.ConditionFalse, "CreateFailed", err.Error())
            _ = r.Status().Update(ctx, pool)
            return ctrl.Result{RequeueAfter: 1 * time.Minute}, nil
        }
    }
    
    // Update status from backend
    pool.Status.Phase = "Ready"
    pool.Status.AvailableCapacityGB = backendPool.AvailableGB
    r.setCondition(pool, "BackendConnected", metav1.ConditionTrue, "Connected", "Storage backend reachable")
    r.setCondition(pool, "Provisioned", metav1.ConditionTrue, "Ready", "Pool is ready")
    
    if err := r.Status().Update(ctx, pool); err != nil {
        return ctrl.Result{}, err
    }
    
    // Re-check periodically (level-triggered)
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *StoragePoolReconciler) handleDeletion(ctx context.Context, pool *storagev1.StoragePool) (ctrl.Result, error) {
    if controllerutil.ContainsFinalizer(pool, finalizerName) {
        // Cleanup: delete pool from storage backend
        pool.Status.Phase = "Deleting"
        _ = r.Status().Update(ctx, pool)
        
        if err := r.StorageClient.DeletePool(ctx, pool.Spec.ArrayID, pool.Spec.PoolName); err != nil {
            return ctrl.Result{}, fmt.Errorf("failed to delete backend pool: %w", err)
        }
        
        // Remove finalizer to allow deletion
        controllerutil.RemoveFinalizer(pool, finalizerName)
        if err := r.Update(ctx, pool); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}

func (r *StoragePoolReconciler) setCondition(pool *storagev1.StoragePool, condType string, status metav1.ConditionStatus, reason, message string) {
    meta.SetStatusCondition(&pool.Status.Conditions, metav1.Condition{
        Type:               condType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        LastTransitionTime: metav1.Now(),
    })
}

// SetupWithManager registers the controller with the manager
func (r *StoragePoolReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&storagev1.StoragePool{}).            // Watch StoragePools
        Owns(&corev1.PersistentVolume{}).          // Watch owned PVs
        WithEventFilter(predicate.GenerationChangedPredicate{}). // Skip status-only updates
        Complete(r)
}
```

---

## 4. Owner References & Garbage Collection

```yaml
# Child resource owned by parent
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-from-pool
  ownerReferences:
    - apiVersion: storage.dell.com/v1
      kind: StoragePool
      name: powerstore-pool-1
      uid: "abc-123-def"
      controller: true      # This owner manages the child
      blockOwnerDeletion: true  # Block parent deletion until child is cleaned up
```

### Garbage Collection Policies

| Policy | Behavior |
|--------|----------|
| **Background** (default) | Delete parent immediately, GC deletes children asynchronously |
| **Foreground** | Parent set to "deleting", children deleted first, then parent |
| **Orphan** | Delete parent only, children become orphans |

```go
// Setting owner reference in controller code
func (r *Reconciler) createPV(ctx context.Context, pool *storagev1.StoragePool) error {
    pv := &corev1.PersistentVolume{
        ObjectMeta: metav1.ObjectMeta{
            Name: fmt.Sprintf("pv-%s", pool.Name),
        },
        Spec: corev1.PersistentVolumeSpec{...},
    }
    
    // Set owner reference — PV will be GC'd when StoragePool is deleted
    if err := controllerutil.SetControllerReference(pool, pv, r.Scheme); err != nil {
        return err
    }
    
    return r.Create(ctx, pv)
}
```

---

## 5. Finalizers

Pre-deletion hooks for resource cleanup.

```
DELETE request received
        │
        ▼
Has finalizers? ──No──→ Object deleted from etcd
        │
       Yes
        │
        ▼
Set deletionTimestamp (object NOT deleted yet)
        │
        ▼
Controller sees deletionTimestamp != nil
        │
        ▼
Performs cleanup (delete external resources)
        │
        ▼
Remove finalizer from object
        │
        ▼
No more finalizers → Object deleted from etcd
```

### Finalizer Rules

- Object with finalizer is **never deleted** until finalizer is removed
- Controller MUST handle deletion logic and remove finalizer
- Stuck finalizer = stuck deletion → `kubectl edit` to remove manually (dangerous)
- Use finalizers when external resources need cleanup (storage volumes, DNS records, cloud resources)

---

## 6. Admission Webhooks

### Mutating Admission Webhook

```go
// Inject default storage tier if not specified
func (h *StoragePoolDefaulter) Default(ctx context.Context, obj runtime.Object) error {
    pool := obj.(*storagev1.StoragePool)
    
    if pool.Spec.Protocol == "" {
        pool.Spec.Protocol = "iSCSI"  // Default protocol
    }
    if pool.Spec.CapacityGB == 0 {
        pool.Spec.CapacityGB = 100    // Default 100GB
    }
    
    return nil
}
```

### Validating Admission Webhook

```go
// Validate storage pool configuration
func (h *StoragePoolValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (warnings admission.Warnings, err error) {
    pool := obj.(*storagev1.StoragePool)
    
    // Validate array ID format
    if !isValidArrayID(pool.Spec.ArrayID) {
        return nil, fmt.Errorf("invalid arrayId format: %s", pool.Spec.ArrayID)
    }
    
    // Validate NFS requires NAS server
    if pool.Spec.Protocol == "NFS" && pool.Spec.NASServer == "" {
        return nil, fmt.Errorf("NFS protocol requires nasServer to be set")
    }
    
    // Warning (non-blocking)
    var warns admission.Warnings
    if pool.Spec.CapacityGB > 10000 {
        warns = append(warns, "Large pool: consider splitting across arrays")
    }
    
    return warns, nil
}
```

### Webhook Configuration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: storage-pool-validator
webhooks:
  - name: validate.storagepool.storage.dell.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: storage-operator-webhook
        namespace: dell-storage
        path: /validate-storage-dell-com-v1-storagepool
      caBundle: <CA-BUNDLE>
    rules:
      - apiGroups: ["storage.dell.com"]
        apiVersions: ["v1"]
        resources: ["storagepools"]
        operations: ["CREATE", "UPDATE"]
    failurePolicy: Fail    # Reject if webhook unavailable
    # Ignore — allow if webhook unavailable
```

---

## 7. Leader Election

Ensure only one controller instance is active in HA deployments.

```go
func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        LeaderElection:         true,
        LeaderElectionID:       "storage-operator-leader",
        LeaderElectionNamespace: "dell-storage",
        LeaseDuration:          15 * time.Second,
        RenewDeadline:          10 * time.Second,
        RetryPeriod:            2 * time.Second,
    })
    
    // Only leader runs reconciliation loops
    // Other replicas wait, ready to take over if leader fails
}
```

```yaml
# Lease object (auto-created)
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: storage-operator-leader
  namespace: dell-storage
spec:
  holderIdentity: storage-operator-7d4f8-abc12
  leaseDurationSeconds: 15
  acquireTime: "2024-01-15T10:00:00Z"
  renewTime: "2024-01-15T10:30:45Z"
  leaderTransitions: 3
```

---

## 8. Testing Operators

### Unit Tests (EnvTest)

```go
func TestStoragePoolReconciler(t *testing.T) {
    // Setup envtest (lightweight API server + etcd)
    testEnv := &envtest.Environment{
        CRDDirectoryPaths: []string{filepath.Join("..", "..", "config", "crd", "bases")},
    }
    
    cfg, err := testEnv.Start()
    require.NoError(t, err)
    defer testEnv.Stop()
    
    // Create manager and register controller
    mgr, _ := ctrl.NewManager(cfg, ctrl.Options{Scheme: scheme})
    reconciler := &StoragePoolReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
        StorageClient: &mockStorageClient{},
    }
    reconciler.SetupWithManager(mgr)
    
    // Start manager in background
    go mgr.Start(context.Background())
    
    // Test: Create StoragePool → verify status becomes Ready
    pool := &storagev1.StoragePool{
        ObjectMeta: metav1.ObjectMeta{Name: "test-pool", Namespace: "default"},
        Spec: storagev1.StoragePoolSpec{
            ArrayID:    "PS001",
            PoolName:   "test",
            Protocol:   "iSCSI",
            CapacityGB: 100,
        },
    }
    
    err = mgr.GetClient().Create(context.Background(), pool)
    require.NoError(t, err)
    
    // Wait for reconciliation
    eventually(t, func() bool {
        _ = mgr.GetClient().Get(context.Background(), client.ObjectKeyFromObject(pool), pool)
        return pool.Status.Phase == "Ready"
    }, 10*time.Second)
}
```

---

## Interview Questions

1. **What is the Operator pattern?**
   - CRD + custom controller that encodes operational knowledge. Controller watches CRs, compares desired state with actual state, and takes actions to converge. Automates Day 2 operations: upgrades, backup, scaling, failure recovery.

2. **Explain the reconciliation loop.**
   - Reconcile(ctx, Request) → Fetch resource → Check if deleted (handle finalizer) → Compare spec with actual state → Take corrective action → Update status → Return (requeue if needed). Must be **idempotent** — multiple calls produce same result.

3. **What are finalizers?**
   - Strings on metadata.finalizers that prevent object deletion. Controller sees deletionTimestamp, performs cleanup (delete external resources), removes finalizer, then object is actually deleted. Essential for CSI: cleanup storage volumes before PV deletion.

4. **What is the difference between ownerReferences and finalizers?**
   - ownerReferences: automatic garbage collection — parent deleted → children deleted. Finalizers: manual cleanup hook — controller runs cleanup logic before deletion. Use both: ownerReferences for K8s resources, finalizers for external resources.

5. **How would you build a Dell storage operator?**
   - CRDs: StorageArray, StoragePool, ReplicationGroup. Controller: reconcile CRs against Dell PowerStore REST API. Finalizers for volume cleanup. Admission webhooks for validation (valid array ID, protocol). Leader election for HA. Metrics for observability. Test with envtest + mock storage client.

6. **How do admission webhooks work?**
   - API server calls webhook (HTTP POST) during admission. Mutating: modify resource (inject defaults). Validating: approve/reject (enforce policies). Runs after authentication and authorization, before persistence. FailurePolicy controls behavior when webhook is unavailable.

---

*Next: [Chapter 9 — Helm & Application Packaging](HelmAndPackaging.md)*
