# Chapter 5 — Networking

## 1. Kubernetes Networking Model

### Fundamental Rules

1. Every pod gets its own unique IP address
2. Pods on any node can communicate with all pods on any node **without NAT**
3. Agents on a node can communicate with all pods on that node
4. Pod IP is routable within the cluster (flat network)

```
┌────────────────────────────────────────────────────────┐
│                    CLUSTER NETWORK                      │
│                                                        │
│  Node 1 (10.0.1.10)          Node 2 (10.0.1.11)       │
│  ┌──────────────────┐        ┌──────────────────┐     │
│  │ Pod A            │        │ Pod C            │     │
│  │ 10.244.1.2       │◄──────►│ 10.244.2.3       │     │
│  │                  │ Direct │                  │     │
│  │ Pod B            │ (no    │ Pod D            │     │
│  │ 10.244.1.3       │  NAT)  │ 10.244.2.4       │     │
│  └──────────────────┘        └──────────────────┘     │
│                                                        │
│  CNI Plugin handles: pod CIDR allocation, routing      │
│  between nodes (VXLAN, BGP, eBPF, etc.)               │
└────────────────────────────────────────────────────────┘
```

### Communication Layers

| Layer | How | Example |
|-------|-----|---------|
| Container-to-Container | `localhost` (shared network namespace) | App → sidecar on 127.0.0.1:15001 |
| Pod-to-Pod (same node) | Virtual bridge (cbr0/cni0) | 10.244.1.2 → 10.244.1.3 |
| Pod-to-Pod (cross-node) | CNI overlay/BGP routing | 10.244.1.2 → 10.244.2.3 |
| Pod-to-Service | kube-proxy (iptables/IPVS) DNAT | pod → 10.96.0.10 → pod endpoint |
| External-to-Service | NodePort / LoadBalancer / Ingress | client → LB → pod |

---

## 2. Services

Abstract a set of pods behind a stable network endpoint.

### Service Types

```
┌──────────────────────────────────────────────────────┐
│                   Service Types                       │
│                                                      │
│  ClusterIP          NodePort          LoadBalancer    │
│  ┌──────────┐      ┌──────────┐     ┌──────────┐    │
│  │Internal  │      │ClusterIP │     │NodePort  │    │
│  │IP only   │      │+ NodePort│     │+ClusterIP│    │
│  │10.96.x.x │      │30000-    │     │+ Cloud   │    │
│  │          │      │32767     │     │  L4 LB   │    │
│  └──────────┘      └──────────┘     └──────────┘    │
│                                                      │
│  ExternalName       Headless                         │
│  ┌──────────┐      ┌──────────┐                     │
│  │CNAME     │      │clusterIP:│                     │
│  │alias to  │      │None      │                     │
│  │external  │      │DNS A recs│                     │
│  │DNS       │      │per pod   │                     │
│  └──────────┘      └──────────┘                     │
└──────────────────────────────────────────────────────┘
```

### ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: storage-api
spec:
  type: ClusterIP             # Default
  selector:
    app: storage-api
  ports:
    - name: http
      port: 80                # Service port
      targetPort: 8080        # Container port
      protocol: TCP
    - name: grpc
      port: 9090
      targetPort: 9090
  
  # Session affinity (sticky sessions)
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800   # 3 hours
```

### NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: storage-api-nodeport
spec:
  type: NodePort
  selector:
    app: storage-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080         # Fixed port (or auto-assigned 30000-32767)
  
  # externalTrafficPolicy controls source IP preservation
  externalTrafficPolicy: Local
  # Cluster (default): distribute across all pods (SNAT, lose source IP)
  # Local: only send to pods on receiving node (preserves source IP)
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: storage-api-lb
  annotations:
    # Cloud-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: storage-api
  ports:
    - port: 443
      targetPort: 8443
  loadBalancerSourceRanges:   # Restrict access by source CIDR
    - 10.0.0.0/8
```

### Headless Service (StatefulSet DNS)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd
spec:
  clusterIP: None             # Headless
  selector:
    app: etcd
  ports:
    - port: 2379
      name: client

# DNS records created:
# etcd.default.svc.cluster.local → A records for ALL pod IPs
# etcd-0.etcd.default.svc.cluster.local → Pod 0 IP
# etcd-1.etcd.default.svc.cluster.local → Pod 1 IP
# etcd-2.etcd.default.svc.cluster.local → Pod 2 IP
```

### ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.legacy.example.com   # CNAME record
# No selector, no endpoints — just DNS alias
```

---

## 3. Endpoints and EndpointSlices

### Endpoints

```bash
# View endpoints for a service
kubectl get endpoints storage-api
# NAME          ENDPOINTS                                   AGE
# storage-api   10.244.1.5:8080,10.244.2.3:8080,10.244.3.7:8080

# Manual endpoints (no selector service)
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
      - ip: 192.168.1.100
      - ip: 192.168.1.101
    ports:
      - port: 3306
```

### EndpointSlices (K8s 1.21+ default)

```yaml
# Auto-created by EndpointSlice controller
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: storage-api-abc12
  labels:
    kubernetes.io/service-name: storage-api
addressType: IPv4
endpoints:
  - addresses: ["10.244.1.5"]
    conditions:
      ready: true
      serving: true
      terminating: false
    nodeName: worker-1
    zone: us-east-1a
  - addresses: ["10.244.2.3"]
    conditions:
      ready: true
    nodeName: worker-2
ports:
  - name: http
    port: 8080
    protocol: TCP
```

**Why EndpointSlices?** Endpoints objects don't scale — one object for all endpoints (1000+ pods = huge object). EndpointSlices split into chunks of 100 endpoints each, reducing API server and kube-proxy load.

---

## 4. DNS (CoreDNS)

### DNS Resolution

```
┌───────────────────────────────────────────────────────┐
│                    DNS Records                         │
│                                                       │
│  Service:                                             │
│  <service>.<namespace>.svc.cluster.local              │
│  storage-api.default.svc.cluster.local → 10.96.0.50  │
│                                                       │
│  Headless Service (per pod):                          │
│  <pod>.<service>.<namespace>.svc.cluster.local        │
│  etcd-0.etcd.default.svc.cluster.local → 10.244.1.5  │
│                                                       │
│  Pod DNS:                                             │
│  <pod-ip-dashed>.<namespace>.pod.cluster.local        │
│  10-244-1-5.default.pod.cluster.local → 10.244.1.5   │
│                                                       │
│  SRV Records (named ports):                           │
│  _http._tcp.storage-api.default.svc.cluster.local     │
│  → 0 100 8080 storage-api.default.svc.cluster.local   │
└───────────────────────────────────────────────────────┘
```

### Pod DNS Policy

```yaml
spec:
  dnsPolicy: ClusterFirst    # Default — use CoreDNS
  # ClusterFirst: Use cluster DNS, fallback to upstream
  # Default: Inherit node's DNS config
  # ClusterFirstWithHostNet: ClusterFirst for hostNetwork pods
  # None: Use custom dnsConfig below
  
  dnsConfig:
    nameservers:
      - 1.1.1.1
    searches:
      - storage.svc.cluster.local
      - svc.cluster.local
    options:
      - name: ndots
        value: "3"
```

### CoreDNS Configuration

```yaml
# ConfigMap: coredns in kube-system
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### ndots and DNS Performance

```
ndots=5 (default in K8s)
Search domains: default.svc.cluster.local, svc.cluster.local, cluster.local

Lookup "google.com" (1 dot < 5 ndots):
  1. google.com.default.svc.cluster.local  → NXDOMAIN
  2. google.com.svc.cluster.local          → NXDOMAIN
  3. google.com.cluster.local              → NXDOMAIN
  4. google.com.                           → Resolved!

Problem: 4 DNS queries for external names!
Fix: Use FQDN with trailing dot: "google.com."
     Or reduce ndots: 2
```

---

## 5. Ingress

HTTP/HTTPS routing with path/host-based rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: storage-api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
spec:
  ingressClassName: nginx
  
  tls:
    - hosts:
        - storage.dell.com
      secretName: storage-tls
  
  rules:
    - host: storage.dell.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: storage-api
                port:
                  number: 80
          - path: /metrics
            pathType: Exact
            backend:
              service:
                name: prometheus
                port:
                  number: 9090
  
  # Default backend
  defaultBackend:
    service:
      name: default-backend
      port:
        number: 80
```

### Path Types

| Type | Match Behavior |
|------|---------------|
| `Exact` | URL must match exactly: `/api` matches `/api` only |
| `Prefix` | URL must start with: `/api` matches `/api`, `/api/v1`, `/api/foo` |
| `ImplementationSpecific` | Driver-dependent matching |

---

## 6. Gateway API (K8s 1.27+ GA)

Next-generation ingress — more expressive, role-oriented, and portable.

```
┌─────────────────────────────────────────────┐
│            Gateway API Model                 │
│                                             │
│  Infra Admin:   GatewayClass                │
│                 (which controller)           │
│                     │                       │
│  Cluster Admin: Gateway                     │
│                 (listeners, TLS)            │
│                     │                       │
│  App Developer: HTTPRoute / GRPCRoute       │
│                 (routing rules)             │
└─────────────────────────────────────────────┘
```

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
---
# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: wildcard-tls
---
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: storage-route
spec:
  parentRefs:
    - name: main-gateway
  hostnames:
    - storage.dell.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: storage-api
          port: 80
          weight: 90       # Traffic splitting!
        - name: storage-api-canary
          port: 80
          weight: 10
```

---

## 7. Network Policies

Microsegmentation — firewall rules at the pod level.

### Default Allow (no policy = all traffic allowed)

```yaml
# Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # Apply to ALL pods in namespace
  policyTypes:
    - Ingress              # Block all incoming traffic
---
# Default deny all egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: storage-api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: storage-api
  
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # Allow from frontend pods
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
    
    # Allow from monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - protocol: TCP
          port: 9090
    
    # Allow from CIDR (external)
    - from:
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24
      ports:
        - protocol: TCP
          port: 443
  
  egress:
    # Allow to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### AND vs OR Logic

```yaml
# OR: Each array item is OR'd
ingress:
  - from:
      - podSelector: {matchLabels: {app: web}}   # OR
      - namespaceSelector: {matchLabels: {env: prod}}

# AND: Items in same array element are AND'd
ingress:
  - from:
      - podSelector: {matchLabels: {app: web}}
        namespaceSelector: {matchLabels: {env: prod}}
# ^ Must match BOTH: pod label AND namespace label
```

---

## 8. CNI Plugins

| Plugin | Mechanism | Key Feature |
|--------|-----------|-------------|
| **Calico** | BGP / VXLAN / eBPF | Network policy, high performance |
| **Cilium** | eBPF (kernel) | L7 policy, service mesh, observability |
| **Flannel** | VXLAN | Simple, no network policy |
| **Weave** | VXLAN / sleeve | Encryption, ease of use |
| **Multus** | Meta-CNI | Multiple NICs per pod (storage networks) |

### Calico vs Cilium

| Feature | Calico | Cilium |
|---------|--------|--------|
| Data plane | iptables / eBPF | eBPF (native) |
| Network policy | K8s + extended | K8s + L7 (HTTP, gRPC) |
| Performance | Excellent | Excellent (no iptables) |
| Observability | Flow logs | Hubble (flow visibility) |
| Service mesh | — | Sidecar-free (eBPF) |
| Encryption | WireGuard | WireGuard / IPsec |

### Multus CNI (Dell Storage Use Case)

```yaml
# NetworkAttachmentDefinition — additional network
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: storage-network
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24"
      }
    }
---
# Pod with multiple networks
apiVersion: v1
kind: Pod
metadata:
  name: csi-node
  annotations:
    k8s.v1.cni.cncf.io/networks: storage-network
spec:
  containers:
    - name: driver
      # eth0 = cluster network (Calico/Cilium)
      # net1 = storage network (iSCSI/NFS traffic)
```

---

## Interview Questions

1. **Explain the Kubernetes networking model.**
   - Every pod gets a unique IP. Pods communicate directly without NAT across nodes. CNI plugin handles routing (overlay/BGP). Services provide stable endpoints via kube-proxy.

2. **How does kube-proxy implement service load balancing?**
   - iptables mode: DNAT rules with random probability for load distribution. O(n) rule evaluation.
   - IPVS mode: hash-table lookup, O(1). Supports rr, lc, dh, sh algorithms. Better for large clusters (10K+ services).

3. **Explain Network Policies. What is a default-deny strategy?**
   - NetworkPolicy defines ingress/egress rules per pod using selectors. Default-deny: apply empty policy with podSelector: {} — blocks all traffic, then add explicit allow rules. Requires CNI with network policy support (Calico, Cilium).

4. **What is a headless service?**
   - clusterIP: None. No load-balanced VIP. DNS returns A records for individual pod IPs. Used with StatefulSets for stable per-pod DNS names.

5. **How does DNS work in Kubernetes? What is the ndots problem?**
   - CoreDNS resolves service/pod names. Default ndots=5 means names with <5 dots get search domain suffixes appended, causing multiple queries for external names. Fix: use FQDNs with trailing dot or reduce ndots.

6. **What is Multus CNI and why is it relevant for storage?**
   - Meta-CNI that attaches multiple networks to pods. Storage workloads need dedicated networks (iSCSI SAN, NFS) separate from cluster traffic for performance and security isolation.

---

*Next: [Chapter 6 — Security](Security.md)*
