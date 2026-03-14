# Kubernetes — Interview Preparation Guide

> Comprehensive Kubernetes preparation for storage infrastructure and cloud-native platform engineering roles. Covers architecture, workloads, storage, networking, security, operators, and operational topics.

---

## Chapter 1 — [Architecture & Components](k8s/ArchitectureAndComponents.md)

- [ ] Kubernetes cluster architecture — control plane + worker nodes
- [ ] kube-apiserver — RESTful API, authentication, authorization, admission control
- [ ] etcd — distributed key-value store, Raft consensus, cluster state
- [ ] kube-scheduler — pod placement, filtering, scoring, scheduling plugins
- [ ] kube-controller-manager — reconciliation loops (deployment, replicaset, node, job controllers)
- [ ] cloud-controller-manager — cloud provider integration
- [ ] kubelet — node agent, pod lifecycle, container runtime interface (CRI)
- [ ] kube-proxy — service networking, iptables/IPVS modes
- [ ] Container runtimes — containerd, CRI-O, runc, crun
- [ ] API request flow — kubectl → API server → etcd → controllers → kubelet
- [ ] Control loops / reconciliation pattern — desired state vs actual state

---

## Chapter 2 — [Workload Resources](k8s/WorkloadResources.md)

- [ ] Pod — smallest deployable unit, co-located containers, shared network/storage
- [ ] Init containers — run before app containers, sequential execution
- [ ] Sidecar containers (K8s 1.28+) — native sidecar support with restartPolicy
- [ ] ReplicaSet — ensures desired number of pod replicas
- [ ] Deployment — declarative updates, rolling updates, rollback
- [ ] StatefulSet — stable identity, ordered deployment, persistent storage
- [ ] DaemonSet — one pod per node (logging, monitoring, storage agents)
- [ ] Job — run-to-completion workloads, parallelism, backoff limits
- [ ] CronJob — scheduled jobs, cron syntax, concurrency policies
- [ ] Pod lifecycle — Pending → Running → Succeeded/Failed
- [ ] Pod conditions — PodScheduled, Initialized, ContainersReady, Ready
- [ ] Container probes — liveness, readiness, startup (httpGet, tcpSocket, exec, grpc)
- [ ] Restart policies — Always, OnFailure, Never
- [ ] Pod disruption budgets (PDB) — minAvailable, maxUnavailable

---

## Chapter 3 — [Configuration & Secrets](k8s/ConfigurationAndSecrets.md)

- [ ] ConfigMaps — key-value configuration, volume mount vs environment variable
- [ ] Secrets — base64-encoded, types (Opaque, tls, dockerconfigjson, service-account-token)
- [ ] Environment variables — static, from ConfigMap/Secret, fieldRef (downward API)
- [ ] Downward API — expose pod/container metadata (name, namespace, labels, resources)
- [ ] Resource requests and limits — CPU (millicores), memory (Mi/Gi)
- [ ] QoS classes — Guaranteed, Burstable, BestEffort
- [ ] LimitRanges — per-container/pod default limits in a namespace
- [ ] ResourceQuotas — per-namespace aggregate resource limits
- [ ] PriorityClasses — pod scheduling priority, preemption
- [ ] RuntimeClasses — select container runtime per workload

---

## Chapter 4 — [Storage in Kubernetes](k8s/StorageInKubernetes.md)

- [ ] Volumes — emptyDir, hostPath, configMap, secret, projected, downwardAPI
- [ ] Persistent Volumes (PV) — cluster-scoped storage resource, access modes (RWO, ROX, RWX, RWOP)
- [ ] Persistent Volume Claims (PVC) — namespace-scoped request for storage
- [ ] Storage Classes — dynamic provisioning, reclaimPolicy (Retain, Delete), volumeBindingMode
- [ ] CSI (Container Storage Interface) — standard for storage plugins
- [ ] CSI architecture — external-provisioner, external-attacher, external-snapshotter, node-driver-registrar
- [ ] Volume snapshots — VolumeSnapshot, VolumeSnapshotClass, VolumeSnapshotContent
- [ ] Volume cloning — creating PVC from existing PVC
- [ ] Volume expansion — online/offline resize, allowVolumeExpansion
- [ ] Ephemeral volumes — generic ephemeral volumes, CSI ephemeral volumes
- [ ] Storage capacity tracking — CSIStorageCapacity objects
- [ ] Topology-aware provisioning — volumeBindingMode: WaitForFirstConsumer
- [ ] Raw block volumes vs filesystem volumes
- [ ] fsGroup and volume permissions — securityContext.fsGroup
- [ ] CSI drivers for Dell — PowerFlex CSI, PowerScale CSI, PowerStore CSI, Unity CSI

---

## Chapter 5 — [Networking](k8s/Networking.md)

- [ ] Kubernetes networking model — every pod gets a unique IP, flat network
- [ ] Container-to-container — localhost (shared network namespace)
- [ ] Pod-to-pod — CNI plugins handle routing
- [ ] Pod-to-service — kube-proxy (iptables/IPVS), ClusterIP
- [ ] External-to-service — NodePort, LoadBalancer, Ingress
- [ ] Services — ClusterIP, NodePort, LoadBalancer, ExternalName, headless (clusterIP: None)
- [ ] Endpoints and EndpointSlices
- [ ] DNS (CoreDNS) — service discovery, `<svc>.<ns>.svc.cluster.local`
- [ ] Ingress — HTTP/HTTPS routing, TLS termination, path-based/host-based routing
- [ ] Ingress controllers — NGINX, Traefik, HAProxy, AWS ALB
- [ ] Gateway API (K8s 1.27+ GA) — GatewayClass, Gateway, HTTPRoute
- [ ] Network Policies — ingress/egress rules, pod selectors, namespace selectors
- [ ] CNI plugins — Calico, Cilium, Flannel, Weave, Multus (multi-network)
- [ ] Service Mesh integration — Istio, Linkerd sidecar injection

---

## Chapter 6 — [Security](k8s/Security.md)

- [ ] Authentication — X.509 certificates, service account tokens, OIDC, webhook
- [ ] Authorization — RBAC, ABAC, webhook, Node authorizer
- [ ] RBAC — Role, ClusterRole, RoleBinding, ClusterRoleBinding
- [ ] ServiceAccounts — per-pod identity, token projection, bound service account tokens
- [ ] Pod Security Standards — Privileged, Baseline, Restricted
- [ ] Pod Security Admission (PSA) — enforce/audit/warn modes per namespace
- [ ] SecurityContext — runAsUser, runAsGroup, runAsNonRoot, readOnlyRootFilesystem
- [ ] Capabilities — drop ALL, add specific (NET_BIND_SERVICE, SYS_PTRACE)
- [ ] Seccomp profiles — restricting system calls
- [ ] AppArmor / SELinux — mandatory access control
- [ ] Network Policies — microsegmentation, deny-all default
- [ ] Secrets management — external secrets (Vault, AWS Secrets Manager), sealed-secrets
- [ ] Image security — image signing, admission webhooks, vulnerability scanning
- [ ] Audit logging — policy rules, event levels (None, Metadata, Request, RequestResponse)
- [ ] mTLS — service-to-service encryption (Istio, Linkerd, cert-manager)

---

## Chapter 7 — [Scheduling & Resource Management](k8s/SchedulingAndResources.md)

- [ ] Scheduling framework — filtering → scoring → binding
- [ ] nodeSelector — simple label-based node selection
- [ ] Node affinity — requiredDuringScheduling, preferredDuringScheduling
- [ ] Pod affinity / anti-affinity — co-locate or spread pods
- [ ] Taints and tolerations — NoSchedule, PreferNoSchedule, NoExecute
- [ ] Topology spread constraints — maxSkew, topologyKey, whenUnsatisfiable
- [ ] Pod priority and preemption — PriorityClass, preemptionPolicy
- [ ] Resource requests vs limits — scheduling based on requests, throttling/OOM based on limits
- [ ] Horizontal Pod Autoscaler (HPA) — CPU, memory, custom metrics
- [ ] Vertical Pod Autoscaler (VPA) — right-sizing resource requests
- [ ] Cluster Autoscaler — scale node pools based on pending pods
- [ ] KEDA — event-driven autoscaling (Kafka, queue depth, Prometheus metrics)
- [ ] Descheduler — rebalance pods after cluster changes

---

## Chapter 8 — [Operators & Custom Resources](k8s/OperatorsAndCRDs.md)

- [ ] Custom Resource Definitions (CRDs) — extending the Kubernetes API
- [ ] Custom Resources (CRs) — instances of CRDs
- [ ] Operator pattern — CRD + custom controller for domain-specific automation
- [ ] Controller-runtime — Reconcile loop, client, scheme, manager
- [ ] Kubebuilder — scaffolding, markers, webhook generation
- [ ] Operator SDK — Go, Ansible, Helm-based operators
- [ ] Reconciliation loop — observe → diff → act, idempotent operations
- [ ] Owner references and garbage collection — cascading deletes
- [ ] Finalizers — pre-deletion hooks for cleanup
- [ ] Admission webhooks — MutatingAdmissionWebhook, ValidatingAdmissionWebhook
- [ ] Conversion webhooks — CRD version migration
- [ ] Status subresource — .status vs .spec separation
- [ ] Operator lifecycle manager (OLM) — packaging, distribution, upgrades
- [ ] Leader election — controller HA using lease-based election
- [ ] OperatorHub.io — publishing and discovery

---

## Chapter 9 — [Helm & Application Packaging](k8s/HelmAndPackaging.md)

- [ ] Helm architecture — charts, releases, repositories
- [ ] Chart structure — Chart.yaml, values.yaml, templates/, charts/
- [ ] Template syntax — Go templates, {{ .Values.x }}, {{ include }}, {{ toYaml }}
- [ ] Built-in objects — .Release, .Values, .Chart, .Capabilities, .Template
- [ ] Values — default values, override with -f or --set
- [ ] Hooks — pre-install, post-install, pre-upgrade, pre-delete, test
- [ ] Dependencies — Chart.lock, condition/tags for optional sub-charts
- [ ] Release management — install, upgrade, rollback, uninstall, history
- [ ] Helm test — validate release health
- [ ] Kustomize — base + overlays, patches, generators, no templating
- [ ] Helm vs Kustomize — when to use each

---

## Chapter 10 — [Observability](k8s/Observability.md)

- [ ] Metrics pipeline — metrics-server, Prometheus, Prometheus Operator
- [ ] Prometheus — scrape model, PromQL, ServiceMonitor/PodMonitor CRDs
- [ ] Grafana — dashboards, alerting, data sources
- [ ] Logging — node-level (Fluentd/Fluent Bit → Elasticsearch/Loki)
- [ ] Log architecture — sidecar vs DaemonSet log collectors
- [ ] Distributed tracing — OpenTelemetry, Jaeger, Tempo
- [ ] OpenTelemetry Operator — auto-instrumentation, collector deployment
- [ ] kubectl debugging — logs, describe, events, exec, debug
- [ ] Kubernetes events — Warning/Normal, event aggregation
- [ ] Audit logs — API server request logging
- [ ] Resource monitoring — kube-state-metrics, node-exporter
- [ ] SLIs/SLOs — error rate, latency, availability targets

---

## Chapter 11 — [Cluster Operations](k8s/ClusterOperations.md)

- [ ] Cluster upgrades — control plane first, then nodes, version skew policy
- [ ] Node operations — cordon, drain, uncordon
- [ ] etcd operations — backup, restore, defragmentation, snapshot
- [ ] Certificate management — CA rotation, kubelet certificate rotation
- [ ] HA control plane — stacked etcd vs external etcd, multi-master
- [ ] Cluster API (CAPI) — declarative cluster lifecycle management
- [ ] Node problem detector — detect node issues (disk pressure, NTP drift, kernel deadlock)
- [ ] Disaster recovery — etcd backup strategies, velero for workload backup
- [ ] Capacity planning — resource utilization monitoring, right-sizing
- [ ] Cluster federation — multi-cluster management (KubeFed, Liqo)

---

## Chapter 12 — [Service Mesh & Advanced Networking](k8s/ServiceMeshAndAdvancedNetworking.md)

- [ ] Service mesh concepts — sidecar proxy, data plane, control plane
- [ ] Istio — Envoy sidecar, VirtualService, DestinationRule, Gateway
- [ ] Linkerd — lightweight mesh, mTLS, traffic splitting
- [ ] Traffic management — canary, blue-green, A/B testing, circuit breaking
- [ ] Observability — distributed tracing, golden signals via mesh
- [ ] mTLS — automatic certificate rotation, zero-trust networking
- [ ] Cilium — eBPF-based networking, sidecar-free service mesh
- [ ] Multus CNI — multiple network interfaces per pod (storage networks)

---

## Chapter 13 — [CI/CD & GitOps](k8s/CICDAndGitOps.md)

- [ ] GitOps principles — git as source of truth, reconciliation, declarative
- [ ] ArgoCD — Application CRD, sync policies, app-of-apps pattern
- [ ] Flux — GitRepository, Kustomization, HelmRelease controllers
- [ ] CI pipelines — build, test, scan, push image, update manifests
- [ ] Image update automation — ArgoCD Image Updater, Flux Image Automation
- [ ] Progressive delivery — Argo Rollouts (canary, blue-green, analysis)
- [ ] Tekton — cloud-native CI/CD pipelines as Kubernetes resources

---

## Chapter 14 — [Troubleshooting](k8s/Troubleshooting.md)

- [ ] Pod not starting — ImagePullBackOff, CrashLoopBackOff, Pending, CreateContainerConfigError
- [ ] Debugging pending pods — insufficient resources, taints, affinity, PVC binding
- [ ] Debugging CrashLoopBackOff — logs, describe, exec, ephemeral debug containers
- [ ] Service not reachable — endpoint check, DNS, network policies, kube-proxy
- [ ] Node issues — NotReady, disk pressure, memory pressure, PID pressure
- [ ] Storage issues — PVC pending, mount errors, volume permissions
- [ ] kubectl debug — ephemeral containers, node debugging
- [ ] Networking debugging — DNS resolution, pod-to-pod connectivity, curl/nslookup
- [ ] Event investigation — kubectl get events --sort-by=.lastTimestamp
- [ ] Resource exhaustion — CPU throttling, OOMKilled, eviction

---

## Chapter 15 — [Interview Questions](k8s/InterviewQuestions.md)

### Architecture
1. Explain the Kubernetes control plane components and their roles.
2. What happens when you run `kubectl apply -f deployment.yaml`? Trace the full flow.
3. How does etcd store cluster state? What happens if etcd fails?
4. Explain the reconciliation pattern. Why is it central to Kubernetes?

### Workloads
5. When would you use StatefulSet vs Deployment?
6. What is the difference between liveness, readiness, and startup probes?
7. How do rolling updates work in a Deployment? What is maxSurge and maxUnavailable?
8. How do you handle graceful shutdown in Kubernetes (preStop hooks, SIGTERM)?

### Storage (Critical for Dell)
9. Explain the CSI architecture. How does dynamic provisioning work?
10. What is the difference between RWO, RWX, and ROX access modes?
11. How do volume snapshots work in Kubernetes?
12. Explain topology-aware volume provisioning and why it matters.
13. How does a CSI driver handle node failure and volume failover?
14. Design a stateful application (database) deployment with persistent storage on Kubernetes.

### Networking
15. How does kube-proxy implement service load balancing (iptables vs IPVS)?
16. Explain Network Policies. How would you implement a default-deny policy?
17. How does DNS work in Kubernetes? What is a headless service?

### Security
18. Explain RBAC in Kubernetes. How do Role, ClusterRole, and bindings work?
19. What are Pod Security Standards? How do you enforce them?
20. How do you manage secrets securely in Kubernetes?

### Operations
21. Walk through a Kubernetes cluster upgrade strategy.
22. How do you handle etcd backup and disaster recovery?
23. How does Horizontal Pod Autoscaler work? What metrics can it use?
24. How do you debug a pod stuck in Pending state?

### Operators
25. What is the Operator pattern? When would you build a custom operator?
26. Explain the reconciliation loop in a Kubernetes controller.
27. What are finalizers and why are they important?
28. How do admission webhooks work? Give a use case for mutating and validating webhooks.

### Dell-Specific
29. How would you design a CSI driver for a Dell PowerStore storage backend?
30. How does a storage operator manage the lifecycle of persistent volumes across node failures?

---

## Study Checklist

- [ ] Chapter 1: [Architecture & Components](k8s/ArchitectureAndComponents.md)
- [ ] Chapter 2: [Workload Resources](k8s/WorkloadResources.md)
- [ ] Chapter 3: [Configuration & Secrets](k8s/ConfigurationAndSecrets.md)
- [ ] Chapter 4: [Storage in Kubernetes](k8s/StorageInKubernetes.md)
- [ ] Chapter 5: [Networking](k8s/Networking.md)
- [ ] Chapter 6: [Security](k8s/Security.md)
- [ ] Chapter 7: [Scheduling & Resource Management](k8s/SchedulingAndResources.md)
- [ ] Chapter 8: [Operators & Custom Resources](k8s/OperatorsAndCRDs.md)
- [ ] Chapter 9: [Helm & Application Packaging](k8s/HelmAndPackaging.md)
- [ ] Chapter 10: [Observability](k8s/Observability.md)
- [ ] Chapter 11: [Cluster Operations](k8s/ClusterOperations.md)
- [ ] Chapter 12: [Service Mesh & Advanced Networking](k8s/ServiceMeshAndAdvancedNetworking.md)
- [ ] Chapter 13: [CI/CD & GitOps](k8s/CICDAndGitOps.md)
- [ ] Chapter 14: [Troubleshooting](k8s/Troubleshooting.md)
- [ ] Chapter 15: [Interview Questions](k8s/InterviewQuestions.md)

---

*Last updated: March 14, 2026*
