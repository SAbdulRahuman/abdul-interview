# Design a Service Mesh

Examples: Istio, Linkerd, Consul Connect

---

## 1. Requirements

### Functional
- Transparent service-to-service communication (mTLS)
- Traffic management (routing, retries, circuit breaking)
- Observability (metrics, traces, access logs per service)
- Policy enforcement (rate limiting, access control)
- Service discovery and load balancing

### Non-Functional
- Minimal latency overhead (< 1ms per hop)
- Language-agnostic (works with any application framework)
- Gradual adoption (sidecar injection, not rewrite)
- Scalable to thousands of services

---

## 2. High-Level Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                    Control Plane                          │
  │                                                          │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
  │  │  Config API   │  │  Certificate │  │  Telemetry   │  │
  │  │  (Pilot /     │  │  Authority   │  │  Collector   │  │
  │  │   istiod)     │  │  (Citadel)   │  │  (Mixer)     │  │
  │  │              │  │              │  │              │  │
  │  │  Push config │  │  Issue mTLS  │  │  Aggregate   │  │
  │  │  to sidecars │  │  certs       │  │  metrics     │  │
  │  └──────────────┘  └──────────────┘  └──────────────┘  │
  └─────────────────────────┬────────────────────────────────┘
                            │ xDS API (config push)
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Pod A        │  │  Pod B        │  │  Pod C        │
  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
  │ │ App      │ │  │ │ App      │ │  │ │ App      │ │
  │ │ Container│ │  │ │ Container│ │  │ │ Container│ │
  │ └────┬─────┘ │  │ └────┬─────┘ │  │ └────┬─────┘ │
  │      │ localhost│  │      │       │  │      │       │
  │ ┌────▼─────┐ │  │ ┌────▼─────┐ │  │ ┌────▼─────┐ │
  │ │ Sidecar  │ │  │ │ Sidecar  │ │  │ │ Sidecar  │ │
  │ │ Proxy    │◀┼──┼▶│ Proxy    │◀┼──┼▶│ Proxy    │ │
  │ │ (Envoy)  │ │  │ │ (Envoy)  │ │  │ │ (Envoy)  │ │
  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
  └──────────────┘  └──────────────┘  └──────────────┘
                     mTLS encrypted
```

---

## 3. Sidecar Proxy Pattern

```
  Without Service Mesh:
  ┌────────┐    HTTP (plaintext)    ┌────────┐
  │ Svc A  │───────────────────────▶│ Svc B  │
  └────────┘                        └────────┘
  App must implement: retries, circuit breaking, TLS, tracing

  With Service Mesh:
  ┌────────┐   ┌─────────┐   mTLS   ┌─────────┐   ┌────────┐
  │ Svc A  │──▶│ Envoy A │──────────▶│ Envoy B │──▶│ Svc B  │
  │ (app)  │   │(sidecar)│           │(sidecar)│   │ (app)  │
  └────────┘   └─────────┘           └─────────┘   └────────┘
                    │                      │
           All networking concerns handled by proxy:
           • mTLS encryption
           • Retry logic
           • Circuit breaking
           • Load balancing
           • Metrics collection
           • Access logging
           • Distributed tracing (inject headers)

  App code is completely unaware of the mesh
```

---

## 4. Traffic Management

```
  Canary Deployment:
  ┌──────────────────────────────────────────────┐
  │  VirtualService: reviews                     │
  │                                              │
  │  route:                                      │
  │  - destination: reviews-v1                   │
  │    weight: 90                                │
  │  - destination: reviews-v2                   │
  │    weight: 10                                │
  └──────────────────────────────────────────────┘

  A/B Testing (header-based):
  ┌──────────────────────────────────────────────┐
  │  match:                                      │
  │  - headers:                                  │
  │      x-user-group: "beta"                    │
  │    route:                                    │
  │    - destination: reviews-v2                 │
  │  - route:                                    │
  │    - destination: reviews-v1   (default)     │
  └──────────────────────────────────────────────┘

  Fault Injection (testing):
  ┌──────────────────────────────────────────────┐
  │  fault:                                      │
  │    delay:                                    │
  │      percentage: 10%                         │
  │      fixedDelay: 5s                          │
  │    abort:                                    │
  │      percentage: 5%                          │
  │      httpStatus: 500                         │
  └──────────────────────────────────────────────┘
```

---

## 5. Security (mTLS)

```
  Certificate lifecycle:
  ┌──────────────────────────────────────────────┐
  │  1. Control plane CA issues short-lived      │
  │     X.509 certs to each sidecar              │
  │                                              │
  │  2. Sidecar presents cert for all            │
  │     outbound + inbound connections           │
  │                                              │
  │  3. Both sides verify:                       │
  │     Client → verifies server identity        │
  │     Server → verifies client identity        │
  │                                              │
  │  4. Automatic rotation (every 24h)           │
  │                                              │
  │  AuthorizationPolicy:                        │
  │  "Only frontend can call cart-service"       │
  │  "Only payment-service can call bank-api"    │
  └──────────────────────────────────────────────┘
```

---

## 6. Observability

```
  Metrics (automatic, no code changes):
  ┌──────────────────────────────────────────────┐
  │  request_count{source="svc-a", dest="svc-b"} │
  │  request_duration_ms{...}                     │
  │  response_code{code="200"}                    │
  │  tcp_connections_opened{...}                  │
  └──────────────────────────────────────────────┘

  Distributed tracing:
  ┌──────────────────────────────────────────────┐
  │  Envoy injects/propagates trace headers:     │
  │  x-request-id, x-b3-traceid, x-b3-spanid    │
  │                                              │
  │  Trace: frontend → cart → inventory → payment│
  │  Each hop = span with timing + metadata      │
  └──────────────────────────────────────────────┘

  Service graph:
  Automatically generated from sidecar telemetry
  Shows request flow, error rates, latency between services
```

---

---

## 7. Low-Level Design (LLD)

### Control Plane API (Istio / Linkerd)
```
  xDS API (Envoy Discovery Service):
  ┌────────────────────────────────────────────────────────┐
  │  LDS (Listener Discovery): which ports to listen on   │
  │  RDS (Route Discovery): routing rules per listener     │
  │  CDS (Cluster Discovery): upstream service endpoints   │
  │  EDS (Endpoint Discovery): individual pod IPs          │
  │  SDS (Secret Discovery): TLS certificates              │
  │                                                        │
  │  Flow:                                                 │
  │  1. Istiod watches K8s API (Services, Pods, VirtualSvc)│
  │  2. Translates to Envoy xDS config                     │
  │  3. Push to all sidecar Envoys via gRPC stream          │
  │  4. Envoys apply config hot-reload (no restart)        │
  │  5. Config convergence: ~2-5 seconds cluster-wide       │
  └────────────────────────────────────────────────────────┘

  Istio Custom Resources:
  POST /apis/networking.istio.io/v1/namespaces/{ns}/virtualservices
       { "hosts": ["reviews"], "http": [
           { "match": [{"headers":{"end-user":{"exact":"jason"}}}],
             "route": [{"destination":{"host":"reviews","subset":"v2"}}] },
           { "route": [{"destination":{"host":"reviews","subset":"v1"}}] }
       ] }

  POST /apis/networking.istio.io/v1/namespaces/{ns}/destinationrules
       { "host": "reviews", "trafficPolicy": {
           "connectionPool": {"tcp":{"maxConnections":100}},
           "outlierDetection": {"consecutive5xxErrors":5, "interval":"10s",
                                "baseEjectionTime":"30s"} },
         "subsets": [{"name":"v1","labels":{"version":"v1"}},
                     {"name":"v2","labels":{"version":"v2"}}] }
```

### Sidecar Proxy Internals
```
  Envoy Filter Chain (per connection):
  ┌────────────────────────────────────────────────────────┐
  │  Inbound (request to this pod):                        │
  │  iptables REDIRECT → Envoy (port 15006)               │
  │    → TLS termination (verify client cert)              │
  │    → Authorization policy check                        │
  │    → Route to localhost:app_port                        │
  │                                                        │
  │  Outbound (request from this pod):                     │
  │  App → localhost:dest_port                              │
  │    → iptables REDIRECT → Envoy (port 15001)            │
  │    → Service discovery (EDS: pick healthy endpoint)    │
  │    → Load balancing (round-robin / least-request)      │
  │    → mTLS handshake with destination sidecar            │
  │    → Retry on 503 (configurable)                       │
  │    → Emit metrics + access log                         │
  └────────────────────────────────────────────────────────┘

  iptables Rules (init container sets up):
  -A PREROUTING -p tcp -j ISTIO_INBOUND
  -A ISTIO_INBOUND -p tcp --dport {app_port} -j REDIRECT --to-port 15006
  -A OUTPUT -p tcp -j ISTIO_OUTPUT
  -A ISTIO_OUTPUT ! -d 127.0.0.1 -j REDIRECT --to-port 15001
```

---

## 8. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Control Plane:                                         │
  │  • Istiod: 1-3 replicas (handles 10K+ sidecars)        │
  │  • Config push: delta xDS (only send changes)           │
  │  • Scale Istiod horizontally for > 10K pods            │
  │                                                         │
  │  Data Plane:                                            │
  │  • One Envoy per pod (no shared state)                   │
  │  • Each Envoy: ~50MB RAM, ~0.1 CPU baseline             │
  │  • 10K pods → ~500 GB mesh memory overhead              │
  │  • Optimize: limit Envoy config scope (Sidecar resource)│
  │    - Only push routes for services this pod talks to    │
  │    - Reduces memory from 50MB → 20MB per sidecar       │
  │                                                         │
  │  Large Cluster (>10K services):                         │
  │  • Namespace-scoped mesh (gradual rollout)              │
  │  • Ambient mesh (Istio): shared ztunnel per-node        │
  │    instead of per-pod sidecar → 10× fewer proxies      │
  │  • Proxyless gRPC: xDS directly in app, no sidecar     │
  └─────────────────────────────────────────────────────────┘
```

---

## 9. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Mesh is Stateless (Data Plane):                        │
  │  • Sidecars are proxies — no data stored                │
  │  • TLS certificates: auto-rotated (24h default)        │
  │  • Config: pushed from control plane, cached locally    │
  │                                                         │
  │  Request Reliability:                                   │
  │  • Automatic retries: retry on 503, connection reset   │
  │  • Retry budget: max 20% of requests can be retries    │
  │  • Timeout enforcement: prevent indefinite hangs        │
  │  • Request hedging: send to 2 backends, use first reply│
  │                                                         │
  │  Observability Data Protection:                         │
  │  • Metrics: scraped by Prometheus (pull model)          │
  │  • Traces: buffered in Envoy, async export to Jaeger   │
  │  • Access logs: async write, overrun → drop (not block)│
  │  • If collector is down: buffer locally, resume later   │
  │                                                         │
  │  Certificate Management:                                │
  │  • Istiod as CA: signs SVID certificates                │
  │  • Cert rotation: before expiry, no downtime            │
  │  • Root CA: stored in K8s secret (etcd encrypted)       │
  │  • External CA: Vault, cert-manager integration          │
  └─────────────────────────────────────────────────────────┘
```

---

## 10. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Sidecar overhead (per hop)   │ 0.5 ms   │ 2 ms    │
  │ Full call (2 sidecars)       │ 1 ms     │ 4 ms    │
  │ mTLS handshake (first call)  │ 2 ms     │ 10 ms   │
  │ mTLS (connection reuse)      │ 0 ms     │ 0.1 ms  │
  │ Config propagation           │ 1 s      │ 5 s     │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Connection pooling: reuse mTLS connections           │
  │  • HTTP/2: multiplex requests on single connection      │
  │  • Locality-aware routing: prefer same-zone backends    │
  │  • Proxyless gRPC: eliminate sidecar hop entirely       │
  │  • Wasm plugin: avoid Lua overhead for custom logic     │
  │  • Limit Envoy config scope: faster config processing  │
  │  • TCP Keepalive: avoid re-establishing connections     │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Sidecar crash          │ Pod restarts; iptables still    │
  │                        │ active → traffic blackholed     │
  │                        │ Fix: readiness probe on sidecar │
  ├────────────────────────┼─────────────────────────────────┤
  │ Istiod (control plane) │ Sidecars continue with last     │
  │ down                   │ known config; no new cert       │
  │                        │ rotation until Istiod returns   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Backend overloaded     │ Circuit breaker: outlier        │
  │                        │ detection ejects unhealthy pod  │
  │                        │ after 5 consecutive 5xx errors  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Cascading failure      │ Retry budget (20%); timeout     │
  │                        │ enforcement; circuit breakers   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Bad canary deployment  │ Fault injection: test with 1%   │
  │                        │ traffic; automated rollback     │
  │                        │ based on error rate threshold   │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 12. Availability

```
  Target: 99.99% for data plane; 99.9% for control plane

  ┌─────────────────────────────────────────────────────────┐
  │  Data Plane (always on critical path):                  │
  │  • Sidecar lifecycle tied to pod (1:1)                   │
  │  • If sidecar fails, pod restarts → back in ~10s        │
  │  • Design: sidecar must never block app traffic         │
  │  • Wasm plugin crash isolation: don't crash Envoy       │
  │                                                         │
  │  Control Plane HA:                                      │
  │  • 3 Istiod replicas across AZs                         │
  │  • Leader election for cert signing                     │
  │  • Sidecars survive control plane outage indefinitely   │
  │  • Cert validity: 24h → 24h window to fix Istiod       │
  │                                                         │
  │  Graceful Degradation:                                  │
  │  • Permissive mTLS: accept both plain + mTLS during     │
  │    migration (avoid hard cutover breaking traffic)      │
  │  • Fail-open on auth policy: if policy engine down,     │
  │    allow traffic (configurable per workload)            │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Zero-Trust Networking:                                 │
  │  • mTLS everywhere: all pod-to-pod traffic encrypted    │
  │  • SPIFFE identity: spiffe://cluster/ns/sa              │
  │  • No IP-based trust: identity from certificate         │
  │                                                         │
  │  Authorization Policies:                                │
  │  • L7 RBAC: allow/deny by source identity + path       │
  │  • "frontend" can call "api" on GET /products only      │
  │  • Deny-by-default: explicit allow per service pair     │
  │                                                         │
  │  Certificate Management:                                │
  │  • Auto-provisioned per-pod certs (no manual PKI)       │
  │  • Short-lived (24h) → compromise window is limited     │
  │  • Root CA rotation: staged, zero-downtime              │
  │                                                         │
  │  Audit & Compliance:                                    │
  │  • Access logs: who called whom, when, result           │
  │  • Distributed tracing: end-to-end request visibility  │
  │  • Policy audit: all allow/deny decisions logged        │
  │                                                         │
  │  External Threats:                                      │
  │  • Mesh boundary: ingress gateway for external traffic │
  │  • All internal traffic: encrypted, never plaintext    │
  │  • Sidecar: reject connections from outside mesh        │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Sidecar Overhead (per pod):                            │
  │  • CPU: ~100m (0.1 core) baseline                       │
  │  • Memory: ~50 MB                                       │
  │  • 1000 pods: 100 cores + 50 GB RAM overhead            │
  │  • ~$3K/month extra compute cost at 1000 pods           │
  │                                                         │
  │  Control Plane:                                         │
  │  • Istiod: 3 × 2 vCPU = 6 vCPU total (~$200/month)     │
  │  • Prometheus (metrics): ~$500/month at 1000 pods       │
  │  • Jaeger (traces): ~$300/month                         │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Ambient mesh: shared ztunnel per node vs per-pod     │
  │    → 10× fewer proxy instances → 90% cost reduction    │
  │  • Proxyless gRPC for latency-sensitive services         │
  │  • Limit sidecar config scope (Sidecar resource)         │
  │  • Disable access logs for high-throughput services     │
  │                                                         │
  │  Total (1000-pod cluster):                              │
  │  • Sidecar model: ~$4K/month overhead                   │
  │  • Ambient model: ~$800/month overhead                  │
  │  • ROI: mTLS + observability + traffic control would    │
  │    cost $10K+ to build custom                           │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Service mesh vs API gateway?** — Mesh: east-west (internal). Gateway: north-south (external). Often used together
2. **What's the latency overhead?** — ~1ms per hop (2 proxy hops per call). Acceptable for most workloads, may matter for ultra-low-latency
3. **Sidecar vs proxyless mesh?** — Sidecar: language-agnostic, consistent. Proxyless (gRPC xDS): lower latency, but only for gRPC
4. **How to handle sidecar resource consumption?** — Each Envoy uses ~50MB RAM, ~0.1 CPU. At 10K pods = significant. Use resource limits
5. **Gradual adoption?** — Inject sidecars namespace-by-namespace. Permissive mTLS mode allows plaintext + mTLS during migration
