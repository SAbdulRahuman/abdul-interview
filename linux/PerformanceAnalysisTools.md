# Performance Analysis Tools

> System overview tools, CPU/memory/disk/network analysis, perf, blktrace, ftrace, and BPF/bpftrace.

---

## Tool Landscape

```
┌──────────────────────────────────────────────────────────────────┐
│  Linux Performance Tools Landscape                               │
│                                                                  │
│  System Overview:     top, htop, atop, glances, dstat            │
│  CPU:                 mpstat, pidstat, perf, turbostat           │
│  Memory:              free, vmstat, pmap, slabtop               │
│  Disk I/O:            iostat, iotop, blktrace, bcc/biolatency   │
│  Network:             ss, netstat, iftop, tcpdump, wireshark    │
│  File System:         df, du, lsof, inotifywait                 │
│  Process:             ps, pstree, strace, ltrace                │
│  Kernel:              dmesg, ftrace, perf, bpftrace             │
│  Profiling:           perf record/report, flamegraph            │
└──────────────────────────────────────────────────────────────────┘
```

### USE Method (Brendan Gregg)

For every resource, check:
- **U**tilization — how busy is it? (time or capacity)
- **S**aturation — is work queuing? (queue length)
- **E**rrors — are there errors? (error counts)

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat` (%usr+%sys) | `vmstat` (run queue) | `perf stat` (page faults) |
| Memory | `free` (used/total) | `vmstat` (swapping) | `dmesg` (OOM) |
| Disk | `iostat` (%util) | `iostat` (avgqu-sz) | `smartctl`, `dmesg` |
| Network | `ip -s link` | `ss` (Recv-Q) | `ip -s link` (errors, drops) |

---

## System Overview Tools

### top / htop

```bash
# top — system overview
top
# Key commands:
# 1       → per-CPU view
# H       → show threads
# P       → sort by CPU
# M       → sort by memory
# c       → show full command
# f       → select fields
# k       → kill process

# htop — better interactive view
htop
# F5      → tree view
# F6      → sort by column
# F9      → kill process

# top for specific process (per-thread)
top -H -p <pid>
```

### vmstat — System-Wide Activity

```bash
vmstat 1           # Update every 1 second
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  1  0      0 8000000 200000 3000000   0    0    10   100 1000 2000 10  5 84  1  0

# Key columns:
# r:  run queue (processes waiting for CPU) — if > #CPUs, CPU saturated
# b:  blocked (waiting for I/O)
# si/so: swap in/out (should be 0 on healthy system)
# bi/bo: block I/O in/out (KB/s)
# us: user CPU %
# sy: system CPU %
# wa: I/O wait %
# id: idle %
```

### uptime and Load Average

```bash
uptime
# 10:00:00 up 30 days, load average: 4.50, 3.20, 2.10

# Load average: 1-min, 5-min, 15-min
# Represents average number of runnable + uninterruptible tasks
# Rule of thumb: If load > #CPUs, system is overloaded
# Rising trend (1min > 5min > 15min) = load increasing
# Falling trend = load decreasing
```

---

## CPU Analysis

```bash
# === Per-CPU utilization ===
mpstat -P ALL 1
# CPU    %usr   %sys  %iowait  %irq   %soft  %steal  %idle
# all     10      5       1      0       1       0     83
#   0     40     10       0      0       2       0     48    ← hot CPU
#   1      2      1       0      0       0       0     97

# === Per-process CPU ===
pidstat -u -p <pid> 1
# UID  PID  %usr  %system  %CPU  CPU  Command
#   0  1234  30.0     5.0  35.0    2  myapp

# Also:
pidstat -t -p <pid> 1         # Per-thread CPU
pidstat -u 1 | sort -k8 -rn   # All processes, sorted by CPU

# === CPU events ===
perf stat ./myapp
# Performance counter stats:
#  1,500,000,000  instructions
#    500,000,000  cycles                (IPC: 3.0)
#     50,000,000  cache-misses          (5% of cache-references)
#      5,000,000  branch-misses         (0.5% of branches)
#          1,234  page-faults
#            500  context-switches

# === Real-time CPU profiling ===
perf top                      # Interactive, shows functions consuming CPU
perf top -p <pid>             # For specific process
```

---

## Memory Analysis

```bash
# === System memory ===
free -h
#               total    used    free    shared  buff/cache  available
# Mem:           32G     10G     8G      100M       14G        20G
# Swap:           8G      0B     8G

# Key: "available" = free + reclaimable cache (what you can actually use)
# buff/cache is reclaimable under memory pressure

# === Detailed memory ===
cat /proc/meminfo | head -20
# MemTotal:       32000000 kB
# MemFree:         8000000 kB
# MemAvailable:   20000000 kB
# Buffers:         200000 kB
# Cached:        13000000 kB
# SwapCached:           0 kB
# Active:        12000000 kB
# Inactive:       8000000 kB
# Dirty:            50000 kB    ← pages waiting to be written to disk
# Slab:           1000000 kB    ← kernel slab allocator

# === Per-process memory ===
# RSS: Resident Set Size (physical memory actually used)
# VSZ: Virtual Size (total virtual memory mapped)
ps aux --sort=-rss | head -10    # Sort by RSS

# Detailed per-process
cat /proc/<pid>/status | grep -i "vm\|rss\|threads"
# VmPeak:   500000 kB   ← Peak virtual memory
# VmSize:   450000 kB   ← Current virtual memory
# VmRSS:    120000 kB   ← Resident memory
# VmSwap:        0 kB   ← Swapped out
# Threads:       8

# Memory map
pmap -x <pid>                    # Detailed memory regions
cat /proc/<pid>/smaps_rollup     # PSS (proportional share)

# === Kernel slab ===
slabtop                          # Interactive slab usage
cat /proc/slabinfo | sort -k3 -rn | head  # Sort by active objects

# === NUMA ===
numastat                         # Per-node allocation stats
numastat -p <pid>                # Per-process NUMA stats
```

---

## Disk I/O Analysis

```bash
# === iostat — Device-level I/O ===
iostat -xz 1
# Device     r/s    w/s   rkB/s   wkB/s  rrqm/s  wrqm/s  await  r_await  w_await  %util
# nvme0n1   5000   3000  100000   60000       0       0    0.5     0.3      0.8     40%
# sda        100    200    5000   10000       5      10     5.0     2.0      7.0     80%

# Key metrics:
# r/s, w/s:        Read/write IOPS
# rkB/s, wkB/s:    Read/write throughput (KB/s)
# rrqm/s, wrqm/s:  Request merges per second
# await:            Average I/O latency (ms) — includes queue time
# r_await, w_await: Separate read/write latency
# %util:            Device utilization
#   HDD: meaningful (100% = fully busy)
#   SSD/NVMe: misleading (can handle many parallel I/Os)

# === iotop — Per-process I/O ===
iotop -oa                        # Accumulated I/O per process
iotop -oPa                       # Accumulated, only active processes

# === I/O scheduler ===
cat /sys/block/sda/queue/scheduler
cat /sys/block/nvme0n1/queue/scheduler

# === Queue depth ===
cat /sys/block/nvme0n1/queue/nr_requests   # Max queue depth
cat /sys/block/nvme0n1/inflight            # Currently in-flight reads/writes

# === Device sector size ===
cat /sys/block/sda/queue/hw_sector_size
cat /sys/block/sda/queue/logical_block_size
```

---

## Network Analysis

```bash
# === Socket statistics ===
ss -tnlp                        # Listening TCP sockets with process
ss -tn state established | wc -l # Count established connections
ss -tn state time-wait | wc -l   # Count TIME_WAIT (too many = problem)
ss -tn state close-wait          # CLOSE_WAIT (app bug — not closing sockets)

# === Interface statistics ===
ip -s link show eth0
# TX/RX: packets, bytes, errors, dropped, overruns
# errors/dropped > 0 → investigate NIC, driver

# === Network traffic ===
iftop                            # Interactive bandwidth per connection
nload                           # Interface bandwidth graph

# === TCP statistics ===
cat /proc/net/snmp | grep Tcp
nstat -az | grep -i retrans      # Retransmissions (network issues)
nstat -az | grep -i overflow     # Listen overflow (backlog full)
```

---

## Process Tracing

```bash
# === strace — system call tracing ===
strace -f -p <pid>                             # Trace all threads
strace -e trace=open,read,write -p <pid>       # File I/O only
strace -e trace=network -p <pid>               # Network syscalls
strace -e trace=memory -p <pid>                # Memory (mmap, brk)
strace -c -p <pid>                             # Summary statistics
strace -T -p <pid>                             # Show time in each syscall

# === ltrace — library call tracing ===
ltrace -p <pid>                                # Trace shared library calls
ltrace -e malloc+free -p <pid>                 # Track memory allocation

# === lsof — open files ===
lsof -p <pid>                                  # All open files/sockets
lsof -i :8080                                  # Processes using port 8080
lsof -i tcp                                    # All TCP connections
ls /proc/<pid>/fd | wc -l                      # Count open FDs

# === Kernel messages ===
dmesg -T | tail -50                            # Recent kernel messages
dmesg -T | grep -i "error\|warn\|oom\|failed"
journalctl -k --since "5 min ago"              # Kernel log via systemd
```

---

## perf — Linux Performance Profiler

### CPU Profiling

```bash
# Record CPU profile (samples stack traces)
perf record -g -p <pid> -- sleep 30
# -g: record call graphs (stack traces)
# -p: target PID
# sleep 30: record for 30 seconds

# Record all CPUs (system-wide)
perf record -ag -- sleep 10

# Analyze
perf report                    # Interactive TUI
perf report --stdio            # Terminal text output
perf report --sort comm,dso    # Sort by process and library
```

### Flame Graphs

```bash
# Generate flame graph
perf record -g -p <pid> -- sleep 30
perf script > out.perf

# Using Brendan Gregg's FlameGraph tools
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flame.svg

# Open flame.svg in browser
# Width = time (CPU cycles)
# Height = stack depth
# Look for wide plateaus (functions consuming most CPU)
```

### Hardware Events

```bash
# CPU performance counters
perf stat ./myapp
perf stat -e cycles,instructions,cache-misses,cache-references,branch-misses ./myapp

# Specific events
perf stat -e L1-dcache-load-misses,LLC-load-misses ./myapp

# Block I/O events
perf record -e block:block_rq_issue -a -- sleep 10
perf report

# Scheduler events
perf sched record -- sleep 5
perf sched latency               # Scheduling latency
perf sched map                   # CPU timeline

# List available events
perf list
```

### perf Annotate

```bash
# Source-level annotation (needs debug symbols)
perf record -g ./myapp
perf annotate hot_function
# Shows assembly + source with per-instruction CPU percentage
```

---

## blktrace / blkparse — Block I/O Tracing

### Recording Block I/O Events

```bash
# Trace block I/O on a device
blktrace -d /dev/sda -o trace -w 10    # 10 seconds

# Parse trace
blkparse -i trace | head -50

# Output format:
#  8,0    3       1     0.000000000  1234  Q  WS 1048576 + 8 [myapp]
#  8,0    3       2     0.000001234  1234  D  WS 1048576 + 8 [myapp]
#  8,0    3       3     0.000125678     0  C  WS 1048576 + 8 [0]
#
# Fields: device, CPU, seq, timestamp, PID, action, R/W, sector+count, [process]
# Actions:
#   Q = Queued (bio created)
#   G = Get request (allocated)
#   I = Inserted into scheduler
#   D = Dispatched to driver
#   C = Completed
#   M = Merged with existing request
```

### I/O Latency Analysis

```bash
# Calculate D→C latency (dispatch to completion = device latency)
blkparse -i trace -f "%d %T.%t %S %n\n" | awk '
    /D/ { start[$3] = $2 }
    /C/ { if ($3 in start) print $2 - start[$3], $3; delete start[$3] }
'

# blktrace with btrace (simplified)
btrace /dev/sda
```

### Modern Alternative: BCC biotools

```bash
# Block I/O latency histogram
biolatency                 # Distribution of all block I/O latencies
biolatency -D              # Per-device
biolatency -Q              # Include queue time

# Per-I/O tracing
biosnoop                   # Each block I/O with latency
biosnoop -d /dev/sda       # Filter by device

# Slow filesystem operations
ext4slower 10              # ext4 operations > 10ms
xfsslower 10               # XFS operations > 10ms
```

---

## ftrace — Kernel Function Tracer

### Basic Usage

```bash
# ftrace directory
cd /sys/kernel/tracing

# Available tracers
cat available_tracers
# function function_graph nop

# Function tracer
echo function > current_tracer
echo "ext4_*" > set_ftrace_filter       # Only ext4 functions
echo 1 > tracing_on
cat trace_pipe                           # Stream output
# Press Ctrl+C to stop
echo 0 > tracing_on

# Function graph tracer (with call depth)
echo function_graph > current_tracer
echo "vfs_read" > set_graph_function
echo 1 > tracing_on
cat trace_pipe
# Shows call tree:
#  vfs_read() {
#    rw_verify_area() {
#      security_file_permission();
#    }
#    ext4_file_read_iter() {
#      generic_file_read_iter() {
#        ...
#      }
#    }
#  }

# Clear filter
echo > set_ftrace_filter

# Available functions to trace
cat available_filter_functions | wc -l    # Thousands of functions
cat available_filter_functions | grep ext4
```

### Trace Events

```bash
# List trace events
ls /sys/kernel/tracing/events/
# block/, sched/, net/, syscalls/, ext4/, ...

# Enable specific events
echo 1 > /sys/kernel/tracing/events/block/block_rq_issue/enable
echo 1 > /sys/kernel/tracing/events/sched/sched_switch/enable

cat /sys/kernel/tracing/trace_pipe

# Disable
echo 0 > /sys/kernel/tracing/events/block/block_rq_issue/enable
```

---

## BPF / bpftrace — Modern Dynamic Tracing

### bpftrace One-Liners

```bash
# Trace file opens
bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%s %s\n", comm, str(args->filename));
}'

# Block I/O count by process
bpftrace -e 'tracepoint:block:block_rq_issue {
    @[comm] = count();
}'

# VFS read size distribution
bpftrace -e 'kprobe:vfs_read {
    @bytes = hist(arg2);
}'

# Syscall latency
bpftrace -e 'tracepoint:syscalls:sys_enter_read {
    @start[tid] = nsecs;
}
tracepoint:syscalls:sys_exit_read /@start[tid]/ {
    @us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

# CPU scheduling latency
bpftrace -e 'tracepoint:sched:sched_switch {
    @[comm] = count();
}'

# Track memory allocations
bpftrace -e 'tracepoint:kmem:kmalloc {
    @bytes = hist(args->bytes_alloc);
}'
```

### BCC Tools (Python + BPF)

```bash
# Pre-built tools in bcc-tools package

# Process execution tracking
execsnoop                  # Trace new process execution

# File operations
opensnoop                  # Trace file opens
filelife                   # Track file creation to deletion
fileslower 10              # Files slower than 10ms

# Block I/O
biolatency                 # Block I/O latency histogram
biosnoop                   # Per-I/O tracing
biotop                     # Top-like for block I/O

# Networking
tcplife                    # TCP session lifecycle
tcpconnect                 # Trace TCP connect() calls
tcpretrans                 # TCP retransmissions

# Memory
cachestat                  # Page cache hit/miss statistics
memleak                    # Memory leak detection

# CPU
runqlat                    # Run queue latency
cpufreq                    # CPU frequency sampling
profile                    # CPU profiling via BPF

# General
funccount 'vfs_*'          # Count function calls
trace 'do_sys_open "%s", arg2'  # Trace with arguments
argdist -H 'r::vfs_read():int:$retval'  # Return value histogram
```

### Why BPF Over ftrace/kprobes?

| Feature | ftrace | BPF/bpftrace |
|---|---|---|
| Safety | Can crash kernel (kprobes) | Verified by BPF verifier |
| Overhead | Low | Very low (JIT compiled) |
| Flexibility | Limited filtering | Programmable (maps, histograms) |
| User space | Limited | Full access (uprobe) |
| Aggregation | None (raw events) | In-kernel (maps, histograms) |

---

## Quick Reference

### "Process is slow" Checklist

```bash
# 1. CPU bound?
top -H -p <pid>                    # Per-thread CPU
perf top -p <pid>                  # What functions are hot?
perf record -g -p <pid> -- sleep 30 && perf report

# 2. I/O bound?
pidstat -d -p <pid> 1              # Disk I/O per process
strace -c -p <pid>                 # Which syscalls?
iostat -xz 1                       # Device busy?

# 3. Memory pressure?
cat /proc/<pid>/status | grep VmRSS
cat /proc/<pid>/smaps_rollup
free -h                             # System memory available?
dmesg | grep oom                    # OOM events?

# 4. Network?
ss -tnp | grep <pid>               # Connections
strace -e trace=network -p <pid>   # Network syscalls
tcpdump -i eth0 -nn port <port>    # Packet capture
```

---

## Interview Tips

1. **How do you approach a performance problem?**
   → USE method: check Utilization, Saturation, Errors for each resource (CPU, memory, disk, network). Start broad (top, vmstat, iostat), then drill down (perf, strace, blktrace).

2. **What is perf and how to generate a flame graph?**
   → perf record -g -p <pid> -- sleep 30, then perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg. Width = CPU time, height = stack depth.

3. **iostat %util — is 100% meaningful?**
   → For HDD (single queue): yes, fully busy. For SSD/NVMe (parallel queues): misleading, can handle many concurrent I/Os at 100%.

4. **What is bpftrace?**
   → High-level tracing language using BPF. Safely traces kernel/user functions, builds histograms and aggregations in-kernel. No kernel module needed, JIT compiled, verified for safety.

5. **How to find a memory leak in a running process?**
   → Check RSS growth over time (pidstat -r). Use valgrind (if can restart), or bcc memleak tool (trace un-freed allocations). Check /proc/<pid>/smaps for growing anonymous regions.

---

*Last updated: March 13, 2026*
