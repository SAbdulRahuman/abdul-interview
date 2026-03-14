# Chapter 12 — Service Mesh & Advanced Networking

## 1. Service Mesh Concepts

```
┌────────────────────────────────────────────────────────┐
│                   SERVICE MESH                          │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │              CONTROL PLANE                      │   │
│  │  Config distribution, certificate management,   │   │
│  │  policy enforcement, telemetry aggregation      │   │
│  └──────────────────┬─────────────────────────────┘   │
│                     │ xDS / gRPC                       │
│  ┌──────────────────┼─────────────────────────────┐   │
│  │              DATA PLANE                         │   │
│  │                  │                              │   │
│  │  ┌───────────────┼───────────────┐              │   │
│  │  │    Pod A       │    Pod B      │              │   │
│  │  │  ┌──────┐    ┌┴─────┐  ┌──────┐ ┌──────┐   │   │
│  │  │  │ App  │←──→│Proxy │──│Proxy │→│ App  │   │   │
│  │  │  └──────┘    └──────┘  └──────┘ └──────┘   │   │
│  │  │              (Envoy)   (Envoy)              │   │
│  │  └─────────────────────────────────┘              │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
│  Features: mTLS, traffic management, observability,    │
│  circuit breaking, retries, rate limiting              │
└────────────────────────────────────────────────────────┘
```

---

## 2. Istio

### Architecture

```
Control Plane:
  istiod (single binary):
    - Pilot: service discovery, config distribution (xDS)
    - Citadel: certificate management, mTLS
    - Galley: configuration validation

Data Plane:
  Envoy proxy sidecars (auto-injected via MutatingWebhook)
```

### Key Resources

```yaml
# VirtualService: traffic routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: storage-api
spec:
  hosts:
    - storage-api
  http:
    # Canary: 90/10 split
    - route:
        - destination:
            host: storage-api
            subset: v1
          weight: 90
        - destination:
            host: storage-api
            subset: v2
          weight: 10
      
      # Retries
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure
      
      # Timeout
      timeout: 10s
      
      # Fault injection (testing)
      fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
        abort:
          percentage:
            value: 1
          httpStatus: 503
---
# DestinationRule: traffic policies per subset
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: storage-api
spec:
  host: storage-api
  trafficPolicy:
    # Connection pool
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        maxRequestsPerConnection: 1000
    
    # Circuit breaker
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
    
    # mTLS
    tls:
      mode: ISTIO_MUTUAL
  
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
# Gateway: north-south traffic entry point
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: storage-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: storage-tls
      hosts:
        - storage.dell.com
```

---

## 3. Linkerd

Lightweight alternative to Istio.

| Feature | Istio | Linkerd |
|---------|-------|---------|
| **Proxy** | Envoy (C++) | linkerd2-proxy (Rust) |
| **Resource usage** | Higher | ~50% less memory |
| **Complexity** | High (many CRDs) | Low (simpler model) |
| **mTLS** | Yes (auto) | Yes (auto) |
| **Policy** | L4-L7 | L4-L7 (K8s 1.24+) |
| **Multi-cluster** | Supported | Supported |

---

## 4. Traffic Management Patterns

### Canary Deployment

```yaml
# Istio: gradually shift traffic
# Step 1: 10% canary
- route:
    - destination: {host: app, subset: v1}
      weight: 90
    - destination: {host: app, subset: v2}
      weight: 10

# Step 2: Observe metrics, increase if healthy
# Step 3: 50/50, then 100% v2
```

### Blue-Green Deployment

```yaml
# Switch all traffic at once
# Blue (current): v1
# Green (new): v2

# Before cutover:
- route:
    - destination: {host: app, subset: v1}
      weight: 100

# After cutover:
- route:
    - destination: {host: app, subset: v2}
      weight: 100
# Instant rollback: switch weight back to v1
```

### Circuit Breaking

```
Healthy state: requests flow normally
            │
            ▼
5 consecutive 5xx errors within 30s
            │
            ▼
Circuit OPEN: pod ejected from load balancing
            │
            ▼
Wait 60s (baseEjectionTime)
            │
            ▼
Circuit HALF-OPEN: send test request
            │
     ┌──────┴───────┐
     ▼              ▼
  Success        Failure
  → CLOSED       → OPEN again
  (resume)       (extend ejection)
```

---

## 5. Cilium — eBPF-Based Networking

```
Traditional: Pod → iptables → kube-proxy → Destination
Cilium:      Pod → eBPF program (kernel) → Destination

Benefits:
- No iptables rules (O(1) instead of O(n))
- L7 visibility without sidecar proxies
- Kernel-level network policy enforcement
- WireGuard encryption (transparent)
- Hubble: real-time flow observability
```

### Cilium Network Policy (L7)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: storage-api-l7
spec:
  endpointSelector:
    matchLabels:
      app: storage-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/api/v1/volumes.*"
              - method: POST
                path: "/api/v1/volumes"
              # Deny DELETE at L7!
```

---

## 6. Multus CNI — Multiple Networks

Critical for **storage workloads** needing dedicated data networks.

```
┌──────────────────────────────────────────────┐
│                  Pod                          │
│                                              │
│  eth0 ─── Cluster Network (Calico/Cilium)   │
│           10.244.1.5                         │
│           API traffic, service discovery     │
│                                              │
│  net1 ─── Storage Network (macvlan/SR-IOV)  │
│           192.168.100.10                     │
│           iSCSI data path to Dell array      │
│                                              │
│  net2 ─── Management Network (macvlan)      │
│           172.16.0.10                        │
│           Storage array management API       │
└──────────────────────────────────────────────┘
```

### Use Case: CSI Driver with Dedicated Storage Network

```yaml
# Separate iSCSI traffic from cluster network
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: iscsi-network
  namespace: dell-storage
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24",
        "rangeStart": "192.168.100.10",
        "rangeEnd": "192.168.100.200"
      }
    }
```

---

## Interview Questions

1. **What is a service mesh and when would you use one?**
   - Infrastructure layer for service-to-service communication. Provides mTLS, traffic management, observability, retries, circuit breaking. Use when: many microservices, need zero-trust networking, traffic splitting for canary, detailed service telemetry.

2. **Istio vs Linkerd — how do you choose?**
   - Istio: feature-rich, Envoy proxy, more config options, larger community. Linkerd: lighter, Rust proxy, simpler, lower resource overhead. Choose Linkerd for simplicity; Istio for advanced traffic management.

3. **Why would storage workloads need Multus CNI?**
   - Separate storage data path (iSCSI, NFS) from cluster network. Performance isolation: storage traffic doesn't compete with pod-to-pod traffic. Security: storage network not reachable from application pods. Dell CSI drivers benefit from dedicated SAN access.

4. **How does Cilium differ from traditional CNIs?**
   - eBPF-based: programs loaded into Linux kernel, no iptables. O(1) service routing. L7 network policies (HTTP, gRPC). Sidecar-free service mesh. Hubble for real-time flow visibility. WireGuard encryption.

---

*Next: [Chapter 13 — CI/CD & GitOps](CICDAndGitOps.md)*
