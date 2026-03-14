# Performance Tuning

> Kernel parameter tuning for memory, network, file descriptors, storage, and NUMA optimization.

---

## Memory Tuning

### vm.swappiness

```bash
# Controls tendency to swap out application memory vs page cache
# Range: 0-200 (default: 60)
# Lower = prefer reclaiming page cache, avoid swapping app memory
# Higher = more willing to swap app memory

sysctl vm.swappiness              # Check current
sysctl -w vm.swappiness=10        # Set to 10 (recommended for servers)

# Server recommendation: 10 or lower
# Storage/database servers: 1 (avoid swapping at almost all costs)
# Never set to 0: may cause OOM kills even with swap available (before kernel 3.5)
```

### Dirty Page Writeback

```bash
# Background writeback: starts async writeback when dirty pages exceed this %
sysctl vm.dirty_background_ratio=5     # Default: 10 (% of total RAM)
# Or absolute bytes:
sysctl vm.dirty_background_bytes=104857600   # 100MB

# Sync writeback: blocking writes when dirty pages exceed this %
sysctl vm.dirty_ratio=20                # Default: 20 (% of total RAM)
# Or absolute bytes:
sysctl vm.dirty_bytes=209715200         # 200MB

# How long dirty pages can stay before writeback
sysctl vm.dirty_expire_centisecs=3000   # Default: 3000 (30 seconds)

# How often writeback thread runs
sysctl vm.dirty_writeback_centisecs=500 # Default: 500 (5 seconds)
```

#### Tuning for Workload Type

| Workload | dirty_background_ratio | dirty_ratio | Why |
|---|---|---|---|
| **General server** | 5 | 20 | Balanced |
| **Database** | 3 | 10 | Limit dirty page buildup, predictable latency |
| **Streaming write** | 10 | 40 | Allow more buffering for throughput |
| **Real-time** | 1 | 5 | Minimize writeback spikes |

### Overcommit

```bash
# vm.overcommit_memory:
# 0 (default): Heuristic — allow "reasonable" overcommit
# 1: Always allow (malloc never fails; may OOM later)
# 2: Strict — commit limit = swap + RAM × overcommit_ratio

sysctl vm.overcommit_memory=0
sysctl vm.overcommit_ratio=50    # Only for mode=2 (default: 50%)

# Check commit status
grep -i commit /proc/meminfo
# CommitLimit:    20000000 kB    ← max allowed (mode=2)
# Committed_AS:  15000000 kB    ← currently committed
```

### Other Memory Parameters

```bash
# Minimum free memory (emergency reserve, avoids OOM under sudden pressure)
sysctl vm.min_free_kbytes=262144    # 256MB (default: varies)
# Too low: OOM under memory pressure
# Too high: wastes memory

# Zone reclaim mode (NUMA)
sysctl vm.zone_reclaim_mode=0       # 0 = allocate from remote node first (default)
                                     # 1 = reclaim local cache first

# VFS cache pressure (inode/dentry cache)
sysctl vm.vfs_cache_pressure=100    # Default: 100
# < 100: keep more inodes/dentries in cache (good for many files)
# > 100: reclaim inode/dentry cache more aggressively
```

---

## Network Tuning

### Connection Backlog

```bash
# Maximum listen() backlog
sysctl net.core.somaxconn=65535            # Default: 4096 (was 128)
# Affects: listen(fd, backlog) — actual backlog = min(somaxconn, backlog)

# SYN queue size (half-open connections)
sysctl net.ipv4.tcp_max_syn_backlog=65535  # Default: 1024

# SYN cookies (protect against SYN flood)
sysctl net.ipv4.tcp_syncookies=1           # Default: 1 (enabled)
```

### TCP Buffer Sizes

```bash
# Maximum buffer sizes (cap for auto-tuning)
sysctl net.core.rmem_max=16777216          # 16MB receive
sysctl net.core.wmem_max=16777216          # 16MB send

# TCP auto-tuning buffer sizes: min, default, max
sysctl net.ipv4.tcp_rmem="4096 65536 16777216"   # Receive
sysctl net.ipv4.tcp_wmem="4096 65536 16777216"   # Send
# Default: "4096 87380 6291456"

# When to increase:
# - High bandwidth-delay product (BDP) links (WAN, data replication)
# - Formula: BDP = bandwidth (bytes/s) × RTT (seconds)
# - Example: 10Gbps, 10ms RTT → BDP = 1.25GB/s × 0.01s = 12.5MB
# - Buffer must be >= BDP for full throughput
```

### Connection Reuse

```bash
# Reuse TIME_WAIT sockets (safe for outbound connections)
sysctl net.ipv4.tcp_tw_reuse=1             # Default: 0

# Ephemeral port range (increase if making many outbound connections)
sysctl net.ipv4.ip_local_port_range="1024 65535"   # Default: "32768 60999"
# Available ports: 65535 - 1024 = 64511 (vs default 28231)
```

### Keepalive

```bash
# When to start sending keepalive probes (seconds of idle)
sysctl net.ipv4.tcp_keepalive_time=60      # Default: 7200 (2 hours!)
# Interval between probes
sysctl net.ipv4.tcp_keepalive_intvl=10     # Default: 75
# Number of probes before declaring dead
sysctl net.ipv4.tcp_keepalive_probes=5     # Default: 9

# Total detection time: 60 + 10×5 = 110 seconds (vs default 7200 + 75×9 = ~2.5 hours)
```

### Other Network Parameters

```bash
# Enable TCP Fast Open (TFO — send data in SYN)
sysctl net.ipv4.tcp_fastopen=3             # 1=client, 2=server, 3=both

# Congestion control algorithm
sysctl net.ipv4.tcp_congestion_control=bbr  # Google BBR (better for WAN)
# Options: cubic (default), bbr, reno, vegas

# Enable BBR
modprobe tcp_bbr
sysctl net.ipv4.tcp_congestion_control=bbr
sysctl net.core.default_qdisc=fq           # Fair queueing (recommended with BBR)

# Disable slow start after idle (better for persistent connections)
sysctl net.ipv4.tcp_slow_start_after_idle=0

# Window scaling (for high BDP)
sysctl net.ipv4.tcp_window_scaling=1       # Default: 1 (enabled)
```

---

## File Descriptor Limits

### System-Wide Limits

```bash
# Maximum file descriptors system-wide
sysctl fs.file-max=1000000                 # Default: varies
cat /proc/sys/fs/file-nr
# 5000    0    1000000
# allocated  unused  maximum

# Maximum per-process
sysctl fs.nr_open=1000000                  # Default: 1048576
```

### Per-Process Limits (ulimit / pam)

```bash
# Check current limits
ulimit -n                                  # Soft limit (open files)
ulimit -Hn                                 # Hard limit

# Set for current shell
ulimit -n 65535

# Persistent per-user (requires re-login)
# /etc/security/limits.conf:
# *    soft    nofile    65535
# *    hard    nofile    65535
# root soft    nofile    65535
# root hard    nofile    65535

# For systemd services:
# /etc/systemd/system/myapp.service
# [Service]
# LimitNOFILE=65535

# Check process limits
cat /proc/<pid>/limits | grep "open files"
```

### inotify Limits (File Watchers)

```bash
# Maximum inotify watches per user (for file monitoring tools)
sysctl fs.inotify.max_user_watches=524288   # Default: 8192
sysctl fs.inotify.max_user_instances=1024   # Default: 128

# Running out causes "Too many open files" or "No space left on device"
# Needed by: IDEs, build tools, container runtimes
```

---

## Storage Tuning

### I/O Scheduler

```bash
# View current scheduler
cat /sys/block/sda/queue/scheduler
cat /sys/block/nvme0n1/queue/scheduler

# Change scheduler
echo mq-deadline > /sys/block/sda/queue/scheduler     # HDDs
echo none > /sys/block/nvme0n1/queue/scheduler         # NVMe (skip scheduling)

# Persistent via udev
# /etc/udev/rules.d/60-scheduler.rules
ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/scheduler}="mq-deadline"
```

### Queue Depth and Readahead

```bash
# Queue depth (number of I/O requests device can queue)
cat /sys/block/sda/queue/nr_requests     # Default: 64-256
echo 256 > /sys/block/sda/queue/nr_requests

# Readahead (prefetch for sequential access)
blockdev --getra /dev/sda                # Get (in 512-byte sectors)
blockdev --setra 8192 /dev/sda           # Set 4MB readahead (8192 × 512B)

# Good readahead values:
# Sequential workload:  8192-32768 (4MB-16MB)
# Random workload:      256 (128KB) or less
# Database:             256-2048 (depends on access pattern)
```

### NVMe-Specific

```bash
# NVMe queue count and depth
cat /sys/block/nvme0n1/queue/nr_hw_queues     # Hardware queues (1 per CPU ideal)
cat /sys/block/nvme0n1/queue/nr_requests      # Queue depth per queue

# NVMe power state management (disable for latency-sensitive)
nvme get-feature /dev/nvme0 -f 0x02           # Power management feature
# Or via kernel: nvme_core.force_apst=0       # Disable autonomous power state
```

### Filesystem Mount Options

```bash
# ext4 performance options
mount -o noatime,data=ordered,barrier=1,discard /dev/sda1 /data

# XFS performance options
mount -o noatime,logbsize=256k,allocsize=64k /dev/sda1 /data

# Key options:
# noatime       — don't update access time (reduces writes)
# relatime      — update atime only if older than mtime (default, less write)
# discard       — TRIM support for SSDs (or use fstrim cron)
# barrier=1     — enable write barriers (default, important for data integrity)
# data=ordered  — ext4: write data before metadata journal (default)
```

---

## NUMA Optimization

### Understanding NUMA Topology

```bash
# View NUMA topology
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 16384 MB
# node 1 cpus: 8 9 10 11 12 13 14 15
# node 1 size: 16384 MB
# node distances:
# node   0   1
#   0:  10  21    ← 2.1x latency for remote access
#   1:  21  10

lscpu | grep NUMA
lstopo                           # Graphical topology (hwloc)
```

### NUMA-Aware Execution

```bash
# Pin process to NUMA node 0 (CPU + memory)
numactl --cpunodebind=0 --membind=0 ./myapp

# Interleave memory across all nodes (good for initialization)
numactl --interleave=all ./myapp

# Prefer local allocation but allow remote
numactl --preferred=0 ./myapp

# Specific CPUs
numactl --physcpubind=0-7 ./myapp
taskset -c 0-7 ./myapp            # Same effect
```

### NUMA Monitoring

```bash
# System-wide NUMA stats
numastat
# Per-process NUMA stats
numastat -p <pid>
# Look for:
# numa_miss:  allocations from non-preferred node (bad)
# other_node: memory allocated on remote node (bad for latency)
# local_node: memory allocated locally (good)

# Per-node memory info
cat /sys/devices/system/node/node0/meminfo
```

### NUMA Best Practices for Storage

1. **Bind storage workers to same NUMA node as NVMe device**
   - Check NVMe NUMA node: `cat /sys/block/nvme0n1/device/numa_node`
   - Bind: `numactl --cpunodebind=<node> --membind=<node> ./storage-app`

2. **Avoid cross-node memory access in hot paths**
   - Allocate buffers on the same NUMA node as the processing thread
   - Use `mbind()` or `set_mempolicy()` for fine-grained control

3. **Monitor NUMA misses in production**
   - `perf stat -e node-load-misses,node-store-misses ./myapp`

---

## Applying Changes

### Temporary (Until Reboot)

```bash
sysctl -w vm.swappiness=10
sysctl -w net.core.somaxconn=65535
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### Persistent (Survives Reboot)

```bash
# Create sysctl config file
cat > /etc/sysctl.d/99-performance.conf << 'EOF'
# Memory
vm.swappiness = 10
vm.dirty_background_ratio = 5
vm.dirty_ratio = 20
vm.min_free_kbytes = 262144

# Network
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 5

# File descriptors
fs.file-max = 1000000
fs.inotify.max_user_watches = 524288
EOF

# Apply
sysctl -p /etc/sysctl.d/99-performance.conf

# Verify
sysctl vm.swappiness net.core.somaxconn
```

### systemd Service Tuning

```ini
# /etc/systemd/system/myapp.service
[Service]
LimitNOFILE=65535
LimitNPROC=65535
LimitCORE=infinity
MemoryMax=4G
CPUQuota=200%

# For NUMA pinning:
ExecStart=/usr/bin/numactl --cpunodebind=0 --membind=0 /usr/local/bin/myapp
```

---

## Tuning Checklist for Storage Servers

```bash
# 1. CPU
# - Set CPU governor to performance
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
# - Disable CPU idle states for latency-sensitive
# - NUMA: bind to NVMe device's NUMA node

# 2. Memory
sysctl vm.swappiness=1
sysctl vm.dirty_background_ratio=3
sysctl vm.dirty_ratio=10
sysctl vm.min_free_kbytes=262144
# - Use huge pages if applicable
# - Disable THP for databases (defrag latency spikes)

# 3. Network
sysctl net.core.somaxconn=65535
sysctl net.ipv4.tcp_tw_reuse=1
sysctl net.ipv4.tcp_keepalive_time=60
# - Increase buffer sizes for high-BDP links
# - Enable BBR for WAN replication

# 4. Storage
echo none > /sys/block/nvme0n1/queue/scheduler   # NVMe
echo mq-deadline > /sys/block/sda/queue/scheduler # HDD
# - Set appropriate readahead
# - Mount with noatime
# - Use discard/fstrim for SSDs

# 5. File Descriptors
sysctl fs.file-max=1000000
ulimit -n 65535
```

---

## Interview Tips

1. **How to reduce swap usage?**
   → Set vm.swappiness=10 (or 1 for databases). Ensure enough RAM. Use memory limits (cgroups) to prevent single process from consuming all memory.

2. **Why is TIME_WAIT accumulating?**
   → Many short-lived connections. Fix: connection pooling, keep-alive, tcp_tw_reuse=1, increase ip_local_port_range.

3. **Network throughput not reaching wire speed?**
   → Check TCP buffer sizes (must match BDP). Enable window scaling. Try BBR congestion control. Check for packet drops (ip -s link).

4. **NUMA optimization for storage?**
   → Bind threads + memory to same NUMA node as NVMe device. Monitor with numastat. Remote memory access adds 50-100ns per access.

5. **How to tune for NVMe?**
   → Scheduler=none (no reordering needed), appropriate queue depth, disable power management for consistent latency, use blk-mq per-CPU queues.

---

*Last updated: March 13, 2026*
