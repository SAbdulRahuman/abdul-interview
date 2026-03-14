# Chapter 14 — Troubleshooting

## 1. Pod Not Starting — Diagnostic Flowchart

```
Pod status? ──→ kubectl get pod <name> -o wide
        │
   ┌────┴──────────────────────────┐
   │                               │
Pending                         Not Running
   │                               │
   ├── ImagePullBackOff     ┌─────┴──────┐
   ├── Pending (no events)  │            │
   ├── Unschedulable        CrashLoop    Error
   └── PVC Pending          BackOff      
                            │
                            └── Check logs
                                kubectl logs --previous
```

---

## 2. Pending Pods

```bash
# Step 1: Describe the pod
kubectl describe pod <name>

# Look in Events section:
# FailedScheduling: Insufficient CPU/memory
# FailedScheduling: No nodes match nodeSelector
# FailedScheduling: Taint not tolerated
# FailedScheduling: PVC not bound
```

### Common Causes & Fixes

| Symptom | Event Message | Fix |
|---------|--------------|-----|
| Insufficient resources | `Insufficient cpu/memory` | Scale cluster, adjust requests, delete unused pods |
| No matching node | `node(s) didn't match Pod's node affinity/selector` | Fix nodeSelector/affinity, add labels to nodes |
| Taints | `node(s) had untolerated taint` | Add toleration or remove taint |
| PVC not bound | `persistentvolumeclaim "X" not found` or `unbound` | Check StorageClass, CSI driver, PVC spec |
| Too many pods | `Too many pods` | Increase --max-pods on kubelet |
| Topology spread | `doesn't satisfy spread constraint` | Add nodes in required topology |

```bash
# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check available resources
kubectl top nodes

# Check PVC status
kubectl get pvc -o wide
kubectl describe pvc <name>

# Check StorageClass
kubectl get sc
kubectl describe sc <name>

# Check CSI driver
kubectl get csidrivers
kubectl get csinodes
```

---

## 3. ImagePullBackOff

```bash
# Events show:
# Failed to pull image "registry.dell.com/storage:v2":
#   rpc error: code = Unknown desc = Error response from daemon:
#   pull access denied, repository does not exist or may require authentication

# Debugging steps:
# 1. Verify image exists
docker pull registry.dell.com/storage:v2

# 2. Check imagePullSecrets
kubectl get pod <name> -o jsonpath='{.spec.imagePullSecrets}'

# 3. Verify secret has correct credentials
kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# 4. Check image name for typos
kubectl get pod <name> -o jsonpath='{.spec.containers[*].image}'

# Common fixes:
# - Create/fix imagePullSecrets
# - Use correct image tag
# - Check network access to registry
# - Verify registry credentials haven't expired
```

---

## 4. CrashLoopBackOff

Container starts, crashes, restarts with exponential backoff (10s → 20s → 40s → ... → 5min max).

```bash
# Step 1: Check logs
kubectl logs <pod> --previous    # Logs from crashed container
kubectl logs <pod> -c <container>  # Specific container

# Step 2: Describe pod
kubectl describe pod <pod>
# Look for: OOMKilled, Exit Code, Last State

# Step 3: Common exit codes
# Exit 0:  Success (shouldn't crash loop — check restartPolicy)
# Exit 1:  Application error (check logs)
# Exit 2:  Shell/command not found
# Exit 126: Permission denied (binary not executable)
# Exit 127: Command not found (wrong image or entrypoint)
# Exit 128: Invalid exit argument
# Exit 137: SIGKILL (OOMKilled or killed by system)
# Exit 139: SIGSEGV (segfault — memory corruption)
# Exit 143: SIGTERM (graceful shutdown — normal during restart)

# Step 4: OOMKilled check
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# "OOMKilled" → increase memory limits

# Step 5: Debug with ephemeral container
kubectl debug <pod> -it --image=busybox:1.36 --target=<container>

# Step 6: Override entrypoint to investigate
kubectl run debug-pod --image=<same-image> \
  --restart=Never --command -- sleep infinity
kubectl exec -it debug-pod -- /bin/sh
```

---

## 5. Service Not Reachable

```bash
# Step 1: Verify service exists and has endpoints
kubectl get svc <service>
kubectl get endpoints <service>
kubectl get endpointslices -l kubernetes.io/service-name=<service>

# If no endpoints:
# - Check selector matches pod labels
# - Check pod readiness probes (failing = no endpoint)
# - Check pod is Running

# Step 2: DNS resolution
kubectl run dns-test --rm -it --image=busybox:1.36 -- nslookup <service>
kubectl run dns-test --rm -it --image=busybox:1.36 -- nslookup <service>.<namespace>.svc.cluster.local

# Step 3: Direct pod connectivity
kubectl exec -it <client-pod> -- curl http://<pod-ip>:<port>
kubectl exec -it <client-pod> -- curl http://<service>:<port>

# Step 4: Check NetworkPolicy
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name>
# Network policies are additive — whitelist model

# Step 5: Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50
# Check iptables rules on node:
# iptables -t nat -L KUBE-SERVICES | grep <service-ip>

# Step 6: Service type issues
# NodePort: firewall blocking port range 30000-32767?
# LoadBalancer: cloud LB provisioned? Check events
# ExternalName: DNS CNAME resolving?
```

---

## 6. Node Issues

```bash
# Check node status
kubectl get nodes
kubectl describe node <node>

# Common conditions
# Ready=False: kubelet not healthy
# MemoryPressure=True: low memory → pod eviction
# DiskPressure=True: low disk → pod eviction
# PIDPressure=True: too many processes
# NetworkUnavailable=True: CNI plugin issue

# Node debugging
kubectl debug node/<node> -it --image=ubuntu:22.04
# Inside debug container:
chroot /host
journalctl -u kubelet --since "30 minutes ago"
systemctl status kubelet
systemctl status containerd
df -h                    # Disk space
free -h                  # Memory
top                      # CPU/process
dmesg | tail -50         # Kernel messages
```

### Node NotReady

```bash
# Common causes:
# 1. kubelet crashed
ssh <node> "systemctl status kubelet"
ssh <node> "journalctl -u kubelet -n 100"

# 2. Container runtime down
ssh <node> "systemctl status containerd"
ssh <node> "crictl ps"

# 3. Network issue (node can't reach API server)
ssh <node> "curl -k https://<apiserver>:6443/healthz"

# 4. Certificate expired
ssh <node> "openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates"

# 5. Disk full
ssh <node> "df -h"
# Fix: clean images: crictl rmi --prune
```

---

## 7. Storage Issues

```bash
# PVC stuck in Pending
kubectl describe pvc <name>
# Events:
# "waiting for a volume to be created" → CSI driver issue
# "no persistent volumes available" → no matching PV
# "storageclass not found" → wrong SC name

# Volume mount errors
kubectl describe pod <name>
# Events:
# "FailedMount: MountVolume.SetUp failed" → CSI node issue
# "FailedAttachVolume" → CSI controller issue / storage backend down

# CSI driver logs
kubectl logs -n dell-storage -l app=csi-powerstore -c driver --tail=100
kubectl logs -n dell-storage -l app=csi-powerstore-node -c driver --tail=100

# VolumeAttachment status
kubectl get volumeattachments
kubectl describe volumeattachment <name>

# Check CSI node info
kubectl get csinodes <node-name> -o yaml

# iSCSI connectivity (on node)
kubectl debug node/<node> -it --image=ubuntu:22.04
chroot /host
iscsiadm -m session      # Active iSCSI sessions
iscsiadm -m node          # Discovered targets
multipath -ll              # Multipath status
lsblk                     # Block devices
mount | grep kubernetes    # K8s volume mounts
```

---

## 8. kubectl debug

### Ephemeral Debug Containers

```bash
# Debug running pod (add container with debugging tools)
kubectl debug <pod> -it \
  --image=nicolaka/netshoot \
  --target=<container-name>
# Shares PID namespace → can see processes in target container
# Process list: ps aux
# Network: netstat -tlnp, ss -tlnp
# DNS: nslookup, dig
# HTTP: curl, wget

# Debug with different image (copy pod, override image)
kubectl debug <pod> -it \
  --copy-to=debug-pod \
  --image=ubuntu:22.04 \
  --share-processes

# Debug with root (copy pod, override security context)
kubectl debug <pod> -it \
  --copy-to=debug-pod \
  --image=busybox:1.36 \
  --set-image=*=busybox:1.36   # Replace all container images
```

### Node Debugging

```bash
# Debug node (creates privileged pod)
kubectl debug node/worker-1 -it --image=ubuntu:22.04
# Pod runs with hostPID, hostNetwork, and mounts host root at /host

# Inside:
chroot /host                    # Enter host filesystem
journalctl -u kubelet -f        # Live kubelet logs
crictl ps                       # List containers
crictl logs <container-id>      # Container logs
nsenter -t 1 -m -u -i -n       # Enter init namespace
iptables -t nat -L KUBE-SERVICES | head
```

---

## 9. Networking Debugging

```bash
# DNS test
kubectl run dns-debug --rm -it --image=nicolaka/netshoot -- bash
nslookup kubernetes.default.svc.cluster.local
dig storage-api.dell-storage.svc.cluster.local

# Connectivity test
kubectl run net-debug --rm -it --image=nicolaka/netshoot -- bash
curl -v http://storage-api.dell-storage:8080/healthz
traceroute <pod-ip>
ping <node-ip>

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Network policy testing
# Temporarily label a pod to test network policy
kubectl label pod test-pod role=frontend
# Try connecting from labeled pod
```

---

## 10. Resource Exhaustion

### CPU Throttling

```bash
# Detect CPU throttling
kubectl top pod <pod>

# Prometheus query:
# rate(container_cpu_cfs_throttled_periods_total[5m]) 
# / rate(container_cpu_cfs_periods_total[5m]) > 0.25
# → 25%+ of time slots throttled

# Fix: increase CPU limit or remove limit (keep request)
```

### OOMKilled

```bash
# Check for OOM events
kubectl get events --field-selector=reason=OOMKilling -A
kubectl describe pod <pod> | grep -A3 "Last State"

# Prometheus query for pod memory vs limit:
# container_memory_working_set_bytes / kube_pod_container_resource_limits{resource="memory"}

# Fix: increase memory limit, or fix memory leak
# Profile: use pprof if Go app
```

---

## 11. Common Troubleshooting Checklist

```
□ kubectl get pods -o wide          → Pod status, node, IP
□ kubectl describe pod <name>       → Events, conditions
□ kubectl logs <pod> [--previous]   → Application logs
□ kubectl get events --sort-by=.lastTimestamp
□ kubectl top pod/node              → Resource utilization
□ kubectl get endpoints <svc>       → Service backend pods
□ kubectl get pvc                   → Volume status
□ kubectl get nodes                 → Node health
□ kubectl debug                     → Interactive debugging
```

---

## Interview Questions

1. **Pod stuck in Pending — how do you debug?**
   - `kubectl describe pod` → check Events. Common: insufficient resources (scale cluster or reduce requests), node selector/affinity mismatch, untolerated taint, unbound PVC. Check node allocatable resources.

2. **CrashLoopBackOff — systematic debugging approach?**
   - Check `kubectl logs --previous`. Look at exit code (137=OOM, 1=app error, 127=cmd not found). Check `kubectl describe pod` for OOMKilled. Use ephemeral debug containers. Override entrypoint with sleep to investigate filesystem/config.

3. **How do you debug storage issues (PVC not binding)?**
   - Check StorageClass exists and CSI driver is running. `kubectl describe pvc` for events. Check CSI controller logs. Verify VolumeAttachment objects. On node: check iSCSI sessions, multipath, device presence. Check topology constraints.

4. **A service is unreachable — troubleshooting steps?**
   - Verify service exists and selector matches pod labels. Check endpoints (empty = no ready pods). Test DNS resolution. Test direct pod connectivity. Check NetworkPolicies. Verify kube-proxy is running. Check for firewall rules.

5. **How do you debug a node in NotReady state?**
   - `kubectl debug node` or SSH. Check kubelet status/logs (`journalctl -u kubelet`). Check container runtime. Verify network to API server. Check certificates. Check disk space and memory. Check dmesg for kernel issues.

---

*Next: [Chapter 15 — Interview Questions](InterviewQuestions.md)*
