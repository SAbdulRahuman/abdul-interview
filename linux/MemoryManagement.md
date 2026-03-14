# Memory Management

> Virtual memory, page tables, memory allocation, mmap, NUMA, allocator internals, and memory debugging.

---

## Virtual Memory Address Space

```
┌──────────────────────────────────────────────────────────────────┐
│  Process Virtual Address Space (64-bit Linux)                    │
│                                                                  │
│  High ┌─────────────────────┐                                   │
│       │  Kernel Space        │  (not accessible from user mode) │
│       ├─────────────────────┤  0xFFFF800000000000              │
│       │  Stack               │  ← grows downward               │
│       │  ↓                   │  (default 8MB limit: ulimit -s) │
│       │                      │                                   │
│       │  ↑                   │                                   │
│       │  Memory-mapped files │  (mmap, shared libraries, vDSO) │
│       │                      │                                   │
│       │  ↑                   │                                   │
│       │  Heap                │  ← grows upward (brk/sbrk)      │
│       ├─────────────────────┤                                   │
│       │  BSS (uninitialized) │  (zero-filled global/statics)    │
│       │  Data (initialized)  │  (global/static with values)     │
│       │  Text (code)         │  (read-only, executable)         │
│  Low  └─────────────────────┘  0x0000000000400000 (typical)    │
└──────────────────────────────────────────────────────────────────┘
```

### Sections in Detail

| Section | Contents | Permissions | Notes |
|---|---|---|---|
| **Text** | Machine code | Read + Execute | Shared between instances of the same binary |
| **Data** | Initialized globals/statics | Read + Write | `int x = 42;` |
| **BSS** | Uninitialized globals/statics | Read + Write | `int y;` — zero-filled at load time |
| **Heap** | Dynamic allocation | Read + Write | malloc/free, grows upward via brk() or mmap() |
| **mmap** | Shared libs, file mappings | Varies | Shared libraries (.so), anonymous mmap() |
| **Stack** | Local variables, frames | Read + Write | One per thread, grows downward, guard page at limit |

---

## Page Tables

### Multi-Level Page Table (x86-64: 4-Level)

```
Virtual Address (48-bit, 4-level paging):
┌───────┬───────┬───────┬───────┬──────────────┐
│ PML4  │  PDP  │  PD   │  PT   │   Offset     │
│(9 bit)│(9 bit)│(9 bit)│(9 bit)│  (12 bit)    │
└───┬───┴───┬───┴───┬───┴───┬───┴──────┬───────┘
    │       │       │       │          │
    ▼       ▼       ▼       ▼          │
  PML4    PDP     Page    Page         │
  Table   Table   Dir     Table        │
    │       │       │       │          │
    └───────┴───────┴───────┘          │
           ↓                           │
    Physical Page Frame  ◄─────────────┘
    (4KB aligned)
```

### 5-Level Paging (x86-64, Linux 4.14+)
- Extends virtual address space from 48-bit to 57-bit
- Adds PML5 level (5th level page table)
- Virtual address space: 128 PB (vs 256 TB with 4-level)
- Used for very large memory systems

### Page Table Entry (PTE) Flags

| Flag | Meaning |
|---|---|
| Present (P) | Page is in physical memory |
| Read/Write (R/W) | Writable if set |
| User/Supervisor (U/S) | User-accessible if set |
| Accessed (A) | Page has been read (for LRU) |
| Dirty (D) | Page has been written (needs writeback) |
| No-Execute (NX) | Cannot execute code on this page |

---

## Page Size

| Page Size | Virtual Bits | Level | Use Case |
|---|---|---|---|
| **4 KB** | 12 | Default | General purpose |
| **2 MB** | 21 | Huge page | Databases, large heaps, reduced TLB misses |
| **1 GB** | 30 | Gigantic page | HPC, very large memory workloads |

### Why Huge Pages?
- **TLB (Translation Lookaside Buffer)**: Caches page table entries
- Typical TLB: 64-1024 entries
- With 4KB pages: 64 entries × 4KB = 256KB addressable
- With 2MB pages: 64 entries × 2MB = 128MB addressable → **far fewer TLB misses**
- Critical for workloads with large working sets (databases, VMs)

---

## TLB (Translation Lookaside Buffer)

```
CPU → TLB lookup (virtual → physical)
       │
       ├─ TLB HIT → physical address → memory access (fast)
       │
       └─ TLB MISS → walk page table (slow, 4 memory accesses for 4-level)
                      → update TLB → memory access
```

### TLB Flush
- **Context switch** between processes → TLB flush (different page tables)
- **PCID (Process Context ID)**: Tags TLB entries per process → avoid full flush
- `mmap()`/`munmap()` → partial TLB invalidation
- Costs: Full TLB flush ≈ hundreds of nanoseconds + subsequent misses

---

## Page Faults

### Minor Page Fault
- Page exists in physical memory but PTE not set up
- Examples: first access to mmap'd region, COW after fork
- Cost: ~1-10 μs (kernel just updates page table)

### Major Page Fault
- Page must be fetched from disk (swap or file)
- Cost: ~1-10 ms (disk I/O required)
- Triggers: accessing swapped-out page, memory-mapped file not in page cache

```bash
# Monitor page faults
ps -o min_flt,maj_flt,cmd -p <pid>
perf stat -e page-faults,minor-faults,major-faults ./myapp
```

---

## Demand Paging and Copy-on-Write

### Demand Paging
1. Process calls `mmap()` or `malloc()` (large allocation)
2. Kernel creates VMA (Virtual Memory Area) but **no physical pages allocated**
3. First access triggers **page fault**
4. Kernel allocates physical page, maps it, resumes process
5. This means `malloc()` never fails for overcommit (until OOM)

### Copy-on-Write (COW)
1. `fork()` creates child with **same page table entries** (shared physical pages)
2. Pages marked **read-only** in both parent and child
3. When either writes → **page fault** → kernel copies the page → marks writable
4. Only modified pages are copied, unmodified pages remain shared

---

## OOM Killer

### How It Works
1. Kernel cannot satisfy memory allocation
2. Tries to reclaim memory (page cache, swap, etc.)
3. If still out of memory → **OOM killer** selects a victim
4. Victim selection based on **oom_score** (higher = more likely to kill)

### oom_score Calculation
- Based on: process RSS, page table size, number of children, nice value
- Higher memory usage → higher score
- Root processes get slight reduction

```bash
# Check OOM score
cat /proc/<pid>/oom_score         # Current score (0-1000+)
cat /proc/<pid>/oom_score_adj     # Adjustment (-1000 to 1000)

# Protect process from OOM killer
echo -1000 > /proc/<pid>/oom_score_adj    # Never kill this process

# Make process preferred OOM target
echo 1000 > /proc/<pid>/oom_score_adj

# Check if OOM happened
dmesg | grep -i "oom\|out of memory\|killed process"
journalctl -k | grep -i oom
```

### Overcommit Modes

```bash
# vm.overcommit_memory:
# 0 (default) — Heuristic: allow "reasonable" overcommit
# 1 — Always overcommit (malloc never returns NULL)
# 2 — Never overcommit (strict: commit ≤ swap + RAM×ratio)

sysctl vm.overcommit_memory
sysctl vm.overcommit_ratio   # Only matters when mode=2 (default 50%)
```

---

## NUMA (Non-Uniform Memory Access)

```
┌───────────────────────────────────────────────────────────┐
│  NUMA Topology (2-socket example)                         │
│                                                           │
│  ┌─────────────────┐        ┌─────────────────┐          │
│  │    Node 0        │        │    Node 1        │          │
│  │  ┌───┐ ┌───┐    │        │  ┌───┐ ┌───┐    │          │
│  │  │CPU│ │CPU│    │        │  │CPU│ │CPU│    │          │
│  │  │ 0 │ │ 1 │    │        │  │ 2 │ │ 3 │    │          │
│  │  └───┘ └───┘    │        │  └───┘ └───┘    │          │
│  │       │          │        │       │          │          │
│  │  ┌────▼────┐     │◄──────►│  ┌────▼────┐     │          │
│  │  │ Memory  │     │ QPI/   │  │ Memory  │     │          │
│  │  │ (local) │     │ UPI    │  │ (local) │     │          │
│  │  └─────────┘     │ ~100ns │  └─────────┘     │          │
│  │   Access: ~70ns  │        │   Access: ~70ns  │          │
│  └─────────────────┘        └─────────────────┘          │
│                                                           │
│  Local access: ~70ns    Remote access: ~100-150ns        │
│  → 1.5-2x latency penalty for remote memory              │
└───────────────────────────────────────────────────────────┘
```

### NUMA Tools

```bash
# Check NUMA topology
numactl --hardware
# Output: available nodes, CPU-to-node mapping, distances

lscpu | grep NUMA
# NUMA node0 CPU(s): 0-15
# NUMA node1 CPU(s): 16-31

# Run process pinned to NUMA node 0
numactl --cpunodebind=0 --membind=0 ./myapp

# Memory policy: interleave (spread across nodes)
numactl --interleave=all ./myapp

# Check NUMA statistics
numastat -p <pid>
# → "other_node" = remote memory accesses (bad for performance)

# Per-node memory info
cat /sys/devices/system/node/node0/meminfo
```

### Why NUMA Matters for Storage
- Storage applications often have large memory buffers (caches, buffers)
- Remote memory access adds 50-100ns per access
- For high-IOPS NVMe workloads, NUMA misses significantly impact latency
- Best practice: bind storage worker threads + memory to same NUMA node

---

## Memory Allocation

### malloc() / free()

```c
#include <stdlib.h>

// Small allocations (< MMAP_THRESHOLD, typically 128KB)
void *p = malloc(1024);     // Uses brk() to extend heap
free(p);                     // Returns to malloc's free list (may not return to OS)

// Large allocations (>= MMAP_THRESHOLD)
void *p = malloc(256 * 1024);  // Uses mmap() (anonymous mapping)
free(p);                        // munmap() returns memory to OS immediately
```

### How malloc Works Internally

```
┌──────────────────────────────────────────────────────────────┐
│  glibc malloc Architecture                                   │
│                                                              │
│  Thread 1 → Arena 1 ─┐                                      │
│  Thread 2 → Arena 2 ─┤─► Multiple arenas reduce lock        │
│  Thread 3 → Arena 1 ─┤   contention                         │
│  Thread 4 → Arena 3 ─┘                                      │
│                                                              │
│  Each Arena:                                                 │
│  ┌─────────────────────────────────────┐                     │
│  │  Fastbins (16-80 bytes, LIFO)      │ ← No coalescing    │
│  │  Smallbins (< 512 bytes, FIFO)     │ ← Exact-fit bins   │
│  │  Largebins (≥ 512 bytes, sorted)   │ ← Best-fit search  │
│  │  Unsorted bin (recently freed)     │ ← First-fit cache  │
│  └─────────────────────────────────────┘                     │
│                                                              │
│  Small allocs: brk() to extend heap                         │
│  Large allocs: mmap() (typically > 128KB)                   │
│  Threshold is adaptive (MMAP_THRESHOLD)                     │
└──────────────────────────────────────────────────────────────┘
```

### mmap() — Memory Mapping

```c
#include <sys/mman.h>

// Anonymous mapping (like malloc but for large regions)
void *p = mmap(NULL, size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1, 0);

// File mapping (memory-mapped I/O)
int fd = open("data.bin", O_RDWR);
void *p = mmap(NULL, file_size,
    PROT_READ | PROT_WRITE,
    MAP_SHARED,          // Changes visible to other processes and written to file
    fd, 0);

// Access file data directly as memory
struct record *rec = (struct record *)p;
rec->field = 42;  // Writes to file (eventually, or on msync/munmap)

// Sync changes to disk
msync(p, file_size, MS_SYNC);   // Synchronous flush
msync(p, file_size, MS_ASYNC);  // Async hint

// Unmap
munmap(p, size);
```

### mmap Flags

| Flag | Meaning |
|---|---|
| `MAP_SHARED` | Changes visible to other processes, written to file |
| `MAP_PRIVATE` | Copy-on-write — changes are private |
| `MAP_ANONYMOUS` | No file backing (like malloc) |
| `MAP_FIXED` | Map at exact address (dangerous) |
| `MAP_HUGETLB` | Use huge pages |
| `MAP_POPULATE` | Pre-fault all pages (avoid later page faults) |
| `MAP_LOCKED` | Lock pages in memory (like mlock) |

### mlock() — Pin Pages in RAM

```c
// Lock pages in memory (prevent swapping)
mlock(addr, length);      // Lock specific range
mlockall(MCL_CURRENT | MCL_FUTURE);  // Lock all current and future pages

// Unlock
munlock(addr, length);
munlockall();
```

Use cases: Real-time systems, NVRAM-backed buffers, crypto keys (prevent swap to disk)

### madvise() — Memory Usage Hints

```c
// Tell kernel about access patterns
madvise(addr, length, MADV_SEQUENTIAL);   // Will access sequentially (readahead)
madvise(addr, length, MADV_RANDOM);       // Random access (disable readahead)
madvise(addr, length, MADV_DONTNEED);     // Don't need this anymore (free pages)
madvise(addr, length, MADV_WILLNEED);     // Will need soon (pre-fault)
madvise(addr, length, MADV_HUGEPAGE);     // Prefer huge pages (THP)
madvise(addr, length, MADV_FREE);         // Lazy free (keep until kernel needs memory)
```

---

## Huge Pages

### Explicit Huge Pages

```bash
# Reserve huge pages at boot
# /etc/default/grub: GRUB_CMDLINE_LINUX="hugepagesz=2M hugepages=512"
# Or at runtime:
echo 512 > /proc/sys/vm/nr_hugepages   # Reserve 512 × 2MB = 1GB

# Check huge page status
cat /proc/meminfo | grep HugePages
# HugePages_Total:   512
# HugePages_Free:    512
# Hugepagesize:     2048 kB
```

```c
// mmap with huge pages
void *p = mmap(NULL, 2 * 1024 * 1024,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
    -1, 0);
```

### Transparent Huge Pages (THP)

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Recommended for databases: madvise (opt-in per region)
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

```c
// Opt-in with madvise
madvise(addr, length, MADV_HUGEPAGE);
```

**THP trade-offs:**
- Pro: Automatic, no reservation needed, reduces TLB misses
- Con: Compaction overhead, latency spikes during defragmentation, memory waste

---

## Allocator Alternatives

| Allocator | Strengths | Used By |
|---|---|---|
| **glibc malloc** | Default, good general purpose | Most Linux programs |
| **jemalloc** | Low fragmentation, good for long-running servers | Firefox, Redis, Rust |
| **tcmalloc** | Per-thread caching, fast allocation | Google (Go runtime uses similar approach) |
| **mimalloc** | Microsoft, compact, excellent performance | Azure services |

### Kernel Slab Allocator

```
┌──────────────────────────────────────────────────────┐
│  Slab Allocator (kernel memory)                      │
│                                                      │
│  kmem_cache: fixed-size object pool                  │
│  e.g., inode cache, dentry cache, task_struct cache  │
│                                                      │
│  ┌──── Slab ─────┐ ┌──── Slab ─────┐               │
│  │ [obj][obj][obj]│ │ [obj][___][___]│               │
│  │ [obj][obj][obj]│ │ [___][___][___]│               │
│  └───────────────┘ └───────────────┘               │
│       Full              Partial                     │
└──────────────────────────────────────────────────────┘
```

```bash
# View slab usage
slabtop               # Interactive slab cache viewer
cat /proc/slabinfo     # Raw slab data
```

---

## Memory Fragmentation

### External Fragmentation
- Free memory exists but is **scattered** in non-contiguous chunks
- Cannot satisfy large contiguous allocation even though total free is sufficient
- Fix: Compaction (kernel moves pages), buddy allocator

### Internal Fragmentation
- Allocated block is **larger** than requested (wasted within the block)
- Example: Request 20 bytes, allocator gives 32-byte slot → 12 bytes wasted
- Fix: Multiple size classes, slab allocator for fixed-size objects

---

## Memory Debugging

### Valgrind — Memory Error Detector

```bash
# Detect memory leaks
valgrind --leak-check=full --show-leak-kinds=all ./myapp

# Track all allocations
valgrind --tool=massif ./myapp      # Heap profiler
ms_print massif.out.<pid>

# Detect use-after-free, buffer overflows
valgrind --tool=memcheck ./myapp
```

### Sanitizers (Compile-Time Instrumentation)

```bash
# AddressSanitizer (ASan) — buffer overflows, use-after-free, double-free
gcc -fsanitize=address -g -O1 -o myapp myapp.c
./myapp   # Crashes with detailed error report

# MemorySanitizer — uninitialized reads
gcc -fsanitize=memory -g -O1 -o myapp myapp.c

# LeakSanitizer (included in ASan)
gcc -fsanitize=address -g -o myapp myapp.c
ASAN_OPTIONS=detect_leaks=1 ./myapp
```

### Process Memory Analysis

```bash
# Quick overview
cat /proc/<pid>/status | grep -i "vm\|rss\|threads"
# VmPeak:   500000 kB   ← Peak virtual memory
# VmSize:   450000 kB   ← Current virtual memory
# VmRSS:    120000 kB   ← Resident set size (physical memory)
# VmSwap:        0 kB   ← Swapped out memory

# Detailed memory map
pmap -x <pid>
cat /proc/<pid>/smaps_rollup
# Rss:    120000 kB      ← All RSS
# Pss:    100000 kB      ← Proportional share (shared pages divided)
# Shared_Clean:  50000 kB
# Shared_Dirty:   5000 kB
# Private_Clean: 30000 kB
# Private_Dirty: 35000 kB

# Track memory over time
pidstat -r -p <pid> 1    # Memory stats every 1 second
```

---

## Interview Tips

1. **How does malloc work?**
   → Small: heap (brk), free-list bins. Large: mmap. glibc uses arenas for multi-threading.

2. **What is a TLB? Why do huge pages help?**
   → TLB caches virtual→physical translations. Huge pages = more memory per TLB entry = fewer misses.

3. **Page fault types?**
   → Minor: page in memory, just update PTE. Major: page on disk, must load from swap/file.

4. **How does COW work with fork?**
   → Share pages read-only. Copy individual pages on write. Makes fork() fast.

5. **OOM killer — how to protect a process?**
   → `echo -1000 > /proc/<pid>/oom_score_adj` or use cgroups memory limits.

6. **NUMA optimization for storage?**
   → Bind threads + memory to same NUMA node. Avoid remote memory access (1.5-2x latency penalty).

---

*Last updated: March 13, 2026*
