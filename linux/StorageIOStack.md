# Storage I/O Stack

> Block layer, I/O schedulers, blk-mq, SCSI, NVMe, Device Mapper, LVM, RAID, page cache, and storage protocols.

---

## Linux Block I/O Stack Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Linux Block I/O Stack                                           │
│                                                                  │
│  File System / Application                                       │
│      │                                                           │
│      ▼                                                           │
│  Page Cache (dirty page writeback)                              │
│      │                                                           │
│      ▼                                                           │
│  Bio Layer (block I/O requests)                                 │
│      │                                                           │
│      ▼                                                           │
│  I/O Scheduler (merge, sort, prioritize)                        │
│      │   ┌────────────────────────────────┐                     │
│      │   │ mq-deadline — latency-focused  │                     │
│      │   │ bfq        — fairness-focused  │                     │
│      │   │ kyber      — latency targets   │                     │
│      │   │ none       — no scheduling     │                     │
│      │   │             (NVMe, fast SSDs)  │                     │
│      │   └────────────────────────────────┘                     │
│      ▼                                                           │
│  Multi-Queue Block Layer (blk-mq)                               │
│      │     (hardware dispatch queues per CPU)                   │
│      ▼                                                           │
│  SCSI Layer / NVMe Driver                                       │
│      │                                                           │
│      ▼                                                           │
│  Hardware (HDD/SSD/NVMe)                                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Block Devices

### What Is a Block Device?
- Storage device accessed in **fixed-size blocks** (sectors)
- Sector size: traditionally 512 bytes, modern drives use 4K (Advanced Format)
- Supports random access (unlike character devices)
- Examples: HDDs, SSDs, NVMe drives, loop devices, LVM volumes

```bash
# List block devices
lsblk
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,PHY-SEC,LOG-SEC

# Block device info
blockdev --getsz /dev/sda          # Size in 512-byte sectors
blockdev --getss /dev/sda          # Logical sector size
blockdev --getpbsz /dev/sda       # Physical sector size
cat /sys/block/sda/queue/hw_sector_size
cat /sys/block/sda/queue/logical_block_size
cat /sys/block/sda/queue/physical_block_size
```

---

## Bio Layer (Block I/O Requests)

### What Is a Bio?
- `struct bio` — fundamental block I/O request in the kernel
- Contains: device, start sector, length, data pages (bio_vec), direction (read/write)
- File system or page cache creates bios → submitted to block layer

```
┌─────────────────────────────────────────────────┐
│  Bio Structure                                   │
│                                                  │
│  bio_disk:  target device                       │
│  bi_iter:   sector, size, remaining             │
│  bi_opf:    operation (READ, WRITE, FLUSH)      │
│  bi_io_vec: ┌──────────────┐                    │
│             │ page + offset │ ← scatter-gather  │
│             │ page + offset │   list of memory  │
│             │ page + offset │   pages           │
│             └──────────────┘                    │
└─────────────────────────────────────────────────┘
```

### Bio Merging
- Adjacent bios (contiguous sectors) are **merged** into larger requests
- Reduces number of I/O operations sent to hardware
- Plugging: kernel batches bios before dispatching (plug/unplug mechanism)

---

## I/O Schedulers

### Purpose
- Reorder and merge I/O requests for better performance
- Different algorithms suit different workloads and devices

### Available Schedulers

| Scheduler | Strategy | Best For | Notes |
|---|---|---|---|
| **mq-deadline** | Deadline-based, prevents starvation | Rotating disks, general use | Default for HDDs |
| **bfq** (Budget Fair Queueing) | Per-process fairness, low-latency interactive | Desktop, mixed workloads | Good for latency-sensitive apps |
| **kyber** | Token-based, latency targets | Fast devices (SSDs) | Lightweight, simple |
| **none** | Pass-through, no reordering | NVMe, very fast SSDs | Minimal overhead, let device handle scheduling |

### mq-deadline Details

```
┌─────────────────────────────────────────────────────────────┐
│  mq-deadline Scheduler                                      │
│                                                             │
│  Read queue:  sorted by sector (for sequential merging)     │
│  Write queue: sorted by sector                              │
│  Read FIFO:   sorted by deadline (500ms default)            │
│  Write FIFO:  sorted by deadline (5000ms default)           │
│                                                             │
│  Dispatch: prefer reads, but switch to writes to            │
│  prevent starvation. Always dispatches expired              │
│  deadlines first.                                           │
│                                                             │
│  Reads get priority because they are typically synchronous  │
│  (application waiting for data)                             │
└─────────────────────────────────────────────────────────────┘
```

### Changing Scheduler

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# Change scheduler (runtime)
echo none > /sys/block/nvme0n1/queue/scheduler

# Persistent (udev rule)
# /etc/udev/rules.d/60-scheduler.rules
# ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
# ACTION=="add|change", KERNEL=="sd*", ATTR{queue/scheduler}="mq-deadline"

# Scheduler parameters
ls /sys/block/sda/queue/iosched/
# read_expire, write_expire, fifo_batch, writes_starved...
```

---

## Multi-Queue Block Layer (blk-mq)

### Why blk-mq?

```
Legacy single-queue (pre-3.13):
┌─────────────┐     ┌──────────┐
│ All CPUs ────┼────►│ Single   │──► Device
│              │     │ Queue    │
└─────────────┘     └──────────┘
Problem: Single queue = lock contention on multi-core systems
         Bottleneck for NVMe (which has 64K hardware queues)

blk-mq (Linux 3.13+):
┌─────────┐  ┌──────────────┐  ┌────────────────┐
│ CPU 0 ──┼─►│ Software Q 0 │─►│ Hardware Q 0   │──► Device
│ CPU 1 ──┼─►│ Software Q 1 │─►│ Hardware Q 1   │──► Device
│ CPU 2 ──┼─►│ Software Q 2 │─►│ Hardware Q 2   │──► Device
│ CPU 3 ──┼─►│ Software Q 3 │─►│ Hardware Q 3   │──► Device
└─────────┘  └──────────────┘  └────────────────┘
Each CPU has its own queue → no lock contention
Maps directly to NVMe hardware submission queues
```

### Key Benefits
- Per-CPU software queues → eliminates lock contention
- Maps to hardware queues (NVMe supports up to 64K queues × 64K entries)
- Scales linearly with CPU count
- Essential for NVMe devices (millions of IOPS)

```bash
# Check blk-mq info
cat /sys/block/nvme0n1/queue/nr_hw_queues    # Hardware queues
cat /sys/block/nvme0n1/queue/nr_requests     # Queue depth per HW queue
```

---

## SCSI Subsystem

### SCSI Architecture

```
┌───────────────────────────────────────────────────────────┐
│  SCSI Subsystem                                           │
│                                                           │
│  Upper Level Drivers:                                     │
│  ├── sd (SCSI disk)    → /dev/sdX                        │
│  ├── sr (SCSI CD-ROM)  → /dev/srX                        │
│  ├── st (SCSI tape)    → /dev/stX                        │
│  └── sg (SCSI generic) → /dev/sgX (passthrough)         │
│                                                           │
│  Mid-Level (SCSI core):                                   │
│  ├── Command queuing                                      │
│  ├── Error handling                                       │
│  └── Device discovery                                     │
│                                                           │
│  Low-Level Drivers (Host Bus Adapters):                   │
│  ├── megaraid_sas    (Dell PERC/MegaRAID)               │
│  ├── mpt3sas         (LSI/Broadcom)                      │
│  ├── qla2xxx         (QLogic FC HBA)                     │
│  ├── iscsi_tcp       (iSCSI initiator)                   │
│  └── virtio_scsi     (virtio SCSI)                       │
└───────────────────────────────────────────────────────────┘
```

### SCSI Command Protocol
- CDB (Command Descriptor Block): structured command packet
- Common commands: READ(10), WRITE(10), INQUIRY, TEST_UNIT_READY, MODE_SENSE
- Sense data: error information returned by device
- Queue depth: number of commands outstanding to device (typical: 32-256 for SAS)

```bash
# SCSI device info
lsscsi
lsscsi -s              # With size
sg_inq /dev/sda        # SCSI inquiry
sg_readcap /dev/sda    # Capacity
smartctl -a /dev/sda   # SMART health
```

---

## NVMe (Non-Volatile Memory Express)

### NVMe Architecture

```
┌───────────────────────────────────────────────────────────┐
│  NVMe Architecture                                        │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Admin Queue (1 pair)                                │ │
│  │  - Device identification, feature set/get            │ │
│  │  - Namespace management                              │ │
│  │  - Firmware operations                               │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  I/O Queues (up to 64K pairs)                       │ │
│  │                                                      │ │
│  │  ┌──────────────┐  ┌──────────────┐                 │ │
│  │  │ Submission Q  │  │ Completion Q  │  ← Per CPU    │ │
│  │  │ (up to 64K    │  │ (up to 64K    │                │ │
│  │  │  entries)     │  │  entries)     │                │ │
│  │  └──────────────┘  └──────────────┘                 │ │
│  │                                                      │ │
│  │  × 64K queue pairs (typically 1 per CPU core)       │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  Transport: PCIe (direct bus connection)                 │
│  Latency: ~10-20μs (vs ~1-10ms for SATA/SAS)           │
└───────────────────────────────────────────────────────────┘
```

### Why NVMe Is Faster Than SCSI

| Feature | SCSI (SAS/SATA) | NVMe |
|---|---|---|
| Interface | SAS/SATA controller | PCIe direct |
| Queues | 1 (SAS: 256 depth) | Up to 64K queues |
| Queue depth | 32-256 | 64K per queue |
| Latency | ~1ms+ (HDD), ~100μs (SSD) | ~10-20μs |
| Protocol overhead | CDB parsing, SCSI mid-layer | Minimal (8 commands) |
| CPU efficiency | High CPU overhead | Designed for parallelism |
| Max IOPS | ~200K (SAS SSD) | ~1M+ (modern NVMe) |

```bash
# NVMe tools
nvme list                              # List NVMe devices
nvme id-ctrl /dev/nvme0                # Controller identification
nvme smart-log /dev/nvme0              # SMART data
nvme get-feature /dev/nvme0 -f 0x07   # Number of queues
cat /sys/block/nvme0n1/queue/nr_hw_queues   # Active queues
```

### NVMe Namespaces
- NVMe supports **namespaces** — logical partitions of the device
- Similar to LUNs in SCSI world
- Each namespace appears as `/dev/nvmeXnY`
- Allows multi-tenancy on a single physical device

---

## Device Mapper (dm)

### Overview

```
┌───────────────────────────────────────────────────────────┐
│  Device Mapper Framework                                   │
│                                                           │
│  Virtual block devices built from underlying devices      │
│                                                           │
│  ┌─────────────┐                                         │
│  │ dm-linear    │ → Concatenation (LVM)                  │
│  ├─────────────┤                                         │
│  │ dm-striped   │ → Striping across devices              │
│  ├─────────────┤                                         │
│  │ dm-mirror    │ → Mirroring (RAID 1)                   │
│  ├─────────────┤                                         │
│  │ dm-snapshot  │ → COW snapshots                        │
│  ├─────────────┤                                         │
│  │ dm-thin      │ → Thin provisioning                    │
│  ├─────────────┤                                         │
│  │ dm-crypt     │ → LUKS encryption                      │
│  ├─────────────┤                                         │
│  │ dm-multipath │ → Multipath I/O                        │
│  ├─────────────┤                                         │
│  │ dm-cache     │ → SSD caching for HDD                  │
│  └─────────────┘                                         │
│                                                           │
│  All appear as /dev/dm-X or /dev/mapper/<name>           │
└───────────────────────────────────────────────────────────┘
```

```bash
# View device mapper devices
dmsetup ls
dmsetup table /dev/mapper/vg0-lv0
dmsetup status

# Device mapper target types
dmsetup targets
```

---

## LVM (Logical Volume Manager)

### LVM Architecture

```
┌───────────────────────────────────────────────────────────────┐
│  LVM Hierarchy                                                │
│                                                               │
│  Physical Volumes (PV)    Volume Groups (VG)    Logical Vols │
│  ┌──────────┐                                                │
│  │ /dev/sda1│─┐                                              │
│  └──────────┘ │  ┌──────────┐  ┌──────────────┐             │
│  ┌──────────┐ ├─►│   vg0    │─►│ lv_root      │ /           │
│  │ /dev/sdb1│─┤  │          │─►│ lv_data      │ /data       │
│  └──────────┘ │  └──────────┘  │ lv_swap      │ swap        │
│  ┌──────────┐ │                └──────────────┘             │
│  │ /dev/sdc1│─┘                                              │
│  └──────────┘                                                │
│                                                               │
│  Physical Extents (PEs): 4MB default chunks                  │
│  Logical Extents (LEs): mapped to PEs                        │
└───────────────────────────────────────────────────────────────┘
```

### Key Commands

```bash
# Physical Volume
pvcreate /dev/sdb1 /dev/sdc1
pvs                             # List PVs
pvdisplay /dev/sdb1

# Volume Group
vgcreate vg0 /dev/sdb1 /dev/sdc1
vgs                             # List VGs
vgextend vg0 /dev/sdd1         # Add disk to VG

# Logical Volume
lvcreate -L 100G -n lv_data vg0
lvcreate -l 100%FREE -n lv_data vg0    # Use all free space
lvs                             # List LVs
lvextend -L +50G /dev/vg0/lv_data      # Grow LV
resize2fs /dev/vg0/lv_data             # Grow filesystem (ext4)
xfs_growfs /mnt/data                    # Grow filesystem (XFS)

# Thin Provisioning
lvcreate --type thin-pool -L 500G -n thinpool vg0
lvcreate --type thin -V 100G --thinpool thinpool -n thin_vol1 vg0

# Snapshots
lvcreate -s -L 10G -n snap_data /dev/vg0/lv_data    # Snapshot
lvconvert --merge /dev/vg0/snap_data                  # Restore from snapshot
```

---

## RAID Levels

```
┌──────────────────────────────────────────────────────────────────┐
│  RAID Level Comparison                                           │
│                                                                  │
│  RAID 0 (Stripe):    ┌──┐┌──┐┌──┐     No redundancy            │
│                       │A1││A2││A3│     N× performance            │
│                       │B1││B2││B3│     N× capacity               │
│                       └──┘└──┘└──┘     1 disk fail = data loss  │
│                                                                  │
│  RAID 1 (Mirror):    ┌──┐┌──┐         Full redundancy           │
│                       │A1││A1│         2× read, 1× write        │
│                       │B1││B1│         50% capacity              │
│                       └──┘└──┘         Can lose 1 disk          │
│                                                                  │
│  RAID 5 (Parity):    ┌──┐┌──┐┌──┐     Single parity            │
│                       │A1││A2││Ap│     (N-1)× capacity          │
│                       │B1││Bp││B2│     Can lose 1 disk          │
│                       │Cp││C1││C2│     Write penalty (parity)   │
│                       └──┘└──┘└──┘     Slow rebuild             │
│                                                                  │
│  RAID 6 (Dual Parity):┌──┐┌──┐┌──┐┌──┐   Dual parity          │
│                        │A1││A2││Ap││Aq│   (N-2)× capacity      │
│                        └──┘└──┘└──┘└──┘   Can lose 2 disks     │
│                                                                  │
│  RAID 10 (1+0):      ┌──┐┌──┐ ┌──┐┌──┐   Stripe of mirrors   │
│                       │A1││A1│ │A2││A2│   N/2 capacity         │
│                       └──┘└──┘ └──┘└──┘   Best performance     │
│                       Mirror 1  Mirror 2   Can lose 1 per pair │
└──────────────────────────────────────────────────────────────────┘
```

### RAID Comparison Table

| Level | Min Disks | Capacity | Read | Write | Fault Tolerance | Use Case |
|---|---|---|---|---|---|---|
| **RAID 0** | 2 | N × disk | N× | N× | None | Temp/scratch |
| **RAID 1** | 2 | 1 × disk | 2× | 1× | 1 disk | OS, boot |
| **RAID 5** | 3 | (N-1) × disk | (N-1)× | Slow (parity) | 1 disk | General storage |
| **RAID 6** | 4 | (N-2) × disk | (N-2)× | Slow (dual parity) | 2 disks | Enterprise |
| **RAID 10** | 4 | N/2 × disk | N× | N/2× | 1 per mirror | Databases, high perf |

### Write Penalty (RAID 5/6)
- **RAID 5 write**: Read old data + read old parity + write new data + write new parity = **4 I/O ops per write**
- **RAID 6**: 6 I/O ops per write (two parity calculations)
- **RAID 10**: 2 I/O ops per write (mirror only)

```bash
# Linux software RAID (mdadm)
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd{a,b,c,d}1
mdadm --detail /dev/md0
cat /proc/mdstat

# Add spare
mdadm --add /dev/md0 /dev/sde1

# Remove failed disk
mdadm --fail /dev/md0 /dev/sda1
mdadm --remove /dev/md0 /dev/sda1
```

---

## Multipath I/O

```
┌───────────────────────────────────────────────────────────┐
│  Multipath I/O                                            │
│                                                           │
│  Server                                                   │
│  ┌──────────┐                                            │
│  │ multipath│                                            │
│  │ device   │                                            │
│  │ /dev/dm-0│                                            │
│  └──┬───┬───┘                                            │
│     │   │                                                │
│     ▼   ▼                                                │
│  Path A  Path B     (via different HBAs/switches)        │
│     │       │                                            │
│     ▼       ▼                                            │
│  ┌──────────────┐                                        │
│  │ Storage Array │                                       │
│  │ (same LUN)    │                                       │
│  └──────────────┘                                        │
│                                                           │
│  Policies:                                                │
│  - round-robin: alternate between paths                  │
│  - queue-length: prefer path with shorter queue          │
│  - service-time: prefer path with faster response        │
│                                                           │
│  Benefits:                                                │
│  - Redundancy: survive path/switch/HBA failure           │
│  - Performance: aggregate bandwidth                      │
└───────────────────────────────────────────────────────────┘
```

```bash
# Multipath commands
multipath -ll                  # Show multipath topology
multipathd show maps           # Show maps
multipathd show paths          # Show path status
```

---

## Page Cache and Dirty Writeback

### Page Cache

```
┌───────────────────────────────────────────────────────────┐
│  Page Cache                                               │
│                                                           │
│  write() → page cache (dirty page)                       │
│                                                           │
│  read() → check page cache first                         │
│           Hit: return from RAM (fast)                    │
│           Miss: read from disk, cache the page          │
│                                                           │
│  Unified cache: file data + metadata                     │
│  Indexed by: (inode, offset) → page                      │
│  Eviction: LRU (Least Recently Used) based               │
└───────────────────────────────────────────────────────────┘
```

### Dirty Page Writeback

```bash
# Key tuning parameters
sysctl vm.dirty_ratio              # % of RAM before writer blocks (default: 20)
sysctl vm.dirty_background_ratio   # % of RAM before async writeback starts (default: 10)
sysctl vm.dirty_expire_centisecs   # Age of dirty page before writeback (default: 3000 = 30s)
sysctl vm.dirty_writeback_centisecs # Writeback thread interval (default: 500 = 5s)

# For bytes-based control (overrides ratio if set)
sysctl vm.dirty_bytes
sysctl vm.dirty_background_bytes

# Flush cache
sync                               # Flush all dirty pages to disk
echo 1 > /proc/sys/vm/drop_caches  # Free pagecache
echo 2 > /proc/sys/vm/drop_caches  # Free dentries and inodes
echo 3 > /proc/sys/vm/drop_caches  # Free pagecache + dentries + inodes
```

### Write Barriers
- Ensure ordering of writes for crash consistency
- Filesystem flushes disk write cache before and after critical writes
- Critical for journal commits, superblock updates
- Disk must honor flush/FUA (Force Unit Access) commands

---

## Storage Protocols

| Protocol | Transport | Latency | Max Bandwidth | Use Case |
|---|---|---|---|---|
| **SATA** | Serial ATA | ~ms | 600 MB/s (SATA3) | Consumer drives |
| **SAS** | Serial Attached SCSI | ~ms | 1.2/2.4 GB/s | Enterprise HDDs/SSDs |
| **NVMe** | PCIe direct | ~10μs | 7-14 GB/s (PCIe 4/5) | High-performance SSDs |
| **iSCSI** | TCP/IP | ~ms | Network speed | SAN over Ethernet |
| **Fibre Channel** | FC fabric | ~μs-ms | 32/64/128 Gbps | Enterprise SAN |
| **NVMe-oF (TCP)** | TCP/IP | ~tens μs | Network speed | Disaggregated NVMe |
| **NVMe-oF (RDMA)** | RDMA (RoCE/IB) | ~μs | 100+ Gbps | Ultra-low latency |
| **NVMe-oF (FC)** | FC-NVMe | ~μs | FC speed | FC + NVMe combined |

### iSCSI

```
┌───────────────────────────────────────────────────────────┐
│  iSCSI                                                     │
│                                                           │
│  Initiator (client)  ──TCP/IP──►  Target (storage)       │
│  /dev/sdX                         LUN (Logical Unit)     │
│                                                           │
│  Encapsulates SCSI commands over TCP                     │
│  Port: 3260                                               │
│  Authentication: CHAP                                     │
│  Discovery: SendTargets, iSNS                            │
│                                                           │
│  Tools: iscsiadm, targetcli (target), open-iscsi (init) │
└───────────────────────────────────────────────────────────┘
```

```bash
# iSCSI initiator
iscsiadm -m discovery -t sendtargets -p <target_ip>:3260
iscsiadm -m node --login
iscsiadm -m session -P 3     # Show sessions with details
```

---

## Monitoring Storage I/O

```bash
# === iostat — Device-Level I/O Statistics ===
iostat -xz 1
# Device     r/s   w/s   rkB/s   wkB/s  await  %util
# nvme0n1   5000  3000  100000   60000   0.5    40%

# Key metrics:
# r/s, w/s:       IOPS (reads/writes per second)
# rkB/s, wkB/s:   Throughput
# await:           Average I/O latency (ms)
# r_await, w_await: Read/write latency separately
# %util:           Device utilization (100% = saturated, for HDD only)
#                  For NVMe/SSD, %util is misleading (parallel queues)

# === iotop — Per-Process I/O ===
iotop -oa                      # Accumulated I/O per process

# === Queue Depth ===
cat /sys/block/nvme0n1/queue/nr_requests    # Max queue depth
cat /sys/block/nvme0n1/inflight             # Currently in-flight

# === blktrace — Detailed Block I/O Tracing ===
blktrace -d /dev/sda -o trace -w 10        # Trace for 10 seconds
blkparse -i trace
# Fields: timestamp CPU seq device action sector+count [process]
```

---

## Interview Tips

1. **Describe the Linux I/O stack from write() to disk.**
   → write() → VFS → filesystem → page cache (dirty page) → block layer (bio) → I/O scheduler → blk-mq → device driver → hardware.

2. **Why NVMe is faster than SCSI/SAS?**
   → Direct PCIe (no controller overhead), 64K parallel queues, minimal protocol overhead (8 commands vs SCSI CDB complexity), per-CPU queues map to blk-mq.

3. **RAID 5 write penalty?**
   → 4 I/O ops: read old data + read old parity + write new data + write new parity. RAID 10 is only 2 ops (write data + write mirror).

4. **What is blk-mq and why does it matter?**
   → Per-CPU software queues mapped to hardware queues. Eliminates lock contention. Essential for NVMe (millions of IOPS).

5. **Page cache vs O_DIRECT?**
   → Page cache buffers I/O in RAM (transparent to app). O_DIRECT bypasses cache (app manages its own buffer pool). Databases use O_DIRECT.

6. **LVM thin provisioning?**
   → Allocate virtual capacity larger than physical. Space allocated on write (like VM overcommit). Monitor usage to avoid running out.

---

*Last updated: March 13, 2026*
