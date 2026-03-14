# Chapter 6 — Security

## 1. Authentication

How users and services prove their identity to the API server.

### Authentication Methods

| Method | Type | Use Case |
|--------|------|----------|
| **X.509 Certificates** | User | Admin access, kubeconfig |
| **Bearer Tokens** | User/Service | CI/CD, automation |
| **OIDC Tokens** | User | SSO (Keycloak, Azure AD, Okta) |
| **Webhook Token** | User | Custom auth integration |
| **ServiceAccount Tokens** | In-cluster | Pod identity |
| **Bootstrap Tokens** | Node | kubeadm node join |

### X.509 Certificate Auth

```bash
# Generate user certificate
openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -out dev-user.csr \
  -subj "/CN=dev-user/O=storage-team"
# CN = username, O = group

# Sign with cluster CA
openssl x509 -req -in dev-user.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out dev-user.crt -days 365

# Add to kubeconfig
kubectl config set-credentials dev-user \
  --client-certificate=dev-user.crt \
  --client-key=dev-user.key
```

### ServiceAccount Tokens

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: storage-controller
  namespace: dell-storage
automountServiceAccountToken: true
---
# Bound token (projected, auto-rotated, audience-scoped)
# Automatically mounted at /var/run/secrets/kubernetes.io/serviceaccount/token
# K8s 1.24+: No auto-created long-lived secret tokens

# Create long-lived token (if needed)
apiVersion: v1
kind: Secret
metadata:
  name: storage-controller-token
  annotations:
    kubernetes.io/service-account.name: storage-controller
type: kubernetes.io/service-account-token
```

---

## 2. Authorization

### RBAC — Role-Based Access Control

```
┌─────────────────────────────────────────────────────┐
│                  RBAC Model                          │
│                                                     │
│  WHO                WHAT              WHERE          │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐  │
│  │ User     │      │ Role     │      │Namespace │  │
│  │ Group    │──────│ (verbs + │──────│ (scoped) │  │
│  │ SvcAcct  │ Bind │ resources)│      │ or       │  │
│  └──────────┘ ───→ └──────────┘      │ Cluster  │  │
│                                      └──────────┘  │
│                                                     │
│  Role + RoleBinding              = namespace-scoped │
│  ClusterRole + ClusterRoleBinding = cluster-scoped  │
│  ClusterRole + RoleBinding        = reuse across ns │
└─────────────────────────────────────────────────────┘
```

### Role & RoleBinding

```yaml
# Role: defines permissions within a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]              # Core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
  
  - apiGroups: [""]
    resources: ["pods"]
    resourceNames: ["my-pod"]    # Specific resource
    verbs: ["get"]
---
# RoleBinding: grants Role to a subject
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: storage-team
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: monitoring-agent
    namespace: monitoring
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole & ClusterRoleBinding

```yaml
# ClusterRole: cluster-wide (or reusable across namespaces)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-admin
rules:
  # Storage resources
  - apiGroups: [""]
    resources: ["persistentvolumes", "persistentvolumeclaims"]
    verbs: ["*"]
  
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "volumeattachments"]
    verbs: ["*"]
  
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots", "volumesnapshotclasses", "volumesnapshotcontents"]
    verbs: ["*"]
  
  # Node access for CSI
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  
  # Events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: storage-admin-binding
subjects:
  - kind: ServiceAccount
    name: csi-controller
    namespace: dell-storage
roleRef:
  kind: ClusterRole
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Verbs

| Verb | HTTP Method | Description |
|------|------------|-------------|
| `get` | GET (single) | Read one resource |
| `list` | GET (collection) | List resources |
| `watch` | GET (stream) | Watch for changes |
| `create` | POST | Create resource |
| `update` | PUT | Full update |
| `patch` | PATCH | Partial update |
| `delete` | DELETE | Delete resource |
| `deletecollection` | DELETE | Delete collection |
| `impersonate` | — | Act as another user |
| `bind` | — | Bind roles |
| `escalate` | — | Create roles with more perms |

### Aggregated ClusterRoles

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-viewer
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
# Auto-aggregated into built-in "view" ClusterRole
```

### Check Permissions

```bash
# Can I do X?
kubectl auth can-i create pods -n production
kubectl auth can-i '*' '*'   # Am I cluster admin?

# Can ServiceAccount do X?
kubectl auth can-i get pods \
  --as=system:serviceaccount:dell-storage:csi-controller

# List all roles/bindings
kubectl get roles,rolebindings -n production
kubectl get clusterroles,clusterrolebindings
```

---

## 3. Pod Security

### Pod Security Standards (PSS)

| Level | Description | Examples |
|-------|-------------|---------|
| **Privileged** | No restrictions | CSI node drivers, system daemons |
| **Baseline** | Minimal restrictions | Most workloads |
| **Restricted** | Hardened | Untrusted/multi-tenant workloads |

### Restricted Policy Requirements

```
- Must run as non-root (runAsNonRoot: true)
- No privilege escalation (allowPrivilegeEscalation: false)
- No hostPath, hostNetwork, hostPID, hostIPC
- No privileged containers
- Drop ALL capabilities
- Restricted volume types (configMap, emptyDir, PVC, projected, secret)
- Seccomp profile required (RuntimeDefault or Localhost)
- Read-only root filesystem (recommended)
```

### Pod Security Admission (PSA)

```yaml
# Enforce pod security standards per namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce = reject violating pods
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Audit = log violations but allow
    pod-security.kubernetes.io/audit: restricted
    
    # Warn = show warning but allow
    pod-security.kubernetes.io/warn: restricted
```

### SecurityContext (Pod and Container Level)

```yaml
spec:
  # Pod-level security
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
    supplementalGroups: [4000]
  
  containers:
    - name: app
      # Container-level security (overrides pod-level)
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]  # Only if needed
        seccompProfile:
          type: RuntimeDefault
      
      # Writable directories via emptyDir
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache
  
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
```

---

## 4. Capabilities

Linux capabilities grant granular privileges instead of full root.

| Capability | Purpose | When Needed |
|-----------|---------|-------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 | Web servers on port 80/443 |
| `SYS_PTRACE` | Trace processes | Debugging (ephemeral containers) |
| `NET_ADMIN` | Network configuration | CNI plugins, iptables |
| `SYS_ADMIN` | Broad admin ops | Avoid — too powerful |
| `DAC_OVERRIDE` | Bypass file permissions | File operations as non-owner |
| `CHOWN` | Change file ownership | fsGroup operations |

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]          # Start from zero
    add:                   # Add only what's needed
      - NET_BIND_SERVICE
```

---

## 5. Network Policies for Security

```yaml
# Zero-trust: deny all, then allow specific flows
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow only required communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-storage-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: storage-api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:        # Allow DNS
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

---

## 6. Secrets Management

### External Secrets Operator

```yaml
# SecretStore: connection to external secrets backend
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault
  namespace: dell-storage
spec:
  provider:
    vault:
      server: https://vault.internal:8200
      path: secret
      auth:
        kubernetes:
          mountPath: kubernetes
          role: storage-reader
          serviceAccountRef:
            name: vault-auth
---
# ExternalSecret: sync secret from external source
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: storage-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: powerstore-creds        # K8s Secret name created
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: dell/powerstore
        property: username
    - secretKey: password
      remoteRef:
        key: dell/powerstore
        property: password
```

### Sealed Secrets (Git-safe)

```bash
# Encrypt secret — safe to commit to Git
kubeseal --format=yaml < secret.yaml > sealed-secret.yaml

# Only the cluster's Sealed Secrets controller can decrypt
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-creds
spec:
  encryptedData:
    password: AgBy8hPs...  # Encrypted with cluster public key
```

---

## 7. Image Security

```yaml
# Admission webhook to enforce image policies
# (using OPA Gatekeeper or Kyverno)

# Kyverno policy: require signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["registry.dell.com/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYH...
                      -----END PUBLIC KEY-----
```

---

## 8. Audit Logging

```yaml
# Audit policy: what to log
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all secret access at Request level
  - level: Request
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log pod creation/deletion
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "delete"]
  
  # Skip health checks
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
  
  # Log everything else at Metadata level
  - level: Metadata
```

### Audit Levels

| Level | Records |
|-------|---------|
| `None` | Nothing |
| `Metadata` | Request metadata (user, verb, resource, timestamp) |
| `Request` | Metadata + request body |
| `RequestResponse` | Metadata + request + response body |

---

## 9. mTLS with cert-manager

```yaml
# Install cert-manager, then create certificates
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: ca-key-pair
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: storage-api-tls
  namespace: production
spec:
  secretName: storage-api-tls
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
  commonName: storage-api.production.svc.cluster.local
  dnsNames:
    - storage-api
    - storage-api.production
    - storage-api.production.svc
    - storage-api.production.svc.cluster.local
  duration: 8760h    # 1 year
  renewBefore: 720h  # Renew 30 days before expiry
```

---

## Interview Questions

1. **Explain RBAC in Kubernetes.**
   - Role defines permissions (verbs on resources). RoleBinding grants Role to users/groups/ServiceAccounts. Namespace-scoped (Role/RoleBinding) or cluster-scoped (ClusterRole/ClusterRoleBinding). Principle of least privilege.

2. **How would you set up RBAC for a CSI driver?**
   - ClusterRole with access to PV, PVC, StorageClass, VolumeAttachment, Node, Events. ClusterRoleBinding to CSI controller ServiceAccount. Node plugin needs minimal RBAC (just node registration).

3. **What are Pod Security Standards?**
   - Three levels: Privileged (no restrictions), Baseline (prevent known escalations), Restricted (hardened). Enforced via Pod Security Admission labels on namespaces (enforce/audit/warn modes).

4. **How do you manage secrets securely in Kubernetes?**
   - Enable etcd encryption at rest. Use external secret managers (Vault, AWS SM) with External Secrets Operator. Limit RBAC access to secrets. Use Sealed Secrets for GitOps. Never commit secrets to Git.

5. **What capabilities does a CSI node driver need?**
   - Typically runs privileged (securityContext.privileged: true) because it needs: mount/unmount filesystem, device scanning, iSCSI login, multipath configuration. This is why CSI node plugins are exempted from Restricted PSS.

---

*Next: [Chapter 7 — Scheduling & Resource Management](SchedulingAndResources.md)*
