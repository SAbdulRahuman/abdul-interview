# File Systems

> VFS layer, inodes, journaling, B-trees, extents, copy-on-write, and filesystem comparisons.

---

## VFS (Virtual File System) Layer

```
┌──────────────────────────────────────────────────────────────────┐
│  Linux Storage Stack                                             │
│                                                                  │
│  Application                                                     │
│      │                                                           │
│      ▼                                                           │
│  System Calls (open, read, write, close, ioctl)                 │
│      │                                                           │
│      ▼                                                           │
│  VFS Layer (Virtual File System)                                │
│      │           │            │            │                     │
│      ▼           ▼            ▼            ▼                     │
│    ext4        XFS         Btrfs      NFS/CIFS (network)        │
│      │           │            │                                  │
│      ▼           ▼            ▼                                  │
│  Page Cache (buffer cache unified since 2.4)                    │
│      │                                                           │
│      ▼                                                           │
│  Block Layer (I/O scheduler, merging, plug/unplug)              │
│      │                                                           │
│      ▼                                                           │
│  Device Drivers (SCSI, NVMe, virtio-blk)                       │
│      │                                                           │
│      ▼                                                           │
│  Hardware (HDD, SSD, NVMe)                                      │
└──────────────────────────────────────────────────────────────────┘
```

### VFS Design
- Provides a **uniform interface** for all file systems (local and network)
- Applications use the same system calls regardless of underlying filesystem
- Each filesystem implements a set of **operations** (inode_operations, file_operations, super_operations)
- VFS maintains caches (dentry cache, inode cache) for fast path lookups

---

## Key VFS Objects

| Object | Purpose | Key Fields |
|---|---|---|
| **Superblock** | Represents a mounted filesystem | Block size, inode count, free blocks, fs type |
| **Inode** | Metadata for a file/directory | Permissions, size, timestamps, data block pointers |
| **Dentry** | Directory entry (name ↔ inode mapping) | Cached in dcache for fast lookups |
| **File** | Open file instance (per open() call) | File position, access mode, pointer to inode |

### Relationships

```
Superblock (one per mounted FS)
  │
  ├── Inode (one per file/directory)
  │     │
  │     ├── Dentry (one per path component)
  │     │     │
  │     │     └── Name → Inode mapping
  │     │
  │     └── Data blocks (actual file content)
  │
  └── Inode ...

File (one per open() call)
  │
  ├── f_pos (current read/write position)
  ├── f_mode (read/write/append flags)
  └── f_inode → points to Inode
```

---

## Inodes

### What's in an Inode?

| Field | Description |
|---|---|
| Mode | File type (regular, dir, symlink, etc.) + permissions |
| Owner | UID, GID |
| Size | File size in bytes |
| Timestamps | atime (access), mtime (modify), ctime (metadata change) |
| Link count | Number of hard links (0 = file deleted when last reference closed) |
| Block pointers | Location of actual data blocks on disk |
| Extended attributes | ACLs, SELinux labels, etc. |

### Inode Number

```bash
# View inode number
ls -i file.txt
# 1234567 file.txt

stat file.txt
# File: file.txt
# Size: 1024       Blocks: 8          IO Block: 4096   regular file
# Inode: 1234567   Links: 1

# Check inode usage
df -i /
# Filesystem     Inodes   IUsed   IFree  IUse% Mounted on
# /dev/sda1     6553000  250000 6303000    4%  /
```

### Running Out of Inodes
- Each filesystem has a **fixed number of inodes** (set at format time for ext4)
- You can run out of inodes before running out of disk space (many tiny files)
- `df -i` shows inode usage
- Fix: Reformat with more inodes (`mkfs.ext4 -N <count>`) or use XFS (dynamic inodes)

### Data Block Addressing

```
┌─────────────────────────────────────────────────────────────┐
│  Traditional (ext3, pre-extent): Indirect Block Mapping     │
│                                                             │
│  Inode:                                                     │
│  ├── Direct blocks (12)     → 48 KB                        │
│  ├── Indirect (1)           → 4 MB (1024 pointers × 4K)   │
│  ├── Double indirect (1)    → 4 GB                         │
│  └── Triple indirect (1)    → 4 TB                         │
│                                                             │
│  Problem: Large files need many indirect lookups            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Modern (ext4, XFS): Extent-Based Mapping                   │
│                                                             │
│  Extent = (start_block, length)                             │
│                                                             │
│  Inode:                                                     │
│  ├── Extent 1: blocks 1000-1999 (1000 blocks contiguous)  │
│  ├── Extent 2: blocks 5000-5499 (500 blocks contiguous)   │
│  └── Extent 3: blocks 8000-8099 (100 blocks contiguous)   │
│                                                             │
│  Advantage: One extent can describe millions of             │
│  contiguous blocks — much more efficient                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Hard Links vs Symbolic Links

| Feature | Hard Link | Symbolic (Soft) Link |
|---|---|---|
| Points to | Same inode | Path string |
| Cross filesystem | No | Yes |
| Directory link | No (except . and ..) | Yes |
| Target deleted | File still accessible | Dangling (broken) link |
| Inode | Same inode number | Different inode |
| Space | Only dentry entry | Inode + path string |

```bash
# Hard link
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt
# Same inode number, link count = 2

# Symbolic link
ln -s original.txt symlink.txt
ls -li symlink.txt
# Different inode, shows target path

# Removing original:
# Hard link: file still accessible via hardlink.txt
# Symlink: symlink.txt becomes broken (dangling)
```

---

## Journaling

### Why Journaling?
Without journaling, a crash during a write can leave the filesystem in an **inconsistent state** (e.g., allocated blocks without inode, inode pointing to wrong blocks).

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│  Write with Journaling (ext4 ordered mode)                   │
│                                                              │
│  1. Write data blocks to their final location               │
│  2. Write metadata changes to journal (write-ahead log)     │
│  3. Commit journal transaction (fsync journal)              │
│  4. Write metadata to their final location                  │
│  5. Mark journal transaction as complete                    │
│                                                              │
│  On crash recovery:                                          │
│  - Replay committed journal entries                         │
│  - Discard uncommitted entries                              │
│  - Filesystem is consistent (maybe lose uncommitted data)   │
└──────────────────────────────────────────────────────────────┘
```

### Journal Modes (ext4)

| Mode | Journals | Performance | Safety |
|---|---|---|---|
| **journal** | Metadata + data | Slowest (double write) | Safest |
| **ordered** (default) | Metadata only (data written first) | Balanced | Good |
| **writeback** | Metadata only (data order not guaranteed) | Fastest | Risk of stale data |

```bash
# Check current journal mode
tune2fs -l /dev/sda1 | grep "Journal"

# Mount with specific mode
mount -o data=ordered /dev/sda1 /mnt
```

### XFS Log
- Uses a **log** (similar to journal) for metadata
- Log can be on a **separate device** for better performance
- Supports **delayed allocation** — allocates blocks at flush time for better contiguity

---

## B-Tree / B+Tree in File Systems

```
┌──────────────────────────────────────────────────────────────┐
│  B+Tree (used by XFS, Btrfs)                                │
│                                                              │
│  Properties:                                                │
│  - Balanced: all leaves at same depth                       │
│  - High fan-out: minimizes tree depth (disk seeks)          │
│  - Leaf nodes linked: efficient range scans                 │
│                                                              │
│              ┌─────────────┐                                 │
│              │  [50 | 100]  │  ← Internal node              │
│              └─┬───┬───┬───┘                                 │
│          ┌─────┘   │   └─────┐                               │
│          ▼         ▼         ▼                               │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐                  │
│  │10|20|30|40│ │55|60|70|80│ │110|120|150│  ← Leaf nodes    │
│  └─────┬─────┘ └─────┬─────┘ └───────────┘                  │
│        └──────────────┘  ← Leaves linked for range scan     │
└──────────────────────────────────────────────────────────────┘
```

### Usage in File Systems

| File System | B-tree Use |
|---|---|
| **XFS** | Inode allocation, free space, directory entries, extent map |
| **Btrfs** | Everything — metadata tree, extent allocation, checksum tree |
| **ext4** | Htree (hash-based B-tree variant) for directory indexing |

---

## Extents

### What Are Extents?
An extent describes a **contiguous range of blocks**: `(start_block, length)`.

Instead of tracking each block individually (indirect blocks), one extent can describe millions of contiguous blocks.

### Benefits
- **Reduced metadata**: One extent instead of thousands of block pointers
- **Better performance**: Contiguous blocks = sequential I/O
- **Less fragmentation**: Allocator tries to allocate contiguous extents

### ext4 Extent Tree

```
┌─────────────────────────────────────────────────────────────┐
│  ext4 Extent Tree (up to 3 levels)                          │
│                                                             │
│  Inode → Extent Header + up to 4 extents inline            │
│                                                             │
│  If more extents needed → extent tree (B-tree of extents)  │
│                                                             │
│  Each extent: ┌─────────────────────────────────┐           │
│               │ Logical block start             │           │
│               │ Physical block start            │           │
│               │ Length (up to 128MB per extent)  │           │
│               └─────────────────────────────────┘           │
│                                                             │
│  Maximum file size: 16 TB (4K blocks) to 256 TB            │
└─────────────────────────────────────────────────────────────┘
```

---

## Copy-on-Write (COW) File Systems

### How COW Works

```
Traditional (overwrite in place):
  Block 5: [old data] → [new data]  (overwrite)
  Problem: crash during overwrite → corrupted data

COW (never overwrite):
  1. Write new data to NEW block (Block 99)
  2. Update metadata to point to Block 99
  3. Free old Block 5

  Block 5:  [old data] → freed (after metadata update)
  Block 99: [new data] → new location
  
  Crash safety: old data intact until metadata committed
```

### Benefits of COW
- **Crash consistency** without journal (atomic pointer swap)
- **Snapshots** are nearly free (just keep old pointers)
- **Checksums** on data + metadata (detect bit rot)
- **Efficient clones** — share blocks until modified

### Downsides
- **Write amplification** — every write needs new block + metadata update
- **Fragmentation** — data spreads across disk over time
- Not good for databases that do many random small writes (workaround: use `nodatacow` or `chattr +C`)

---

## Allocation Groups (XFS)

```
┌──────────────────────────────────────────────────────────────┐
│  XFS Allocation Groups                                       │
│                                                              │
│  Filesystem divided into equal-sized Allocation Groups (AGs)│
│                                                              │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                    │
│  │ AG 0 │  │ AG 1 │  │ AG 2 │  │ AG 3 │                    │
│  │      │  │      │  │      │  │      │                    │
│  │ Free │  │ Free │  │ Free │  │ Free │                    │
│  │ space│  │ space│  │ space│  │ space│                    │
│  │ tree │  │ tree │  │ tree │  │ tree │                    │
│  └──────┘  └──────┘  └──────┘  └──────┘                    │
│   Lock 0    Lock 1    Lock 2    Lock 3                      │
│                                                              │
│  Each AG has its own:                                        │
│  - Free space B+tree                                        │
│  - Inode allocation B+tree                                  │
│  - Independent lock                                         │
│                                                              │
│  Benefit: Multiple threads can allocate in different AGs    │
│  concurrently → excellent parallel I/O performance          │
└──────────────────────────────────────────────────────────────┘
```

---

## File System Comparison

### Common Linux File Systems

| File System | Type | Max File | Max FS | Journaling | COW | Checksums | Snapshots |
|---|---|---|---|---|---|---|---|
| **ext4** | Traditional | 16 TB | 1 EB | Yes | No | Metadata only | No |
| **XFS** | Traditional | 8 EB | 8 EB | Yes (log) | No | Metadata (v5) | No |
| **Btrfs** | COW | 16 EB | 16 EB | No (COW) | Yes | Data + metadata | Yes |
| **ZFS** | COW | 16 EB | 256 ZB | No (COW) | Yes | Data + metadata | Yes |

### Detailed Comparison

#### ext4
- **Strengths**: Mature, reliable, well-tested, good all-round performance
- **Features**: Extents, delayed allocation, journal, online resize, online defrag
- **Weaknesses**: No checksums on data, no snapshots, fixed inode count at format
- **Use case**: Default Linux FS, general-purpose workloads, boot partitions

#### XFS
- **Strengths**: Excellent for large files, parallel I/O, scales to very large filesystems
- **Features**: Allocation groups, B+tree everything, delayed allocation, online defrag, reflink (copy-on-write clones)
- **Weaknesses**: Cannot shrink filesystem, historically no undelete
- **Use case**: Storage servers, media, databases, large file workloads, Red Hat/CentOS default

#### Btrfs
- **Strengths**: COW, snapshots, checksums, compression, RAID built-in, send/receive
- **Features**: Subvolumes, transparent compression (zstd/lzo/zlib), dedup, scrub
- **Weaknesses**: RAID 5/6 still fragile, fragmentation with random writes
- **Use case**: NAS, storage appliances, containers (snapshot-based layers)

#### ZFS (OpenZFS)
- **Strengths**: End-to-end checksums, RAID-Z (1/2/3), dedup, compression, send/receive
- **Features**: Zpools, datasets, arc cache, l2arc, slog, inline dedup
- **Weaknesses**: Not in mainline kernel (license), memory-hungry (ARC cache)
- **Use case**: Enterprise NAS, backup servers, data integrity critical systems

### Special-Purpose File Systems

| File System | Purpose | Backed By |
|---|---|---|
| **tmpfs** | RAM-based volatile storage | RAM + swap |
| **procfs** | Process and kernel info (`/proc`) | Kernel |
| **sysfs** | Device and driver info (`/sys`) | Kernel |
| **devtmpfs** | Device nodes (`/dev`) | Kernel |
| **overlayfs** | Union mount (container layers) | Underlying FS |
| **ceph** | Distributed (CephFS) | Network + Ceph |
| **NFS** | Network file system | Network |
| **CIFS/SMB** | Windows shared folders | Network |

---

## File System Operations

### Creating File Systems

```bash
# ext4
mkfs.ext4 -L mydata /dev/sdb1
mkfs.ext4 -N 1000000 /dev/sdb1     # Set inode count

# XFS
mkfs.xfs -L mydata /dev/sdb1
mkfs.xfs -f /dev/sdb1               # Force overwrite

# Btrfs
mkfs.btrfs -L mydata /dev/sdb1
mkfs.btrfs -d raid1 -m raid1 /dev/sdb1 /dev/sdc1  # RAID 1

# Mount
mount -t ext4 /dev/sdb1 /mnt/data
mount -o noatime,data=ordered /dev/sdb1 /mnt/data
```

### Common Mount Options

| Option | Description |
|---|---|
| `noatime` | Don't update access time (better performance) |
| `relatime` | Update atime only if older than mtime (default) |
| `sync` | Synchronous I/O (slow but safe) |
| `noexec` | Don't allow execution of binaries |
| `nosuid` | Ignore setuid/setgid bits |
| `data=ordered` | ext4: write data before metadata journal |
| `discard` | Enable TRIM for SSDs |
| `compress=zstd` | Btrfs: enable compression |

### File System Maintenance

```bash
# Check filesystem
fsck /dev/sdb1                      # Unmounted only!
xfs_repair /dev/sdb1

# Resize
resize2fs /dev/sdb1                 # ext4 (online grow)
xfs_growfs /mnt/data                # XFS (online grow, no shrink)

# Defragment
e4defrag /mnt/data                  # ext4
xfs_fsr /mnt/data                   # XFS
btrfs filesystem defragment /mnt/data  # Btrfs

# Check usage
df -hT                              # Disk usage with FS type
du -sh /mnt/data/*                  # Directory sizes

# Btrfs specific
btrfs subvolume snapshot /mnt/data /mnt/data/snap1
btrfs scrub start /mnt/data        # Verify checksums
btrfs balance start /mnt/data      # Rebalance data across devices
```

---

## Interview Tips

1. **What is VFS?**
   → Abstract layer providing uniform file ops interface. Each FS implements operations (inode_ops, file_ops). Caches dentries and inodes.

2. **What are inodes? What if you run out?**
   → Metadata structure per file. Fixed count in ext4 (set at format). Use `df -i`. XFS has dynamic inodes.

3. **Hard link vs symlink?**
   → Hard link: same inode, same FS only. Symlink: separate inode pointing to path, cross-FS, can dangle.

4. **How does journaling work?**
   → Write-ahead log. Write metadata changes to journal first, then to final location. Replay on crash.

5. **ext4 vs XFS for storage server?**
   → XFS: better for large files, parallel I/O (allocation groups), scales larger. ext4: better for mixed workloads, more mature fsck.

6. **What is COW and why use it?**
   → Copy-on-write: never overwrite data in place. Crash-safe, enables free snapshots, allows checksums. Tradeoff: fragmentation, write amplification.

---

*Last updated: March 13, 2026*
