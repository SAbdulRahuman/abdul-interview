# Linux Systems Programming — Interview Preparation Guide

> Essential Linux internals, systems programming, I/O stack, and debugging tools knowledge for storage infrastructure and distributed systems roles.

---

## Chapter 1 — [Process Management](linux/ProcessManagement.md)

- [ ] Process lifecycle — states (created, ready, running, waiting, zombie, terminated)
- [ ] fork() — creates child process, copy-on-write (COW) semantics
- [ ] exec() family — replaces process image (execve, execvp, execl)
- [ ] wait() / waitpid() — parent waits for child termination
- [ ] Zombie and orphan processes
- [ ] Process groups and sessions — setpgid(), setsid(), job control
- [ ] Daemon creation — double fork pattern
- [ ] /proc filesystem — /proc/\<pid\>/status, maps, fd
- [ ] Signals — SIGTERM, SIGKILL, SIGINT, SIGQUIT, SIGHUP, SIGUSR1/2, SIGPIPE, SIGCHLD
- [ ] Signal handling in C (sigaction) and Go (os/signal)

---

## Chapter 2 — [Threading & Synchronization](linux/ThreadingAndSynchronization.md)

- [ ] Thread vs Process — shared vs separate address space
- [ ] POSIX threads (pthreads) — create, join, detach
- [ ] Thread-local storage (TLS) — __thread keyword, pthread_key_create
- [ ] Mutex — pthread_mutex_lock/unlock
- [ ] RWLock — multiple readers, single writer
- [ ] Condition variables — pthread_cond_wait/signal/broadcast
- [ ] Semaphores — sem_wait/post/init
- [ ] Spinlocks — busy-wait for short critical sections
- [ ] Barriers — synchronize N threads at a point
- [ ] Concurrency bugs — data race, deadlock, livelock, priority inversion, ABA problem, false sharing
- [ ] Lock-free programming — atomic operations, CAS, memory ordering, memory barriers

---

## Chapter 3 — [Memory Management](linux/MemoryManagement.md)

- [ ] Virtual address space layout — text, data, BSS, heap, mmap, stack, kernel
- [ ] Page tables — 4-level in x86-64, virtual → physical translation
- [ ] Page size — 4KB default, huge pages (2MB/1GB)
- [ ] TLB (Translation Lookaside Buffer)
- [ ] Page faults — minor vs major
- [ ] Demand paging and copy-on-write
- [ ] OOM killer — /proc/\<pid\>/oom_score
- [ ] NUMA — non-uniform memory access, locality matters for performance
- [ ] malloc()/free() — heap allocation, brk + mmap internally
- [ ] mmap() — file mapping, anonymous memory, memory-mapped I/O
- [ ] mlock() — pin pages in RAM (prevent swapping)
- [ ] madvise() — MADV_SEQUENTIAL, MADV_DONTNEED
- [ ] Huge pages — MAP_HUGETLB, transparent huge pages (THP)
- [ ] Allocator internals — glibc malloc arenas, jemalloc, tcmalloc, slab allocator
- [ ] Memory fragmentation — external vs internal
- [ ] Memory debugging — valgrind, AddressSanitizer, MemorySanitizer, pmap

---

## Chapter 4 — [File Systems](linux/FileSystems.md)

- [ ] VFS (Virtual File System) layer — superblock, inode, dentry, file objects
- [ ] Linux storage stack — VFS → filesystem → page cache → block layer → device driver → hardware
- [ ] Inodes — metadata structure, data block pointers
- [ ] Hard links vs symbolic links
- [ ] Journaling — write-ahead log for crash consistency (ext4, XFS)
- [ ] B-tree / B+tree — XFS, Btrfs directory and extent indexing
- [ ] Extents — contiguous block ranges (ext4, XFS)
- [ ] Copy-on-Write — Btrfs, ZFS: never overwrite in place
- [ ] Allocation groups — XFS parallel allocation
- [ ] File systems compared — ext4, XFS, Btrfs, ZFS, tmpfs, procfs, sysfs

---

## Chapter 5 — [I/O Models & System Calls](linux/IOModelsAndSyscalls.md)

- [ ] Five I/O models — blocking, non-blocking, I/O multiplexing, signal-driven, async (AIO/io_uring)
- [ ] select() — portable, O(n), 1024 FD limit
- [ ] poll() — no FD limit, still O(n)
- [ ] epoll — O(1) per event, edge-triggered vs level-triggered
- [ ] io_uring — submission/completion queues, zero-syscall batched I/O (Linux 5.1+)
- [ ] File I/O syscalls — open, read/write, pread/pwrite, readv/writev
- [ ] Zero-copy — sendfile(), splice()
- [ ] mmap() for I/O
- [ ] fsync() vs fdatasync() — data+metadata vs data-only flush
- [ ] fallocate() — pre-allocate file space
- [ ] fadvise() — page cache hints
- [ ] ioctl() / fcntl() — device control, file control
- [ ] Direct I/O — O_DIRECT, O_SYNC, O_DSYNC, alignment requirements

---

## Chapter 6 — [Storage I/O Stack](linux/StorageIOStack.md)

- [ ] Block layer — bio requests, I/O schedulers (mq-deadline, bfq, kyber, none)
- [ ] Multi-queue block layer (blk-mq) — per-CPU hardware dispatch queues
- [ ] Block devices — fixed-size block access (512B/4K sectors)
- [ ] SCSI subsystem — SAS, iSCSI, Fibre Channel
- [ ] NVMe — PCIe-attached, parallel queues (64K × 64K)
- [ ] Device Mapper (dm) — LVM, LUKS, multipath, striping
- [ ] LVM — physical volumes → volume groups → logical volumes
- [ ] RAID levels — 0 (stripe), 1 (mirror), 5 (parity), 6 (dual parity), 10 (stripe of mirrors)
- [ ] Multipath I/O — redundancy and throughput
- [ ] Page cache / buffer cache — unified caching of file data in RAM
- [ ] Dirty writeback — vm.dirty_ratio, vm.dirty_bytes, vm.dirty_expire_centisecs
- [ ] Write barriers — ordering writes for crash consistency
- [ ] Storage protocols — SATA, SAS, NVMe, iSCSI, FC, NVMe-oF (TCP/RDMA)

---

## Chapter 7 — [IPC (Inter-Process Communication)](linux/IPC.md)

- [ ] Pipes — parent-child data flow
- [ ] Named pipes (FIFOs) — unrelated process communication
- [ ] Unix domain sockets — high-performance local IPC, FD passing (SCM_RIGHTS)
- [ ] TCP/UDP sockets — distributed communication
- [ ] Shared memory (POSIX) — shm_open, mmap, fastest IPC
- [ ] Message queues — structured messages
- [ ] Semaphores — synchronization
- [ ] Signals — notifications
- [ ] Memory-mapped files — shared state with persistence

---

## Chapter 8 — [Network Programming](linux/NetworkProgramming.md)

- [ ] Socket programming — socket(), bind(), listen(), accept(), connect()
- [ ] TCP state machine — SYN, ESTABLISHED, FIN_WAIT, TIME_WAIT, CLOSE_WAIT
- [ ] Socket options — SO_REUSEADDR, SO_REUSEPORT, SO_KEEPALIVE, TCP_NODELAY, SO_RCVBUF/SNDBUF, SO_LINGER, TCP_CORK
- [ ] Network namespaces — per-namespace network stack, veth pairs
- [ ] Container networking foundations

---

## Chapter 9 — [Namespaces & Cgroups (Container Foundations)](linux/NamespacesAndCgroups.md)

- [ ] Namespaces — PID, Network, Mount, UTS, IPC, User, Cgroup, Time
- [ ] unshare() and clone() with namespace flags
- [ ] Cgroups v2 — unified hierarchy
- [ ] CPU controller — cpu.max, cpu.weight
- [ ] Memory controller — memory.max, memory.current
- [ ] I/O controller — io.max, io.stat
- [ ] PID controller — pids.max
- [ ] cpuset controller — CPU/memory node pinning
- [ ] How namespaces + cgroups = containers

---

## Chapter 10 — [Performance Analysis Tools](linux/PerformanceAnalysisTools.md)

- [ ] System overview — top, htop, atop, vmstat, dstat
- [ ] CPU analysis — mpstat, pidstat, perf top, perf stat
- [ ] Memory analysis — free, /proc/meminfo, pmap, slabtop, smaps
- [ ] Disk I/O analysis — iostat, iotop, blktrace/blkparse
- [ ] Network analysis — ss, netstat, iftop, tcpdump, ip
- [ ] Process tracing — strace, ltrace, lsof
- [ ] Kernel messages — dmesg, journalctl
- [ ] perf — CPU profiling, record/report, flame graphs, hardware events
- [ ] blktrace — block I/O event tracing
- [ ] ftrace — kernel function tracing
- [ ] BPF/bpftrace — modern dynamic tracing, BCC tools (biolatency, execsnoop, opensnoop, tcplife)

---

## Chapter 11 — [Debugging Tools Deep Dive](linux/DebuggingTools.md)

- [ ] GDB — breakpoints, stepping, threads, watchpoints, core dump analysis
- [ ] strace — system call tracing (file I/O, network, memory)
- [ ] ltrace — library function tracing
- [ ] tcpdump — network packet capture and filtering
- [ ] Core dumps — enabling (ulimit, core_pattern), analyzing with GDB
- [ ] valgrind — memory leak detection, use-after-free
- [ ] AddressSanitizer (ASan) — compile-time memory error detection
- [ ] MemorySanitizer — uninitialized read detection

---

## Chapter 12 — [Performance Tuning](linux/PerformanceTuning.md)

- [ ] Memory tuning — vm.swappiness, vm.dirty_ratio, vm.overcommit_memory, vm.min_free_kbytes
- [ ] Network tuning — somaxconn, tcp_max_syn_backlog, tcp_tw_reuse, buffer sizes
- [ ] File descriptor limits — fs.file-max, fs.nr_open, ulimit -n
- [ ] Storage tuning — I/O scheduler selection, read-ahead, queue depth
- [ ] NUMA optimization — numactl, cpunodebind, membind, numastat
- [ ] sysctl — applying kernel parameter changes

---

## Chapter 13 — [Interview Questions](linux/InterviewQuestions.md)

### Process & Threading
1. What happens when you call fork()? Describe copy-on-write.
2. Process vs thread — differences and when to use each.
3. Four conditions for deadlock (Coffman conditions) and prevention.
4. Priority inversion — what it is and how Linux handles it.
5. User-space threads vs kernel threads.

### Memory
6. Virtual memory and page tables. What is a TLB miss?
7. Page faults — minor vs major. What triggers each?
8. How does malloc() work internally? brk() vs mmap()?
9. OOM killer — how does it choose which process to kill?
10. NUMA — why does memory locality matter for storage?

### I/O & Storage
11. Compare select, poll, and epoll.
12. Difference between O_SYNC, O_DSYNC, and O_DIRECT.
13. Linux I/O stack from write() syscall to data on disk.
14. Page cache and dirty writeback.
15. NVMe vs SCSI — why is NVMe faster?
16. io_uring — why is it significant for storage?

### File Systems
17. Inodes — what happens when you run out?
18. Journaling in ext4 — what does it protect against?
19. ext4 vs XFS — when to choose which?
20. Copy-on-write in filesystems (Btrfs/ZFS).

### Debugging
21. Process consuming 100% CPU — diagnosis steps.
22. Process consuming too much memory — debugging walkthrough.
23. Intermittent network failures — debugging approach.
24. Disk I/O latency increased — investigation steps.
25. Using strace to debug a hanging application.

### Performance
26. USE Method (Utilization, Saturation, Errors).
27. False sharing — detection and fix.
28. Flame graphs — how to generate and read.
29. Storage I/O performance debugging tools.
30. How cgroups + namespaces create containers.

---

*Last updated: March 13, 2026*
