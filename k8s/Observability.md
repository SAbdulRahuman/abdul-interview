# Chapter 10 — Observability

## 1. The Three Pillars

```
┌──────────────────────────────────────────────────────┐
│                OBSERVABILITY                          │
│                                                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐    │
│  │ METRICS  │   │ LOGGING  │   │  TRACING     │    │
│  │          │   │          │   │              │    │
│  │Prometheus│   │Loki/EFK  │   │Jaeger/Tempo  │    │
│  │Grafana   │   │Fluent Bit│   │OpenTelemetry │    │
│  │          │   │          │   │              │    │
│  │ "What is │   │ "What    │   │ "Why is it   │    │
│  │  broken?"│   │ happened?"│  │  slow?"      │    │
│  └──────────┘   └──────────┘   └──────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

## 2. Metrics Pipeline

### Prometheus Architecture

```
┌────────────────────────────────────────────────────────┐
│                    PROMETHEUS                           │
│                                                        │
│  ┌─────────────┐    Scrape    ┌──────────────────┐    │
│  │ Prometheus  │◄─────────────│ Target Pods      │    │
│  │ Server      │   /metrics   │ :9090/metrics    │    │
│  │             │              └──────────────────┘    │
│  │  ┌────────┐ │                                      │
│  │  │ TSDB   │ │    Scrape    ┌──────────────────┐    │
│  │  │(storage)│ │◄─────────────│ node-exporter    │    │
│  │  └────────┘ │              │ (DaemonSet)      │    │
│  │             │              └──────────────────┘    │
│  │  ┌────────┐ │                                      │
│  │  │ PromQL │ │    Scrape    ┌──────────────────┐    │
│  │  │ Engine │ │◄─────────────│ kube-state-metrics│   │
│  │  └────────┘ │              │ (Deployment)      │   │
│  └──────┬──────┘              └──────────────────┘    │
│         │                                              │
│         │ PromQL queries                              │
│         ▼                                              │
│  ┌──────────────┐                                     │
│  │   Grafana    │  Dashboards + Alerting              │
│  └──────────────┘                                     │
│         │                                              │
│  ┌──────────────┐                                     │
│  │ Alertmanager │  Route alerts → Slack/PD/Email      │
│  └──────────────┘                                     │
└────────────────────────────────────────────────────────┘
```

### Prometheus Operator (ServiceMonitor)

```yaml
# ServiceMonitor: tell Prometheus what to scrape
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: storage-api-metrics
  labels:
    release: prometheus    # Must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: storage-api
  namespaceSelector:
    matchNames:
      - dell-storage
  endpoints:
    - port: metrics        # Named port on Service
      interval: 30s
      path: /metrics
      scrapeTimeout: 10s
---
# PodMonitor: scrape pods directly (no Service needed)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: csi-driver-metrics
spec:
  selector:
    matchLabels:
      app: csi-powerstore
  podMetricsEndpoints:
    - port: metrics
      interval: 15s
```

### Key Metrics

```promql
# API server request rate
rate(apiserver_request_total{verb=~"GET|POST|PUT|DELETE"}[5m])

# Pod CPU usage
rate(container_cpu_usage_seconds_total{container!=""}[5m])

# Pod memory usage
container_memory_working_set_bytes{container!=""}

# Node disk pressure
kube_node_status_condition{condition="DiskPressure",status="true"}

# PVC usage (requires kubelet metrics)
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes

# CSI operation latency
histogram_quantile(0.99, rate(csi_operations_seconds_bucket[5m]))

# Pod restart count
kube_pod_container_status_restarts_total
```

### PrometheusRule (Alerting)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
spec:
  groups:
    - name: storage
      rules:
        - alert: PVCNearFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"
        
        - alert: CSIDriverDown
          expr: up{job="csi-powerstore"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "CSI PowerStore driver is down"
        
        - alert: HighPodRestartRate
          expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
          labels:
            severity: warning
```

---

## 3. Logging

### Log Architecture Options

```
Option 1: DaemonSet (recommended)
┌──────────┐    ┌─────────────┐    ┌───────────────┐
│ Container │───→│ Node Disk   │───→│ Fluent Bit    │───→ Loki/ES
│ stdout    │    │ /var/log/   │    │ (DaemonSet)   │
└──────────┘    │ pods/       │    └───────────────┘
                └─────────────┘

Option 2: Sidecar
┌──────────┐    ┌─────────────┐    ┌───────────────┐
│ App      │───→│ Shared Vol  │───→│ Fluent Bit    │───→ Loki/ES
│ (logs to │    │ (emptyDir)  │    │ (sidecar)     │
│  file)   │    └─────────────┘    └───────────────┘
└──────────┘
```

### Fluent Bit DaemonSet Config

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    spec:
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
```

### kubectl Log Commands

```bash
# Basic logs
kubectl logs pod/storage-api-abc12
kubectl logs pod/storage-api-abc12 -c sidecar    # Specific container

# Previous instance (after crash)
kubectl logs pod/storage-api-abc12 --previous

# Follow (stream)
kubectl logs -f pod/storage-api-abc12

# All pods with label
kubectl logs -l app=storage-api --all-containers

# Last N lines / since duration
kubectl logs pod/storage-api-abc12 --tail=100
kubectl logs pod/storage-api-abc12 --since=1h
```

---

## 4. Distributed Tracing

### OpenTelemetry Collector

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
spec:
  mode: daemonset
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000
      
      resource:
        attributes:
          - key: cluster
            value: production
            action: upsert
    
    exporters:
      otlp:
        endpoint: tempo:4317
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [otlp]
```

### Auto-Instrumentation (Go)

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: go-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
  go:
    image: ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:latest
```

---

## 5. kubectl Debugging

```bash
# Describe pod (events, conditions, volumes)
kubectl describe pod storage-api-abc12

# Events (cluster-wide, sorted by time)
kubectl get events --sort-by=.lastTimestamp -A
kubectl get events -n dell-storage --field-selector=reason=FailedScheduling

# Resource utilization
kubectl top pods -n dell-storage
kubectl top nodes

# Exec into pod
kubectl exec -it pod/storage-api-abc12 -- /bin/sh
kubectl exec pod/storage-api-abc12 -c sidecar -- cat /config/settings.yaml

# Ephemeral debug container (K8s 1.25+ GA)
kubectl debug pod/storage-api-abc12 -it --image=busybox:1.36 --target=app
# Shares PID namespace with target container — can see processes

# Debug node
kubectl debug node/worker-1 -it --image=ubuntu:22.04
# Creates privileged pod, host filesystem at /host

# Port-forward
kubectl port-forward pod/storage-api-abc12 8080:8080
kubectl port-forward svc/prometheus 9090:9090

# Copy files
kubectl cp storage-api-abc12:/tmp/dump.txt ./dump.txt
kubectl cp ./fix.sh storage-api-abc12:/tmp/fix.sh
```

---

## 6. Resource Monitoring

### metrics-server

```bash
# Required for kubectl top and HPA
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods --sort-by=cpu -A
```

### kube-state-metrics

Exposes Kubernetes object state as Prometheus metrics:

```promql
# Deployment replicas
kube_deployment_spec_replicas
kube_deployment_status_replicas_available

# Pod status
kube_pod_status_phase{phase="Running"}
kube_pod_container_status_waiting_reason

# PVC status  
kube_persistentvolumeclaim_status_phase{phase="Bound"}
kube_persistentvolumeclaim_resource_requests_storage_bytes

# Node status
kube_node_status_condition{condition="Ready",status="true"}
```

---

## 7. SLIs/SLOs for Storage Platform

| SLI | Measurement | SLO Target |
|-----|------------|------------|
| **Availability** | Successful API responses / total | 99.99% |
| **Latency** | p99 API response time | < 200ms |
| **Volume Provision** | Successful provisions / attempts | 99.9% |
| **Volume Provision Latency** | Time to provision volume | < 30s |
| **Snapshot Success** | Successful snapshots / attempts | 99.9% |
| **Error Rate** | 5xx responses / total responses | < 0.1% |

```promql
# SLO: 99.9% volume provisioning success
1 - (
  sum(rate(csi_operations_seconds_count{method="CreateVolume",grpc_status!="OK"}[30d]))
  /
  sum(rate(csi_operations_seconds_count{method="CreateVolume"}[30d]))
) > 0.999
```

---

## Interview Questions

1. **How does Prometheus monitoring work in Kubernetes?**
   - Pull-based: Prometheus scrapes /metrics endpoints. ServiceMonitor/PodMonitor CRDs define targets. TSDB stores time-series. PromQL for queries. Alertmanager routes alerts.

2. **How would you monitor a CSI driver?**
   - Expose Prometheus metrics: operation latency, error rates, volume count. ServiceMonitor for scraping. PrometheusRule alerts for failures, capacity warnings. Grafana dashboards for visualization. Log aggregation for debugging.

3. **Explain the kubectl debug command.**
   - Creates ephemeral debug container in a running pod. Shares PID/network namespace with target container. Can also debug nodes by creating a privileged pod with host access. Essential for distroless images without shell.

---

*Next: [Chapter 11 — Cluster Operations](ClusterOperations.md)*
