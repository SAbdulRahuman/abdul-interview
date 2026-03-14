# Namespaces & Cgroups (Container Foundations)

> Linux namespaces for isolation, cgroups v2 for resource limits, and how they combine to create containers.

---

## Namespaces — Isolation

### What Are Namespaces?
Namespaces partition kernel resources so that one set of processes sees one set of resources, and another set sees a different set. They are the **isolation** mechanism behind containers.

### Namespace Types

| Namespace | Isolates | Flag | Since |
|---|---|---|---|
| **PID** | Process IDs (PID 1 inside container) | `CLONE_NEWPID` | 2.6.24 |
| **Network** | Network stack (interfaces, routes, iptables) | `CLONE_NEWNET` | 2.6.29 |
| **Mount** | File system mounts | `CLONE_NEWNS` | 2.4.19 |
| **UTS** | Hostname and domain name | `CLONE_NEWUTS` | 2.6.19 |
| **IPC** | System V IPC, POSIX message queues | `CLONE_NEWIPC` | 2.6.19 |
| **User** | UIDs/GIDs (root inside, non-root outside) | `CLONE_NEWUSER` | 3.8 |
| **Cgroup** | Cgroup root directory | `CLONE_NEWCGROUP` | 4.6 |
| **Time** | System clocks (boottime, monotonic) | `CLONE_NEWTIME` | 5.6 |

### PID Namespace

```
Host PID Namespace:
  PID 1 (systemd)
  PID 100 (dockerd)
  PID 200 (containerd)
  PID 300 ─── Container PID Namespace:
              PID 1 (container init / app)
              PID 2 (child process)
              PID 3 (another process)
```

- Process inside container sees PID 1 (thinks it's init)
- From host, same process has a different PID (e.g., 300)
- PID namespaces are **nested** (child can't see parent's PIDs)
- Sending SIGKILL to container PID 1 (from host) kills all processes in the namespace

```bash
# View process namespace
ls -la /proc/$$/ns/
# lrwxrwxrwx 1 root root 0 pid -> 'pid:[4026531836]'
# lrwxrwxrwx 1 root root 0 net -> 'net:[4026531840]'
# lrwxrwxrwx 1 root root 0 mnt -> 'mnt:[4026531841]'
# ...

# Check if two processes share a namespace
readlink /proc/1/ns/pid
readlink /proc/$$/ns/pid
# Same number = same namespace
```

### Mount Namespace

```
Host Mount Namespace:
  /           → host root filesystem
  /home       → host home
  /var        → host var

Container Mount Namespace:
  /           → container rootfs (e.g., overlay)
  /etc        → container /etc
  /proc       → container proc (procfs mounted)
  /dev        → minimal devtmpfs
  /data       → bind mount from host:/var/data

Each namespace has its own mount table (cat /proc/<pid>/mounts)
```

### User Namespace

```
Outside (host):
  UID 1000 (regular user) runs container

Inside (container):
  UID 0 (root) — but mapped to UID 1000 on host
  
  Mapping (/proc/<pid>/uid_map):
  Inside  Outside  Count
  0       1000     1      → container root = host UID 1000
  1       100000   65536  → other UIDs mapped to high-range UIDs
```

- Enables **rootless containers** (unprivileged user runs container with "root" inside)
- Security: container "root" has no special privileges on host
- Used by: Podman (rootless), Docker rootless mode

### Network Namespace
(Covered in detail in [NetworkProgramming.md](NetworkProgramming.md))

- Each namespace has its own interfaces, IPs, routes, iptables, sockets
- Connected via veth pairs to other namespaces or bridges

---

## Creating Namespaces

### unshare() — Create Namespace for Current Process

```c
#include <sched.h>
#include <unistd.h>

// Create new PID + mount namespace
unshare(CLONE_NEWPID | CLONE_NEWNS);

// After fork, child is in new PID namespace
pid_t pid = fork();
if (pid == 0) {
    // Child: PID 1 in new namespace
    // Mount new proc
    mount("proc", "/proc", "proc", 0, NULL);
    execl("/bin/bash", "bash", NULL);
}
```

```bash
# unshare command-line tool
# New PID + mount + UTS namespace
sudo unshare --pid --mount --uts --fork bash

# Inside new namespace:
hostname container-test
hostname  # Shows container-test (only visible inside namespace)

# New network namespace
sudo unshare --net bash
ip addr  # Only loopback visible
```

### clone() — Create Child in New Namespace

```c
#include <sched.h>

// clone() with namespace flags
int flags = CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | CLONE_NEWUTS;
pid_t pid = clone(child_func, child_stack + STACK_SIZE, flags | SIGCHLD, arg);
```

### setns() — Join Existing Namespace

```c
#include <sched.h>
#include <fcntl.h>

// Join another process's namespace
int fd = open("/proc/<pid>/ns/net", O_RDONLY);
setns(fd, CLONE_NEWNET);
close(fd);
// Now in the same network namespace as <pid>
```

```bash
# nsenter — enter namespaces of a running process/container
nsenter -t <pid> --net --pid --mount bash
# Enters the PID, network, and mount namespaces of process <pid>

# Docker equivalent:
docker exec -it <container> bash
# Internally uses setns() to join container namespaces
```

---

## Cgroups — Resource Limits

### What Are Cgroups?
Control groups limit, account for, and isolate **resource usage** (CPU, memory, I/O, PIDs) of process groups.

### Cgroups v1 vs v2

| Feature | Cgroups v1 | Cgroups v2 |
|---|---|---|
| Hierarchy | Multiple hierarchies (one per controller) | Unified hierarchy |
| Controllers | Independent, can differ between hierarchies | Single tree |
| Thread support | Limited | Full (thread-level control) |
| Delegation | Complex | Clean delegation model |
| Interface | Multiple mount points | Single mount at /sys/fs/cgroup |
| Status | Legacy (still works) | Default since Linux 5.6+ |

### Cgroups v2 Hierarchy

```
/sys/fs/cgroup/                          ← Root cgroup
├── cgroup.controllers                   ← Available controllers
├── cgroup.subtree_control               ← Active controllers for children
├── system.slice/                        ← systemd system services
│   ├── docker.service/                  ← Docker daemon
│   └── sshd.service/                    ← SSH daemon
├── user.slice/                          ← User sessions
│   └── user-1000.slice/
└── docker/                              ← Container cgroups
    ├── <container-id-1>/
    │   ├── cgroup.procs                 ← PIDs in this cgroup
    │   ├── cpu.max                      ← CPU limit
    │   ├── memory.max                   ← Memory limit
    │   └── io.max                       ← I/O limit
    └── <container-id-2>/
```

---

## Cgroup Controllers

### CPU Controller

```bash
# CPU limit: 50% of one core
echo "50000 100000" > /sys/fs/cgroup/mygroup/cpu.max
# Format: $MAX $PERIOD (microseconds)
# 50000/100000 = 50% of one CPU
# 200000/100000 = 200% = 2 full CPUs

# CPU weight (relative sharing, like nice)
echo 100 > /sys/fs/cgroup/mygroup/cpu.weight
# Default: 100, range: 1-10000
# Higher weight = more CPU time relative to other groups

# Read CPU statistics
cat /sys/fs/cgroup/mygroup/cpu.stat
# usage_usec: total CPU time used
# user_usec: user-space CPU time
# system_usec: kernel CPU time
# nr_periods: number of enforcement periods
# nr_throttled: times this cgroup was throttled
# throttled_usec: total throttle time
```

### Memory Controller

```bash
# Set memory limit (500MB)
echo 500000000 > /sys/fs/cgroup/mygroup/memory.max
# Or with suffix: echo 500M > memory.max

# Soft limit (try to reclaim when exceeded)
echo 400000000 > /sys/fs/cgroup/mygroup/memory.high

# Minimum guaranteed memory (protect from reclaim)
echo 100000000 > /sys/fs/cgroup/mygroup/memory.min

# Swap limit
echo 0 > /sys/fs/cgroup/mygroup/memory.swap.max  # Disable swap

# Current usage
cat /sys/fs/cgroup/mygroup/memory.current        # RSS + cache
cat /sys/fs/cgroup/mygroup/memory.stat
# anon: anonymous memory (heap, stack)
# file: page cache
# slab: kernel slab
# sock: socket buffers

# OOM events
cat /sys/fs/cgroup/mygroup/memory.events
# oom: number of OOM events
# oom_kill: number of OOM kills
```

### I/O Controller

```bash
# I/O limits (per device)
# Format: $MAJOR:$MINOR rbps=$READ_BPS wbps=$WRITE_BPS riops=$READ_IOPS wiops=$WRITE_IOPS

# Limit to 10MB/s read, 5MB/s write on device 8:0 (/dev/sda)
echo "8:0 rbps=10485760 wbps=5242880" > /sys/fs/cgroup/mygroup/io.max

# Limit IOPS
echo "8:0 riops=1000 wiops=500" > /sys/fs/cgroup/mygroup/io.max

# I/O weight (relative sharing)
echo "8:0 100" > /sys/fs/cgroup/mygroup/io.weight

# Read I/O statistics
cat /sys/fs/cgroup/mygroup/io.stat
# 8:0 rbytes=1048576 wbytes=524288 rios=256 wios=128 dbytes=0 dios=0

# Find device major:minor
ls -la /dev/sda    # Shows major:minor
lsblk -o NAME,MAJ:MIN
```

### PID Controller

```bash
# Limit max processes
echo 100 > /sys/fs/cgroup/mygroup/pids.max

# Current count
cat /sys/fs/cgroup/mygroup/pids.current

# Prevents fork bomb attacks within a cgroup
```

### Cpuset Controller

```bash
# Pin to specific CPUs
echo "0-3" > /sys/fs/cgroup/mygroup/cpuset.cpus    # CPUs 0-3 only

# Pin to specific NUMA memory nodes
echo "0" > /sys/fs/cgroup/mygroup/cpuset.mems       # Node 0 only

# Useful for NUMA-aware storage applications
```

---

## Working with Cgroups

### Creating and Managing Cgroups

```bash
# Create a cgroup
mkdir /sys/fs/cgroup/mygroup

# Enable controllers for children
echo "+cpu +memory +io +pids" > /sys/fs/cgroup/cgroup.subtree_control

# Assign a process to cgroup
echo $PID > /sys/fs/cgroup/mygroup/cgroup.procs

# List processes in cgroup
cat /sys/fs/cgroup/mygroup/cgroup.procs

# Remove cgroup (must be empty — no processes, no children)
rmdir /sys/fs/cgroup/mygroup

# View cgroup of a process
cat /proc/<pid>/cgroup
# 0::/docker/<container-id>

# systemd cgroup management
systemd-cgls                          # Tree view of all cgroups
systemd-cgtop                         # Top-like view of cgroup resource usage
```

### Using systemd for Cgroup Control

```bash
# Run a command with resource limits via systemd-run
systemd-run --scope -p MemoryMax=500M -p CPUQuota=50% ./myapp

# Set limits on existing service
systemctl set-property docker.service MemoryMax=2G CPUQuota=200%

# View service cgroup
systemctl show docker.service -p ControlGroup
```

---

## How Namespaces + Cgroups = Containers

### Container = Namespaces + Cgroups + rootfs

```
┌─────────────────────────────────────────────────────────────┐
│  Container Construction                                      │
│                                                             │
│  1. ISOLATION (Namespaces):                                  │
│     ├── PID namespace    → own PID 1                        │
│     ├── Network namespace → own interfaces, IPs, routes     │
│     ├── Mount namespace  → own filesystem view              │
│     ├── UTS namespace    → own hostname                     │
│     ├── IPC namespace    → own shared memory, semaphores    │
│     └── User namespace   → own UID/GID mapping              │
│                                                             │
│  2. RESOURCE LIMITS (Cgroups):                               │
│     ├── CPU: cpu.max (e.g., 200% = 2 cores)                │
│     ├── Memory: memory.max (e.g., 512MB)                    │
│     ├── I/O: io.max (bandwidth/IOPS per device)            │
│     └── PIDs: pids.max (e.g., 100)                          │
│                                                             │
│  3. FILESYSTEM (rootfs):                                     │
│     ├── Container image (layers via OverlayFS)              │
│     ├── pivot_root() to new root                            │
│     └── Mount proc, sys, dev                                │
│                                                             │
│  4. SECURITY:                                                │
│     ├── Capabilities (drop unnecessary root powers)         │
│     ├── Seccomp (syscall filtering)                         │
│     ├── AppArmor / SELinux (MAC)                            │
│     └── Read-only rootfs                                    │
└─────────────────────────────────────────────────────────────┘
```

### Minimal Container in Shell

```bash
#!/bin/bash
# Create a minimal container with unshare

# Prepare rootfs (e.g., from busybox)
mkdir -p /tmp/container/rootfs
# (populate rootfs with busybox or debootstrap)

# Create container with namespaces
unshare --pid --mount --net --uts --ipc --fork \
  chroot /tmp/container/rootfs /bin/sh -c '
    mount -t proc proc /proc
    mount -t sysfs sys /sys
    hostname container
    exec /bin/sh
'
```

### Container Runtime Flow

```
docker run nginx
    │
    ├── 1. Pull image (layers)
    ├── 2. Create OverlayFS (merge layers)
    ├── 3. Create cgroup (set limits)
    ├── 4. Create namespaces (PID, net, mount, uts, ipc)
    ├── 5. Configure networking (veth pair + bridge)
    ├── 6. Set up rootfs (pivot_root)
    ├── 7. Drop capabilities
    ├── 8. Apply seccomp profile
    └── 9. exec() the entrypoint (/docker-entrypoint.sh)
```

---

## Inspecting Container Namespaces and Cgroups

```bash
# Find container process PID
docker inspect --format '{{.State.Pid}}' <container>

# View namespaces
ls -la /proc/<pid>/ns/

# Enter container namespaces
nsenter -t <pid> -p -n -m bash

# View container cgroup
cat /proc/<pid>/cgroup
# 0::/docker/<container-id>

# View container resource usage
cat /sys/fs/cgroup/docker/<container-id>/memory.current
cat /sys/fs/cgroup/docker/<container-id>/cpu.stat
cat /sys/fs/cgroup/docker/<container-id>/pids.current

# Docker stats (uses cgroup data)
docker stats --no-stream
```

---

## Interview Tips

1. **What are namespaces and which types exist?**
   → Kernel-level isolation. 8 types: PID, Network, Mount, UTS, IPC, User, Cgroup, Time. Each gives a process its own view of that resource.

2. **What are cgroups and what do they control?**
   → Resource limits for process groups. CPU (time, shares), memory (max, swap), I/O (bandwidth, IOPS), PIDs (max processes).

3. **How do namespaces + cgroups = containers?**
   → Namespaces provide isolation (own PID 1, network, filesystem). Cgroups provide limits. Add rootfs (OverlayFS), security (capabilities, seccomp) → container.

4. **Cgroups v1 vs v2?**
   → v1: multiple hierarchies per controller, complex. v2: unified hierarchy, cleaner delegation, better resource distribution. v2 is default since Linux 5.6+.

5. **What is a user namespace?**
   → Maps UIDs: root (UID 0) inside container maps to unprivileged UID on host. Enables rootless containers. Security boundary.

6. **How to debug a container's resource usage?**
   → `docker stats`, or directly read cgroup files: memory.current, cpu.stat, io.stat. Find PID with `docker inspect`, check /proc/<pid>/cgroup.

---

*Last updated: March 13, 2026*
