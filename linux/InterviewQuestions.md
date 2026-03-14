# Linux Systems Programming — Interview Questions

> 30 interview questions with detailed answers covering processes, threading, memory, I/O, storage, file systems, debugging, and performance.

---

## Process & Threading

### 1. What happens when you call fork()? Describe copy-on-write.

**Answer:**
- `fork()` creates a new child process that is a copy of the parent
- It does NOT physically copy memory — uses **Copy-on-Write (COW)**:
  - Both parent and child share the same physical pages, marked read-only
  - When either process writes, a page fault occurs → kernel copies that specific page
  - Only modified pages are duplicated, unmodified pages remain shared
- `fork()` returns: child PID to parent, 0 to child, -1 on error
- Child inherits: open FDs, signal handlers, environment, working directory
- Child gets: new PID, PPID = parent's PID, own copy of file positions

---

### 2. Process vs thread — differences and when to use each.

**Answer:**

| Aspect | Process | Thread |
|---|---|---|
| Memory | Separate address space | Shared address space |
| Creation cost | High (page tables, COW) | Low (shared memory) |
| Communication | IPC needed (pipes, sockets, shmem) | Direct memory access |
| Crash isolation | Independent (one crash doesn't affect others) | One crash kills all threads |
| Context switch | Expensive (TLB flush) | Cheaper |

**When to use:**
- **Processes**: Need isolation (security/stability), running separate programs, untrusted code
- **Threads**: Shared data, fine-grained parallelism, I/O concurrency, lower overhead

---

### 3. Four conditions for deadlock (Coffman conditions) and prevention.

**Answer:**
1. **Mutual exclusion** — resource held exclusively → Use lock-free structures, read-write locks
2. **Hold and wait** — hold one, request another → Acquire all locks at once or none
3. **No preemption** — can't forcibly release → Use try-lock with timeout/backoff
4. **Circular wait** — cycle of waiting → Always acquire locks in a global order

Prevention strategy: Break ANY one condition. Most practical: **lock ordering** (always acquire locks in the same consistent order).

---

### 4. Priority inversion — what it is and how Linux handles it.

**Answer:**
- Low-priority thread holds a lock, high-priority thread blocks on it
- Meanwhile, medium-priority thread preempts low-priority thread (runs instead)
- Result: high-priority thread effectively blocked by medium-priority thread
- **Linux solution**: Priority inheritance — low-priority thread temporarily inherits the high-priority thread's priority while holding the lock
- In pthreads: `PTHREAD_PRIO_INHERIT` mutex attribute

---

### 5. User-space threads vs kernel threads.

**Answer:**
- **Kernel threads**: Scheduled by kernel, can use multiple CPUs, blocking syscalls don't block other threads
- **User-space threads**: Scheduled by user library, faster context switch (no kernel call), but blocking syscall blocks all threads
- **Linux uses 1:1 model**: Each pthread = one kernel thread (via clone())
- **Go uses M:N model**: Many goroutines (user-space) multiplexed onto fewer OS threads

---

## Memory

### 6. Virtual memory and page tables. What is a TLB miss?

**Answer:**
- **Virtual memory**: Each process has its own virtual address space (48-bit on x86-64)
- **Page tables**: Multi-level structure (4 levels) translating virtual → physical addresses
- **TLB**: Cache of recent page table entries (64-1024 entries)
- **TLB miss**: Virtual address not in TLB → must walk 4 levels of page tables (4 memory accesses)
- TLB miss is expensive: hundreds of nanoseconds
- Huge pages (2MB) reduce TLB misses by covering more memory per entry

---

### 7. Page faults — minor vs major. What triggers each?

**Answer:**
- **Minor page fault** (~1-10μs): Page exists in physical memory but PTE not set up
  - First access to mmap'd region, COW after fork, demand paging
  - Kernel just updates page table entry — no disk I/O
- **Major page fault** (~1-10ms): Page must be loaded from disk
  - Accessing swapped-out page, memory-mapped file not in page cache
  - Requires disk read — orders of magnitude slower
- Monitor: `ps -o min_flt,maj_flt` or `perf stat -e page-faults`

---

### 8. How does malloc() work internally? brk() vs mmap()?

**Answer:**
- **Small allocations** (< 128KB typical): Uses `brk()` system call to extend the heap
  - Memory managed in free lists with bins (fastbins, smallbins, largebins)
  - `free()` returns to malloc's free list (may not return to OS)
- **Large allocations** (>= 128KB): Uses `mmap()` for anonymous mapping
  - `free()` calls `munmap()` → returns memory to OS immediately
- **Multi-threaded**: glibc uses per-thread **arenas** to reduce lock contention
- Alternative allocators: jemalloc (Redis), tcmalloc (Google), mimalloc (Microsoft)

---

### 9. OOM killer — how does it choose which process to kill?

**Answer:**
- Triggered when: kernel cannot allocate memory and reclaim efforts fail
- Selection based on **oom_score** (0-1000+):
  - Higher RSS → higher score (more likely to be killed)
  - Root processes get slight reduction
  - Can adjust with `oom_score_adj` (-1000 to 1000)
- **Protect important processes**: `echo -1000 > /proc/<pid>/oom_score_adj`
- **Make disposable**: `echo 1000 > /proc/<pid>/oom_score_adj`
- Check history: `dmesg | grep -i "oom\|killed process"`

---

### 10. NUMA — why does memory locality matter for storage?

**Answer:**
- **NUMA**: Multiple memory nodes, each local to a set of CPUs
- Local memory access: ~70ns, Remote memory access: ~100-150ns (1.5-2x penalty)
- For storage applications:
  - High-IOPS NVMe workloads process millions of I/Os per second
  - Each remote memory access adds 30-80ns latency
  - Cumulatively can reduce IOPS by 10-30%
- **Best practice**: Bind storage worker threads + memory to the same NUMA node as the NVMe device
- Tool: `numactl --cpunodebind=0 --membind=0 ./storage-app`

---

## I/O & Storage

### 11. Compare select, poll, and epoll.

**Answer:**

| Feature | select | poll | epoll |
|---|---|---|---|
| Max FDs | 1024 (FD_SETSIZE) | Unlimited | Unlimited |
| Performance | O(n) per call | O(n) per call | O(1) per event |
| FD passing | Rebuild bitmap each call | Array (persistent-ish) | Kernel maintains (red-black tree) |
| Portability | POSIX (all Unix) | POSIX (all Unix) | Linux only |

**Why epoll is preferred:**
- O(1) per ready event (vs O(n) scan of all FDs)
- Kernel maintains the interest set (no copy from user space each call)
- Supports edge-triggered mode (more efficient, fewer wakeups)

---

### 12. O_SYNC vs O_DSYNC vs O_DIRECT.

**Answer:**
- **O_SYNC**: Every write blocks until data AND metadata are flushed to disk
  - Most expensive but strongest guarantee
- **O_DSYNC**: Every write blocks until data AND essential metadata (file size) are flushed
  - Faster than O_SYNC when inode timestamps aren't critical
- **O_DIRECT**: Bypass the page cache entirely, DMA between user buffer and device
  - Buffer must be sector-aligned (typically 512B or 4K)
  - Used by databases that manage their own buffer pool (avoids double-caching)
  - Does NOT guarantee durability — still need fsync/O_SYNC for persistence

---

### 13. Linux I/O stack from write() to data on disk.

**Answer:**
```
write() system call
  → VFS layer (generic_file_write)
  → Filesystem (ext4/XFS): allocate blocks, update metadata
  → Page cache: data written to dirty page
  → [Returns to user space — write() completes]
  
  ...Later (async writeback or forced by fsync):
  → Page cache writeback: dirty pages scheduled for I/O
  → Block layer: create bio request
  → I/O scheduler: merge, sort, prioritize
  → blk-mq: dispatch to hardware queue
  → Device driver (SCSI/NVMe)
  → Disk controller write cache
  → Persistent media (platter/NAND)
```

Key point: `write()` returns after data is in page cache — NOT on disk. `fsync()` needed for durability.

---

### 14. Page cache and dirty writeback.

**Answer:**
- **Page cache**: In-memory cache of file data blocks
  - Read hit: return from RAM (no disk I/O)
  - Read miss: read from disk, cache the page
  - Write: write to page cache (mark dirty), return immediately
- **Dirty writeback**: Background process writes dirty pages to disk
  - `dirty_background_ratio` (10%): start async writeback
  - `dirty_ratio` (20%): block writes until pages written
  - `dirty_expire_centisecs` (3000): max age before writeback
- Linux uses **unified buffer/page cache** since kernel 2.4

---

### 15. NVMe vs SCSI — why is NVMe faster?

**Answer:**
1. **Direct PCIe connection** vs controller-based (no intermediate controller overhead)
2. **64K parallel queues** (×64K depth each) vs 1 queue (32-256 depth)
3. **Minimal protocol** (8 commands) vs complex SCSI CDB parsing
4. **Per-CPU queues** map directly to blk-mq (no lock contention)
5. **Lower latency**: 10-20μs vs 100μs+ (SAS SSD) or 1ms+ (HDD)
6. **Higher IOPS**: 1M+ vs ~200K (SAS SSD)

---

### 16. io_uring — why is it significant for storage?

**Answer:**
- Modern async I/O interface (Linux 5.1+) using shared ring buffers
- **Submission Queue** (SQ) and **Completion Queue** (CQ) in mmap'd shared memory
- **Zero syscall overhead**: can batch multiple I/Os per syscall (or zero syscalls with SQ polling)
- **Zero copy**: shared memory rings between user and kernel
- **2-5x IOPS improvement** over traditional AIO in database benchmarks
- Critical for NVMe: sub-microsecond device latency means syscall overhead matters
- Supports: read, write, accept, connect, sendmsg, fsync, fallocate

---

## File Systems

### 17. Inodes — what happens when you run out?

**Answer:**
- Inode: metadata structure for each file (permissions, size, timestamps, block pointers)
- In ext4: inode count is **fixed at format time** (mkfs)
- Running out of inodes → cannot create new files even with free disk space
- Symptoms: "No space left on device" but `df -h` shows free space
- Diagnosis: `df -i` shows inode usage
- Prevention: Format with more inodes (`mkfs.ext4 -N <count>`) or use XFS (dynamic inodes)

---

### 18. Journaling in ext4 — what does it protect against?

**Answer:**
- **Protects against**: filesystem inconsistency after crash during write
- Without journaling: crash during metadata update → orphaned blocks, corrupted directories
- **How it works**: Write metadata changes to journal (write-ahead log) first, then to final location
- **Modes**: `journal` (data+metadata), `ordered` (data first, then metadata journal), `writeback` (metadata only, no data ordering)
- **Recovery**: On mount after crash, replay committed journal entries, discard uncommitted ones
- **Note**: Journaling does NOT prevent data loss — it prevents filesystem corruption

---

### 19. ext4 vs XFS — when to choose which?

**Answer:**

| Feature | ext4 | XFS |
|---|---|---|
| Large files | Good | Excellent (designed for it) |
| Parallel I/O | Limited | Excellent (allocation groups) |
| Max filesystem | 1 EB | 8 EB |
| Inode allocation | Fixed at format | Dynamic |
| Shrink filesystem | Yes | No |
| Daily workload | Great all-round | Better for large files, databases |
| Default in | Debian, Ubuntu | RHEL, CentOS, Rocky |

**Choose ext4**: General purpose, small to medium files, need to shrink filesystem
**Choose XFS**: Large files (media, databases), storage servers, high parallelism, large filesystems

---

### 20. Copy-on-write in file systems (Btrfs/ZFS).

**Answer:**
- **COW**: Never overwrite data in place — write to new location, update metadata pointer
- **Benefits**:
  - Crash consistency without journal (atomic pointer swap)
  - Nearly free snapshots (just preserve old pointers)
  - Data checksums (verify integrity, detect bit rot)
  - Efficient clones (share blocks until modified)
- **Downsides**:
  - Write amplification (new block + metadata update per write)
  - Fragmentation (data scattered, not contiguous)
  - Poor for small random writes (databases should use `chattr +C` to disable COW)

---

## Debugging

### 21. Process consuming 100% CPU — diagnosis steps.

**Answer:**
```bash
1. top -H -p <pid>              # Which thread is consuming CPU?
2. perf top -p <pid>            # Which function is hot?
3. perf record -g -p <pid> -- sleep 30  # Profile for 30 seconds
   perf report                  # Analyze (or generate flame graph)
4. strace -c -p <pid>          # Is it syscall-heavy?
5. Common causes:
   - Infinite loop / busy wait
   - Excessive lock contention (spinning)
   - Heavy computation (expected or unexpected)
   - GC thrashing (Go: GOGC, Java: GC logs)
   - Regex backtracking
```

---

### 22. Process consuming too much memory — debugging walkthrough.

**Answer:**
```bash
1. cat /proc/<pid>/status | grep VmRSS   # Current RSS
2. pidstat -r -p <pid> 1                  # Is RSS growing over time? (leak?)
3. pmap -x <pid>                          # Which memory regions are large?
4. For native code:
   - Compile with ASan: gcc -fsanitize=address
   - Or: valgrind --leak-check=full ./myapp
   - Or: bcc memleak (attach to running process)
5. For Go: go tool pprof http://localhost:6060/debug/pprof/heap
6. Common causes:
   - Memory leak (allocated, never freed)
   - Growing cache/buffer without bounds
   - Goroutine leak (Go)
   - File descriptor leak → growing kernel buffers
```

---

### 23. Intermittent network failures — debugging approach.

**Answer:**
```bash
1. ss -tn state established    # Connection states
2. ss -tn state close-wait     # CLOSE_WAIT = app bug (not closing)
3. tcpdump -nn port <port>     # Is traffic flowing?
   Look for: SYN without SYN-ACK, RST, retransmissions
4. nstat -az | grep retrans    # Retransmission count
5. ip -s link show eth0        # Interface errors/drops
6. dmesg | grep -i "link\|eth" # NIC/driver issues
7. Common causes:
   - DNS resolution failures
   - TCP keepalive timeout too long (detect dead connections)
   - Connection pool exhaustion
   - Firewall/security group changes
   - NIC ring buffer overflow (ethtool -S eth0 | grep drop)
```

---

### 24. Disk I/O latency increased — investigation steps.

**Answer:**
```bash
1. iostat -xz 1               # Which device? What's the await?
   - High await: device is slow or queue saturated
   - High %util (HDD): device is busy
2. iotop -oa                  # Which process is generating I/O?
3. Check queue depth:
   cat /sys/block/sda/inflight
4. Check disk health:
   smartctl -a /dev/sda       # SMART errors?
5. Check I/O scheduler:
   cat /sys/block/sda/queue/scheduler
6. biolatency (bcc)           # Latency distribution
7. Common causes:
   - Workload change (more I/O than usual)
   - Write amplification (COW filesystem, RAID rebuild)
   - Disk degradation (SMART errors, bad sectors)
   - Wrong I/O scheduler for workload
   - Writeback storm (dirty page ratio exceeded)
```

---

### 25. Using strace to debug a hanging application.

**Answer:**
```bash
strace -p <pid>
# Shows the syscall the process is stuck in:

# futex(0x..., FUTEX_WAIT, ...)
# → Waiting for mutex/condition → likely DEADLOCK
# → Fix: gdb -p <pid> → thread apply all bt → find lock cycle

# read(5, ...
# → Waiting for data on FD 5
# → ls -la /proc/<pid>/fd/5 → what is FD 5? (socket? pipe? file?)
# → If socket: tcpdump to check if data is being sent

# epoll_wait(3, ...
# → Waiting for events (NORMAL for event-loop servers like Go, Node.js)
# → Check if expected events are arriving

# nanosleep(...)
# → Sleeping (expected? or stuck in retry loop?)

# poll([{fd=N, events=POLLIN}], ...
# → Waiting for I/O on specific FD

# strace -c shows summary if you want to see overall call pattern
```

---

## Performance

### 26. USE Method (Utilization, Saturation, Errors).

**Answer:**
For **each resource** (CPU, memory, disk, network), check three things:
- **Utilization**: How busy? (% time busy or capacity used)
- **Saturation**: Is work queuing? (queue length, wait time)
- **Errors**: Are there failures? (error counts, retries)

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat` %usr+%sys | `vmstat` r (run queue) | `perf stat` (stalls) |
| Memory | `free` used/total | `vmstat` si/so (swap) | `dmesg` (OOM) |
| Disk | `iostat` %util | `iostat` avgqu-sz, await | `smartctl`, `dmesg` |
| Network | `ip -s link` | `ss` Recv-Q | `ip -s link` (errors) |

This systematic approach ensures no resource bottleneck is missed.

---

### 27. False sharing — detection and fix.

**Answer:**
- **What**: Two threads access different variables on the **same cache line** (64 bytes)
- Each write invalidates the other thread's cache → constant cache bouncing between cores
- **Impact**: 10-100x slower than expected
- **Detection**: `perf stat -e cache-misses` or `perf c2c record/report`
- **Fix**: Pad structures to cache line boundaries:
```c
struct counters {
    _Alignas(64) volatile long thread1_count;  // Own cache line
    _Alignas(64) volatile long thread2_count;  // Own cache line
};
```

---

### 28. Flame graphs — how to generate and read.

**Answer:**
```bash
# Generate:
perf record -g -p <pid> -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```
- **X-axis**: Percentage of total CPU time (width = time proportion)
- **Y-axis**: Stack depth (bottom = entry point, top = leaf function)
- **Reading**:
  - Look for **wide plateaus** at the top → functions consuming most CPU
  - Narrow towers are fine (deep but quick calls)
  - Compare before/after flame graphs to validate optimization
- **Colors**: Random (for differentiation), no semantic meaning

---

### 29. Storage I/O performance debugging tools.

**Answer:**
1. **iostat -xz 1**: Device-level IOPS, throughput, latency, utilization
2. **iotop**: Per-process I/O (who's generating I/O?)
3. **blktrace/blkparse**: Block-level I/O event tracing (detailed timeline)
4. **biolatency** (bcc): I/O latency distribution histogram
5. **biosnoop** (bcc): Per-I/O tracing with latency
6. **ext4slower/xfsslower** (bcc): Filesystem operations above latency threshold
7. **perf record -e block:block_rq_issue**: Block I/O events
8. **smartctl**: Disk health (SMART data)
9. **fio**: Benchmark tool to characterize device performance

---

### 30. How cgroups + namespaces create containers.

**Answer:**
- **Namespaces** provide **isolation**: each container gets its own PID space, network stack, filesystem view, hostname, IPC, and optionally its own UID/GID mapping
- **Cgroups** provide **resource limits**: CPU time, memory max, I/O bandwidth, max PIDs
- Together with a **rootfs** (container image via OverlayFS), **security** (capabilities, seccomp, SELinux/AppArmor), they create what we call a "container"
- A container is NOT a VM — it shares the host kernel
- Container runtimes (runc, containerd) automate: creating namespaces, setting cgroup limits, mounting rootfs, configuring networking, dropping capabilities

---

*Last updated: March 13, 2026*
