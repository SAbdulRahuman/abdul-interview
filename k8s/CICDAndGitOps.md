# Chapter 13 — CI/CD & GitOps

## 1. GitOps Principles

```
┌──────────────────────────────────────────────────────┐
│                  GitOps Model                         │
│                                                      │
│  Git Repository (Source of Truth)                    │
│  ┌─────────────────────────┐                         │
│  │ manifests/              │                         │
│  │   deployment.yaml       │  Desired State          │
│  │   service.yaml          │                         │
│  │   configmap.yaml        │                         │
│  └───────────┬─────────────┘                         │
│              │  Watch/Poll                           │
│              ▼                                        │
│  ┌─────────────────────┐    ┌──────────────────┐     │
│  │ GitOps Controller   │───→│ Kubernetes       │     │
│  │ (ArgoCD / Flux)     │    │ Cluster          │     │
│  │                     │    │                  │     │
│  │ Reconcile:          │    │ Actual State     │     │
│  │ Desired ↔ Actual    │    └──────────────────┘     │
│  └─────────────────────┘                              │
│                                                      │
│  Key Principles:                                     │
│  1. Git is the ONLY source of truth                  │
│  2. Declarative desired state                        │
│  3. Automated reconciliation                         │
│  4. All changes via pull request                     │
└──────────────────────────────────────────────────────┘
```

---

## 2. ArgoCD

### Architecture

```
┌──────────────────────────────────────────┐
│              ArgoCD                       │
│                                          │
│  ┌──────────────┐                        │
│  │ API Server   │  Web UI + CLI + API    │
│  └──────┬───────┘                        │
│         │                                │
│  ┌──────┴───────┐                        │
│  │ Application  │  Reconciliation loop   │
│  │ Controller   │  (3min default)        │
│  └──────┬───────┘                        │
│         │                                │
│  ┌──────┴───────┐                        │
│  │ Repo Server  │  Git clone, render     │
│  │              │  (Helm, Kustomize,     │
│  │              │   plain YAML)          │
│  └──────────────┘                        │
└──────────────────────────────────────────┘
```

### Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: storage-operator
  namespace: argocd
spec:
  project: dell-storage
  
  source:
    repoURL: https://github.com/dell/storage-operator.git
    targetRevision: main
    path: deploy/production
    
    # For Helm charts
    # helm:
    #   valueFiles:
    #     - values-prod.yaml
    #   parameters:
    #     - name: image.tag
    #       value: v2.1
    
    # For Kustomize
    # kustomize:
    #   images:
    #     - dell/storage-operator:v2.1
  
  destination:
    server: https://kubernetes.default.svc
    namespace: dell-storage
  
  syncPolicy:
    automated:
      prune: true          # Delete resources removed from Git
      selfHeal: true       # Revert manual changes in cluster
      allowEmpty: false    # Don't sync if source is empty
    
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### App of Apps Pattern

```yaml
# Root application that manages other Applications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-apps
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/dell/platform-config.git
    path: apps/       # Directory containing Application YAMLs
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: {}

# apps/storage-operator.yaml  → deploys storage operator
# apps/monitoring.yaml        → deploys Prometheus stack
# apps/cert-manager.yaml      → deploys cert-manager
```

---

## 3. Flux

### Architecture

```
Source Controller    → Watches Git/Helm/OCI repos
Kustomize Controller → Reconciles Kustomization resources
Helm Controller      → Reconciles HelmRelease resources
Notification Controller → Sends/receives notifications
Image Automation     → Detects new images → updates Git
```

### Flux GitOps Setup

```yaml
# GitRepository: define source
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: storage-platform
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/dell/storage-platform.git
  ref:
    branch: main
  secretRef:
    name: git-credentials
---
# Kustomization: reconcile
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: storage-operator
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: storage-platform
  path: ./deploy/production
  prune: true
  targetNamespace: dell-storage
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: storage-operator
      namespace: dell-storage
  timeout: 3m
---
# HelmRelease: Helm chart reconciliation
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  interval: 10m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: "55.x"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
  values:
    grafana:
      enabled: true
    prometheus:
      retention: 30d
```

---

## 4. CI Pipeline (Build)

```
┌──────────────────────────────────────────────────────┐
│               CI Pipeline (GitHub Actions)            │
│                                                      │
│  Push to main                                        │
│       │                                              │
│       ▼                                              │
│  ┌──────────┐                                        │
│  │  Lint    │  go vet, golangci-lint                 │
│  └────┬─────┘                                        │
│       ▼                                              │
│  ┌──────────┐                                        │
│  │  Test    │  go test ./... -race -coverprofile     │
│  └────┬─────┘                                        │
│       ▼                                              │
│  ┌──────────┐                                        │
│  │  Build   │  docker build, multi-arch              │
│  └────┬─────┘                                        │
│       ▼                                              │
│  ┌──────────┐                                        │
│  │  Scan    │  Trivy (CVE scan), cosign (sign)       │
│  └────┬─────┘                                        │
│       ▼                                              │
│  ┌──────────┐                                        │
│  │  Push    │  Push to registry.dell.com             │
│  └────┬─────┘                                        │
│       ▼                                              │
│  ┌──────────┐                                        │
│  │ Update   │  Update image tag in Git manifests     │
│  │ Manifests│  (ArgoCD Image Updater or Flux IA)     │
│  └──────────┘                                        │
└──────────────────────────────────────────────────────┘
```

---

## 5. Progressive Delivery (Argo Rollouts)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: storage-api
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10        # 10% traffic to canary
        - pause: {duration: 5m}
        - analysis:            # Run automated analysis
            templates:
              - templateName: success-rate
        - setWeight: 30
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100       # Full rollout
      
      canaryService: storage-api-canary
      stableService: storage-api-stable
      
      trafficRouting:
        istio:
          virtualServices:
            - name: storage-api
              routes:
                - primary
  
  selector:
    matchLabels:
      app: storage-api
  template:
    metadata:
      labels:
        app: storage-api
    spec:
      containers:
        - name: api
          image: dell/storage-api:v2.1
---
# AnalysisTemplate: automated canary validation
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] > 0.99
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status=~"2..",app="storage-api",version="canary"}[5m]))
            /
            sum(rate(http_requests_total{app="storage-api",version="canary"}[5m]))
```

---

## 6. Tekton

Cloud-native CI/CD pipelines as Kubernetes resources.

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  params:
    - name: repo-url
    - name: image-name
  
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.repo-url)
    
    - name: test
      taskRef:
        name: golang-test
      runAfter: [clone]
    
    - name: build-image
      taskRef:
        name: kaniko
      runAfter: [test]
      params:
        - name: IMAGE
          value: $(params.image-name)
    
    - name: deploy
      taskRef:
        name: kubernetes-actions
      runAfter: [build-image]
```

---

## Interview Questions

1. **What is GitOps?**
   - Operational model where Git is the single source of truth for infrastructure and applications. A controller (ArgoCD/Flux) continuously reconciles cluster state with Git. Benefits: audit trail, rollback via revert, declarative, reproducible.

2. **ArgoCD vs Flux — how do they compare?**
   - ArgoCD: Web UI, Application CRD, multi-cluster support, RBAC. Flux: lighter, toolkit approach, better Helm/Kustomize integration, image automation built-in. Both: reconciliation loop, Git-based, CNCF projects.

3. **How would you set up CI/CD for a CSI driver?**
   - CI: lint, unit test (envtest), build multi-arch image, CVE scan, sign image. CD: GitOps with ArgoCD — push image tag update to Git, ArgoCD syncs. Use progressive delivery (Argo Rollouts) for canary testing on staging. Separate repos for code and deployment config.

---

*Next: [Chapter 14 — Troubleshooting](Troubleshooting.md)*
