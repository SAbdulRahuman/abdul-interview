# I/O Models & System Calls

> Five I/O models, select/poll/epoll, io_uring, file I/O syscalls, direct I/O, and zero-copy.

---

## Five I/O Models

```
┌──────────────────────────────────────────────────────────────────┐
│  Linux I/O Models                                                │
│                                                                  │
│  1. BLOCKING I/O (default)                                       │
│     read() blocks until data is available                       │
│     Simple but poor scalability (1 thread per connection)       │
│                                                                  │
│  2. NON-BLOCKING I/O                                             │
│     read() returns EAGAIN if no data; application polls         │
│     Wastes CPU in polling loop                                  │
│                                                                  │
│  3. I/O MULTIPLEXING (select/poll/epoll)                        │
│     Monitor multiple FDs; block until any is ready              │
│     ★ Most common for network servers                          │
│                                                                  │
│  4. SIGNAL-DRIVEN I/O (SIGIO)                                   │
│     Kernel sends signal when FD is ready                        │
│     Rarely used (signal handling is complex)                    │
│                                                                  │
│  5. ASYNCHRONOUS I/O (AIO / io_uring)                           │
│     Submit I/O requests; kernel notifies on completion          │
│     ★ Best for high-performance storage (io_uring)             │
└──────────────────────────────────────────────────────────────────┘
```

### Comparison

| Model | Blocking? | Scalability | CPU Efficiency | Complexity |
|---|---|---|---|---|
| Blocking | Yes | Poor (1 thread/conn) | Good (sleeps) | Low |
| Non-blocking | No | Poor (polling) | Poor (busy wait) | Medium |
| Multiplexing | Partially | Good | Good | Medium |
| Signal-driven | No | Moderate | Good | High |
| Async (io_uring) | No | Excellent | Excellent | High |

---

## I/O Multiplexing Evolution

### select()

```c
#include <sys/select.h>

fd_set readfds;
FD_ZERO(&readfds);
FD_SET(sockfd, &readfds);

struct timeval tv = {.tv_sec = 5, .tv_usec = 0};
int nready = select(sockfd + 1, &readfds, NULL, NULL, &tv);

if (nready > 0 && FD_ISSET(sockfd, &readfds)) {
    // sockfd is ready for reading
    read(sockfd, buf, sizeof(buf));
}
```

**Limitations:**
- Max FD: `FD_SETSIZE` = 1024 (compile-time limit)
- O(n) scan per call — kernel checks all FDs in the set
- Must rebuild fd_set before each call (destructive)
- **Portable** across Unix systems

### poll()

```c
#include <poll.h>

struct pollfd fds[MAX_CLIENTS];
fds[0].fd = sockfd;
fds[0].events = POLLIN;

int nready = poll(fds, nfds, timeout_ms);

for (int i = 0; i < nfds; i++) {
    if (fds[i].revents & POLLIN) {
        // fds[i].fd is ready for reading
    }
    if (fds[i].revents & POLLERR) {
        // Error on this FD
    }
}
```

**Improvements over select:**
- No FD_SETSIZE limit (array-based, arbitrary size)
- Events and results in same struct (no rebuild needed)
- Still O(n) per call

### epoll — The Standard for High-Performance I/O

```c
#include <sys/epoll.h>

// Create epoll instance
int epfd = epoll_create1(0);

// Add FD to watch list
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;  // Edge-triggered read
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// Event loop
struct epoll_event events[MAX_EVENTS];
while (1) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout_ms);
    for (int i = 0; i < nfds; i++) {
        if (events[i].events & EPOLLIN) {
            handle_read(events[i].data.fd);
        }
        if (events[i].events & EPOLLOUT) {
            handle_write(events[i].data.fd);
        }
        if (events[i].events & (EPOLLERR | EPOLLHUP)) {
            handle_error(events[i].data.fd);
        }
    }
}

// Modify watched events
ev.events = EPOLLIN | EPOLLOUT | EPOLLET;
epoll_ctl(epfd, EPOLL_CTL_MOD, sockfd, &ev);

// Remove FD
epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);
```

### Comparison Table

| Feature | select() | poll() | epoll |
|---|---|---|---|
| Max FDs | 1024 (FD_SETSIZE) | Unlimited | Unlimited |
| Performance | O(n) per call | O(n) per call | O(1) per event |
| FD state | Rebuild each call | Persistent | Persistent (kernel) |
| Mechanism | Bitmap scan | Array scan | Callback-based |
| Linux-specific | No (POSIX) | No (POSIX) | Yes |
| Memory | Stack (bitmap) | Stack (array) | Kernel (red-black tree) |

### Edge-Triggered vs Level-Triggered

```
Level-Triggered (default):
  Data arrives → epoll_wait returns
  If you don't read all data → epoll_wait returns AGAIN
  (Like poll — FD stays "ready" as long as data exists)

Edge-Triggered (EPOLLET):
  Data arrives → epoll_wait returns ONCE
  If you don't read all data → epoll_wait does NOT return again
  Must read until EAGAIN to drain the buffer

  Rules for edge-triggered:
  1. Use non-blocking FDs
  2. Read/write in a loop until EAGAIN
  3. Miss a drain → lose events until NEW data arrives
```

**Why edge-triggered?**
- Avoids redundant wake-ups (better performance with many connections)
- Forces proper "drain" pattern (read all available data)
- Used by high-performance servers (Nginx, HAProxy)

---

## io_uring — Modern Async I/O (Linux 5.1+)

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  io_uring Architecture                                           │
│                                                                  │
│  User Space          │  Kernel Space                             │
│                      │                                           │
│  ┌──────────────┐    │  ┌──────────────┐                        │
│  │ Submission    │────┼─►│ Submission   │                        │
│  │ Queue (SQ)    │    │  │ Processing   │                        │
│  └──────────────┘    │  └──────┬───────┘                        │
│                      │         │                                 │
│  ┌──────────────┐    │  ┌──────▼───────┐                        │
│  │ Completion   │◄───┼──│ Completion   │                        │
│  │ Queue (CQ)    │    │  │ Results      │                        │
│  └──────────────┘    │  └──────────────┘                        │
│                      │                                           │
│  Key: Shared ring buffers in mmap'd memory                      │
│  → ZERO syscall overhead for batching multiple I/O ops          │
└──────────────────────────────────────────────────────────────────┘
```

### How io_uring Works

1. **Setup**: `io_uring_setup()` creates SQ and CQ ring buffers (mmap'd shared memory)
2. **Submit**: Application writes SQEs (Submission Queue Entries) to SQ ring
3. **Notify kernel**: `io_uring_enter()` or kernel polls automatically (`IORING_SETUP_SQPOLL`)
4. **Completion**: Kernel writes CQEs (Completion Queue Entries) to CQ ring
5. **Consume**: Application reads CQEs from CQ ring

### Key Advantages
- **Zero-copy**: Shared memory rings, no copy between user/kernel
- **Batching**: Submit multiple I/O requests in one syscall (or zero syscalls with SQ polling)
- **Linked operations**: Chain dependent operations
- **Fixed buffers**: Pre-register buffers to avoid per-I/O address translation
- **SQ polling mode**: Kernel thread polls SQ — true zero-syscall I/O

### Supported Operations
- File I/O: `IORING_OP_READ`, `IORING_OP_WRITE`, `IORING_OP_READV`, `IORING_OP_WRITEV`
- File management: `IORING_OP_FSYNC`, `IORING_OP_FALLOCATE`, `IORING_OP_OPENAT`
- Network: `IORING_OP_ACCEPT`, `IORING_OP_CONNECT`, `IORING_OP_SEND`, `IORING_OP_RECV`
- Polling: `IORING_OP_POLL_ADD`

### Using liburing (Helper Library)

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(256, &ring, 0);  // 256 entries

// Submit a read
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, size, offset);
io_uring_sqe_set_data(sqe, user_data);  // Attach context
io_uring_submit(&ring);

// Wait for completion
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);

int result = cqe->res;           // Bytes read or error
void *data = io_uring_cqe_get_data(cqe);  // Get context back
io_uring_cqe_seen(&ring, cqe);  // Mark consumed

io_uring_queue_exit(&ring);
```

### Performance Impact
- Database benchmarks show **2-5x IOPS improvement** vs traditional AIO
- Reduced syscall overhead: critical for NVMe (sub-microsecond I/O)
- Used by: high-performance databases, storage engines, proxy servers

---

## Key File I/O System Calls

### Basic I/O

```c
#include <fcntl.h>
#include <unistd.h>

// Open file
int fd = open("/path/to/file", O_RDWR | O_CREAT, 0644);
int fd = open("/path/to/file", O_RDONLY);

// Sequential read/write (advances file position)
ssize_t n = read(fd, buf, sizeof(buf));
ssize_t n = write(fd, buf, len);

// Positional read/write (thread-safe, no seek needed)
ssize_t n = pread(fd, buf, sizeof(buf), offset);
ssize_t n = pwrite(fd, buf, len, offset);

// Seek
off_t pos = lseek(fd, 0, SEEK_CUR);  // Get current position
lseek(fd, 100, SEEK_SET);            // Go to byte 100
lseek(fd, 0, SEEK_END);              // Go to end of file

close(fd);
```

### Scatter-Gather I/O (Vectored I/O)

```c
#include <sys/uio.h>

// Write multiple buffers in one syscall
struct iovec iov[3];
iov[0].iov_base = header;  iov[0].iov_len = header_len;
iov[1].iov_base = data;    iov[1].iov_len = data_len;
iov[2].iov_base = trailer; iov[2].iov_len = trailer_len;

ssize_t n = writev(fd, iov, 3);   // Atomic write of all 3 buffers
ssize_t n = readv(fd, iov, 3);    // Read into multiple buffers
```

**Benefits:** Reduces syscall count, atomic operation for related data

### System Call Reference Table

| Syscall | Purpose | Notes |
|---|---|---|
| `open()` | Open/create file | Returns file descriptor |
| `read()` / `write()` | Sequential I/O | Advances file position |
| `pread()` / `pwrite()` | Positional I/O | Thread-safe, no seek needed |
| `readv()` / `writev()` | Scatter-gather I/O | Multiple buffers in one call |
| `sendfile()` | Zero-copy file → socket | Avoids user-space copy |
| `splice()` | Zero-copy pipe transfer | Between two file descriptors |
| `mmap()` | Memory-mapped I/O | File mapped into address space |
| `fsync()` | Flush file data + metadata to disk | Ensures durability |
| `fdatasync()` | Flush file data only | Faster than fsync (no metadata) |
| `fallocate()` | Pre-allocate file space | Avoids fragmentation |
| `fadvise()` | Hint page cache behavior | `POSIX_FADV_SEQUENTIAL`, `DONTNEED` |
| `ioctl()` | Device-specific operations | Control device behavior |
| `fcntl()` | File control operations | Locks, flags, duplication |

---

## Zero-Copy I/O

### Traditional Copy (4 copies, 4 context switches)

```
read(file_fd, buf, len):
  1. Disk → Kernel buffer (DMA)
  2. Kernel buffer → User buffer (CPU copy)

write(socket_fd, buf, len):
  3. User buffer → Kernel socket buffer (CPU copy)
  4. Kernel socket buffer → NIC (DMA)
```

### sendfile() — Zero-Copy (2 copies, 2 context switches)

```c
#include <sys/sendfile.h>

// Send file data directly to socket (no user-space copy)
off_t offset = 0;
ssize_t sent = sendfile(socket_fd, file_fd, &offset, file_size);

// Data flow:
// 1. Disk → Kernel buffer (DMA)
// 2. Kernel buffer → NIC (DMA, with sendfile + scatter-gather)
// Skips user-space entirely!
```

### splice() — Zero-Copy via Pipe

```c
#include <fcntl.h>

int pipefd[2];
pipe(pipefd);

// File → Pipe (zero-copy into pipe buffer)
splice(file_fd, &offset, pipefd[1], NULL, len, SPLICE_F_MOVE);

// Pipe → Socket (zero-copy from pipe buffer)
splice(pipefd[0], NULL, socket_fd, NULL, len, SPLICE_F_MOVE);
```

---

## Durability: fsync, fdatasync, O_SYNC

### The Write Path

```
write()
  │
  ▼
User buffer → Kernel page cache (dirty page)
                    │
                    │ (asynchronous writeback)
                    ▼
              Disk controller cache
                    │
                    ▼
              Persistent storage (disk platter / NAND)
```

### Guaranteeing Durability

| Mechanism | What It Flushes | When to Use |
|---|---|---|
| `write()` | To page cache only | Default (no durability guarantee) |
| `fsync(fd)` | Data + metadata to disk | After critical writes (commit point) |
| `fdatasync(fd)` | Data + essential metadata | When file size hasn't changed |
| `O_SYNC` | Each write waits for disk | Every write must be durable |
| `O_DSYNC` | Each write flushes data | Each write durable (metadata lazy) |
| `sync_file_range()` | Flush range of data | Fine-grained control |

```c
// Pattern: batch writes then fsync
for (int i = 0; i < 1000; i++) {
    write(fd, data, len);
}
fsync(fd);  // One fsync after batch — MUCH faster than O_SYNC

// fdatasync — skip metadata update if file size unchanged
write(fd, data, len);  // Overwrite existing data
fdatasync(fd);         // Only flush data blocks (faster)
```

---

## Pre-Allocation and Hints

### fallocate() — Pre-Allocate Space

```c
#include <fcntl.h>

// Pre-allocate 1GB of contiguous disk space
fallocate(fd, 0, 0, 1024 * 1024 * 1024);

// Punch hole (deallocate range, keep file size)
fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
          offset, length);

// Zero range (zero-fill without writing zeros)
fallocate(fd, FALLOC_FL_ZERO_RANGE, offset, length);
```

**Benefits:** Reduces fragmentation, avoids ENOSPC during writes, faster than sequential zero-fill.

### fadvise() — Page Cache Hints

```c
#include <fcntl.h>

// Hint: will access sequentially (enable readahead)
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

// Hint: random access (disable readahead)
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);

// Hint: won't need this data anymore (free page cache)
posix_fadvise(fd, 0, file_size, POSIX_FADV_DONTNEED);

// Hint: will need soon (prefetch into page cache)
posix_fadvise(fd, offset, length, POSIX_FADV_WILLNEED);
```

---

## Direct I/O

### Bypassing the Page Cache

```c
// O_DIRECT: bypass page cache — data goes directly to/from disk
int fd = open("/dev/sdb", O_RDWR | O_DIRECT);

// ALIGNMENT REQUIREMENTS:
// - Buffer must be aligned (512-byte or 4K, depends on device)
// - Offset must be aligned
// - Size must be aligned
void *buf;
posix_memalign(&buf, 4096, 4096);   // 4K-aligned buffer
pread(fd, buf, 4096, 0);             // Read at offset 0
pwrite(fd, buf, 4096, 4096);         // Write at offset 4096
free(buf);
```

### Open Flags for I/O Behavior

| Flag | Behavior | Use Case |
|---|---|---|
| `O_DIRECT` | Bypass page cache | Databases, storage engines (self-managed cache) |
| `O_SYNC` | Synchronous writes (data + metadata) | Write-ahead log, critical data |
| `O_DSYNC` | Synchronous data writes (metadata lazy) | Data durability without metadata overhead |
| `O_NOATIME` | Don't update access time | Read-heavy workloads (reduce metadata writes) |
| `O_NONBLOCK` | Non-blocking I/O | Network sockets, pipes |
| `O_APPEND` | Atomic append | Log files (concurrent writers) |
| `O_CLOEXEC` | Close on exec | Security (don't leak FDs to child processes) |

### Why Databases Use O_DIRECT
- Databases manage their own buffer pool (double-caching is wasteful)
- Predictable I/O latency (no page cache eviction surprises)
- Better control over write ordering (WAL protocol)
- Examples: PostgreSQL, MySQL InnoDB, RocksDB

---

## ioctl() and fcntl()

### ioctl() — Device Control

```c
#include <sys/ioctl.h>

// Get terminal window size
struct winsize ws;
ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws);
printf("Rows: %d, Cols: %d\n", ws.ws_row, ws.ws_col);

// Get block device size
unsigned long long size;
ioctl(fd, BLKGETSIZE64, &size);  // Size in bytes

// SCSI/NVMe device commands (via sg/nvme ioctl)
// Used in storage development for direct device communication
```

### fcntl() — File Control

```c
#include <fcntl.h>

// Set non-blocking mode
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// File locking (advisory)
struct flock fl;
fl.l_type = F_WRLCK;     // Write lock
fl.l_whence = SEEK_SET;
fl.l_start = 0;
fl.l_len = 0;            // Lock entire file
fcntl(fd, F_SETLK, &fl);   // Non-blocking
fcntl(fd, F_SETLKW, &fl);  // Blocking (wait for lock)

// Duplicate file descriptor
int newfd = fcntl(fd, F_DUPFD, 0);      // Like dup()
int newfd = fcntl(fd, F_DUPFD_CLOEXEC, 0);  // With close-on-exec
```

---

## Interview Tips

1. **Compare select, poll, epoll.**
   → select: 1024 FD limit, O(n). poll: no limit, still O(n). epoll: O(1) per event, callback-based.

2. **Edge-triggered vs level-triggered?**
   → LT: returns as long as FD is ready. ET: returns only on state change, must drain all data.

3. **What is io_uring and why is it significant?**
   → Shared ring buffers for async I/O. Zero-copy, zero-syscall batching. 2-5x IOPS vs AIO.

4. **fsync vs fdatasync?**
   → fsync flushes data + all metadata. fdatasync flushes data + essential metadata (file size). fdatasync is faster when size doesn't change.

5. **Why use O_DIRECT?**
   → Bypass page cache. Databases manage their own cache. Avoids double-caching, predictable latency.

6. **What is sendfile()?**
   → Zero-copy: sends file data to socket without copying through user space. Used by web servers (Nginx).

---

*Last updated: March 13, 2026*
