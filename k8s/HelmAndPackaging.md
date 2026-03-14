# Chapter 9 — Helm & Application Packaging

## 1. Helm Architecture

```
┌─────────────────────────────────────────────┐
│                 Helm v3                      │
│                                             │
│  ┌────────┐     ┌──────────┐               │
│  │ helm   │────→│ K8s API  │               │
│  │ CLI    │     │ Server   │               │
│  └────────┘     └──────────┘               │
│                                             │
│  No Tiller (v2 had server-side Tiller)     │
│  Releases stored as Secrets in namespace   │
│                                             │
│  Charts = packages of K8s manifests        │
│  Releases = installed instances            │
│  Repositories = chart collections          │
└─────────────────────────────────────────────┘
```

---

## 2. Chart Structure

```
mychart/
├── Chart.yaml           # Chart metadata (name, version, dependencies)
├── Chart.lock           # Locked dependency versions
├── values.yaml          # Default configuration values
├── values.schema.json   # Optional: JSON schema for validation
├── templates/
│   ├── _helpers.tpl     # Template helper functions
│   ├── deployment.yaml  
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── pvc.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt        # Post-install message
│   └── tests/
│       └── test-connection.yaml
├── charts/              # Packaged dependency charts
├── crds/                # CRD manifests (installed before templates)
└── README.md
```

### Chart.yaml

```yaml
apiVersion: v2           # Helm 3
name: storage-operator
version: 2.1.0           # Chart version (SemVer)
appVersion: "1.5.0"      # Application version
type: application        # or library
description: Dell Storage Operator for Kubernetes
home: https://github.com/dell/storage-operator
maintainers:
  - name: Dell Storage Team
    email: storage@dell.com

dependencies:
  - name: prometheus
    version: "25.x.x"
    repository: https://prometheus-community.github.io/helm-charts
    condition: prometheus.enabled     # Only include if enabled
  - name: cert-manager
    version: "1.14.x"
    repository: https://charts.jetstack.io
    tags:
      - security
```

---

## 3. Template Syntax

### Go Template Basics

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort | default 8080 }}
          
          {{- if .Values.resources }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
          
          env:
            - name: LOG_LEVEL
              value: {{ .Values.logLevel | quote }}
            {{- range $key, $val := .Values.extraEnv }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
          
          volumeMounts:
            - name: config
              mountPath: /etc/config
      
      volumes:
        - name: config
          configMap:
            name: {{ include "mychart.fullname" . }}-config
```

### Helper Templates (_helpers.tpl)

```yaml
# templates/_helpers.tpl

{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Built-in Objects

| Object | Description |
|--------|-------------|
| `.Values` | Values from values.yaml + overrides |
| `.Release.Name` | Release name |
| `.Release.Namespace` | Namespace |
| `.Release.IsInstall` | True if install (not upgrade) |
| `.Release.IsUpgrade` | True if upgrade |
| `.Chart.Name` | Chart name |
| `.Chart.Version` | Chart version |
| `.Chart.AppVersion` | App version |
| `.Template.BasePath` | templates/ directory path |
| `.Capabilities.KubeVersion` | Kubernetes version |

### Control Flow

```yaml
# Conditionals
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# Range (loops)
{{- range .Values.extraPorts }}
- name: {{ .name }}
  containerPort: {{ .port }}
{{- end }}

# With (scope change)
{{- with .Values.affinity }}
affinity:
  {{- toYaml . | nindent 8 }}
{{- end }}

# Whitespace control
# {{-  trims left whitespace (including newline)
# -}}  trims right whitespace
```

---

## 4. Values

### values.yaml

```yaml
replicaCount: 3

image:
  repository: dell/storage-operator
  tag: ""           # Defaults to Chart.AppVersion
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: storage.dell.local
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

nodeSelector: {}
tolerations: []
affinity: {}

storage:
  arrayId: ""
  protocol: iSCSI
  storageClass:
    create: true
    name: dell-powerstore

prometheus:
  enabled: true

logLevel: info
extraEnv: {}
```

### Override Values

```bash
# Override with -f (file)
helm install myrelease mychart -f production-values.yaml

# Override with --set
helm install myrelease mychart \
  --set replicaCount=5 \
  --set image.tag=v2.0 \
  --set-string storage.arrayId="PS001"

# Override precedence (right wins):
# values.yaml < -f values1.yaml < -f values2.yaml < --set
```

---

## 5. Hooks

Execute resources at specific points in the release lifecycle.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade        # When to run
    "helm.sh/hook-weight": "1"         # Order (lower first)
    "helm.sh/hook-delete-policy": hook-succeeded  # Cleanup
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: dell/storage-migrator:v1
          command: ["./migrate", "--up"]
      restartPolicy: Never
```

### Hook Types

| Hook | When |
|------|------|
| `pre-install` | Before any release resources installed |
| `post-install` | After all resources installed |
| `pre-upgrade` | Before upgrade |
| `post-upgrade` | After upgrade |
| `pre-delete` | Before deletion |
| `post-delete` | After deletion |
| `pre-rollback` | Before rollback |
| `post-rollback` | After rollback |
| `test` | When `helm test` is run |

### Delete Policies

| Policy | Behavior |
|--------|----------|
| `hook-succeeded` | Delete after success |
| `hook-failed` | Delete after failure |
| `before-hook-creation` | Delete old before creating new |

---

## 6. Release Management

```bash
# Install
helm install storage-op dell/storage-operator \
  -n dell-storage --create-namespace \
  -f values.yaml

# List releases
helm list -n dell-storage

# Upgrade
helm upgrade storage-op dell/storage-operator \
  -n dell-storage -f values.yaml --set image.tag=v2.1

# Rollback
helm rollback storage-op 1 -n dell-storage    # Revision 1

# History
helm history storage-op -n dell-storage
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete

# Uninstall
helm uninstall storage-op -n dell-storage

# Dry-run / Template
helm template storage-op dell/storage-operator -f values.yaml
helm install storage-op dell/storage-operator --dry-run --debug
```

---

## 7. Helm Test

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "mychart.fullname" . }}-test
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ['wget', '--spider', 'http://{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

```bash
helm test storage-op -n dell-storage
```

---

## 8. Kustomize

Template-free customization using patches and overlays.

```
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── replica-count.yaml
│   │   └── namespace.yaml
│   └── prod/
│       ├── kustomization.yaml
│       ├── replica-count.yaml
│       ├── resource-limits.yaml
│       └── hpa.yaml
```

### Base kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app: storage-api
```

### Overlay kustomization.yaml (prod)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
  - hpa.yaml
patches:
  - path: replica-count.yaml
  - path: resource-limits.yaml
  - target:
      kind: Deployment
      name: storage-api
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: dell/storage-api:v2.0-prod
images:
  - name: dell/storage-api
    newTag: v2.0-prod
```

```bash
# Apply with kubectl
kubectl apply -k overlays/prod/

# Preview
kubectl kustomize overlays/prod/
```

---

## 9. Helm vs Kustomize

| Feature | Helm | Kustomize |
|---------|------|-----------|
| **Approach** | Templating | Patching/overlays |
| **Complexity** | Go templates (steep curve) | YAML only (simpler) |
| **Packaging** | Charts (distributable) | Directory structure |
| **Versioning** | Chart versions, repos | Git branches/tags |
| **Release mgmt** | Built-in (install, upgrade, rollback) | None (kubectl only) |
| **Dependencies** | Chart dependencies | Manual |
| **Best for** | Distributable packages, 3rd-party apps | Internal apps, environment config |

### When to Use Each

- **Helm**: Installing 3rd-party software (Prometheus, cert-manager, CSI drivers), releasing packaged applications
- **Kustomize**: Environment-specific overlays for internal apps, simple config variations
- **Both**: Helm for base templating + Kustomize for environment patches (`helm template | kustomize`)

---

## Interview Questions

1. **What is Helm and how does v3 differ from v2?**
   - Helm is a K8s package manager. v3 removed Tiller (server component) — direct API access, better security. Releases stored as Secrets. Three-way strategic merge patch for upgrades.

2. **How does Helm handle upgrades and rollbacks?**
   - `helm upgrade` does three-way diff (old manifest vs new manifest vs live state). Creates new release revision. `helm rollback` restores previous revision. Release history stored as Secrets.

3. **Helm vs Kustomize — when to use each?**
   - Helm for distributable packages with templating. Kustomize for environment-specific patches without templating. Can combine both. For Dell: Helm for CSI driver distribution, Kustomize for per-cluster config.

---

*Next: [Chapter 10 — Observability](Observability.md)*
