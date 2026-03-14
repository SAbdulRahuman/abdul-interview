# IPC (Inter-Process Communication)

> Pipes, named pipes, Unix domain sockets, shared memory, message queues, semaphores, signals, and memory-mapped files.

---

## IPC Mechanisms Overview

| Mechanism | Speed | Scope | Persistence | Data Flow | Use Case |
|---|---|---|---|---|---|
| **Pipe** | Fast | Parent-child | Process lifetime | Unidirectional | Simple data flow |
| **Named Pipe (FIFO)** | Fast | Any process | Until deleted | Unidirectional | Unrelated process comm |
| **Unix Domain Socket** | Very fast | Same host | Process lifetime | Bidirectional | High-performance local IPC |
| **TCP/UDP Socket** | Network | Any host | Connection lifetime | Bidirectional | Distributed communication |
| **Shared Memory** | Fastest | Same host | Until removed | N/A (direct) | Bulk data sharing |
| **Message Queue** | Fast | Same host | Until removed | Structured | Structured messages |
| **Semaphore** | N/A (sync) | Same host | Until removed | N/A | Synchronization |
| **Signal** | Fast | Same host | Instant | Notification | Notifications |
| **Memory-Mapped File** | Very fast | Any process | File lifetime | N/A (direct) | Shared state + persistence |

---

## Pipes

### Anonymous Pipe

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    int pipefd[2];  // pipefd[0] = read end, pipefd[1] = write end
    pipe(pipefd);

    pid_t pid = fork();
    if (pid == 0) {
        // Child: reads from pipe
        close(pipefd[1]);  // Close write end
        char buf[256];
        ssize_t n = read(pipefd[0], buf, sizeof(buf));
        buf[n] = '\0';
        printf("Child received: %s\n", buf);
        close(pipefd[0]);
        _exit(0);
    } else {
        // Parent: writes to pipe
        close(pipefd[0]);  // Close read end
        const char *msg = "Hello from parent";
        write(pipefd[1], msg, strlen(msg));
        close(pipefd[1]);
        wait(NULL);
    }
    return 0;
}
```

### Key Properties
- **Unidirectional**: One reader, one writer (use two pipes for bidirectional)
- **Byte stream**: No message boundaries
- **Blocking**: read() blocks if empty, write() blocks if full (pipe buffer ~64KB)
- **SIGPIPE**: Sent to writer if reader closes its end
- **Parent-child only**: Created before fork(), shared via inheritance

### Shell Pipeline
```bash
# ls | grep | wc creates two pipes:
#   ls ──pipe1──► grep ──pipe2──► wc
# Each command runs in a separate process
```

---

## Named Pipes (FIFOs)

```c
#include <sys/stat.h>
#include <fcntl.h>

// Create FIFO
mkfifo("/tmp/myfifo", 0666);

// Writer process
int fd = open("/tmp/myfifo", O_WRONLY);  // Blocks until reader opens
write(fd, data, len);
close(fd);

// Reader process (separate program)
int fd = open("/tmp/myfifo", O_RDONLY);  // Blocks until writer opens
read(fd, buf, sizeof(buf));
close(fd);

// Cleanup
unlink("/tmp/myfifo");
```

```bash
# Shell usage
mkfifo /tmp/myfifo
echo "Hello" > /tmp/myfifo &    # Writer (blocks until reader)
cat /tmp/myfifo                  # Reader
```

### Named Pipe vs Anonymous Pipe

| Feature | Anonymous Pipe | Named Pipe (FIFO) |
|---|---|---|
| Scope | Parent-child only | Any process |
| Persistence | Process lifetime | Until unlink() |
| Filesystem | No path | Has path in filesystem |
| Creation | pipe() | mkfifo() |

---

## Unix Domain Sockets

### Stream Socket (like TCP)

```c
#include <sys/socket.h>
#include <sys/un.h>

// Server
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/my.sock", sizeof(addr.sun_path) - 1);
unlink(addr.sun_path);  // Remove old socket file
bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
listen(sockfd, 5);

int clientfd = accept(sockfd, NULL, NULL);
// Read/write on clientfd
read(clientfd, buf, sizeof(buf));
write(clientfd, response, len);
close(clientfd);
close(sockfd);
unlink(addr.sun_path);

// Client
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
connect(sockfd, (struct sockaddr *)&addr, sizeof(addr));
write(sockfd, request, len);
read(sockfd, buf, sizeof(buf));
close(sockfd);
```

### Datagram Socket (like UDP)

```c
// Server
int sockfd = socket(AF_UNIX, SOCK_DGRAM, 0);
bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
recvfrom(sockfd, buf, sizeof(buf), 0, &client_addr, &len);
sendto(sockfd, response, rlen, 0, &client_addr, len);
```

### Abstract Namespace (Linux-specific)

```c
// No filesystem entry — automatically cleaned up
addr.sun_path[0] = '\0';  // First byte is null
memcpy(addr.sun_path + 1, "my_abstract_socket", 18);
// No need to unlink()
```

### File Descriptor Passing (SCM_RIGHTS)

```c
// Send a file descriptor to another process via Unix socket
struct msghdr msg = {0};
struct iovec iov;
char buf[1] = {'F'};
iov.iov_base = buf;
iov.iov_len = 1;
msg.msg_iov = &iov;
msg.msg_iovlen = 1;

// Control message with the FD
char cmsgbuf[CMSG_SPACE(sizeof(int))];
msg.msg_control = cmsgbuf;
msg.msg_controllen = sizeof(cmsgbuf);

struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type = SCM_RIGHTS;
cmsg->cmsg_len = CMSG_LEN(sizeof(int));
*(int *)CMSG_DATA(cmsg) = fd_to_send;

sendmsg(sockfd, &msg, 0);
```

### Why Unix Domain Sockets?
- **Faster than TCP loopback**: No network stack overhead (no checksums, no TCP state machine)
- **Bidirectional**: Unlike pipes
- **FD passing**: Send open file descriptors between processes
- **Credential passing**: Get PID/UID/GID of peer process (SO_PEERCRED)
- **Used by**: Docker daemon, X11, D-Bus, systemd, PostgreSQL local, containerd

```bash
# List unix domain sockets
ss -x | head -20
ss -xlp | grep docker    # Docker socket
ls -la /var/run/docker.sock
```

---

## Shared Memory (POSIX)

### POSIX Shared Memory

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

// Process 1: Create and write
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, SIZE);
void *ptr = mmap(NULL, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);  // FD no longer needed after mmap

// Write data
struct shared_data *data = (struct shared_data *)ptr;
data->counter = 42;
data->ready = 1;

// Process 2: Open and read
int fd = shm_open("/my_shm", O_RDONLY, 0);
void *ptr = mmap(NULL, SIZE, PROT_READ, MAP_SHARED, fd, 0);
close(fd);

// Read data
struct shared_data *data = (struct shared_data *)ptr;
while (!data->ready) { /* spin or use semaphore */ }
printf("Counter: %d\n", data->counter);

// Cleanup
munmap(ptr, SIZE);
shm_unlink("/my_shm");  // Remove shared memory object
```

### System V Shared Memory (Legacy)

```c
#include <sys/ipc.h>
#include <sys/shm.h>

key_t key = ftok("/tmp/shmfile", 'R');
int shmid = shmget(key, SIZE, IPC_CREAT | 0666);
void *ptr = shmat(shmid, NULL, 0);

// Use shared memory...

shmdt(ptr);                         // Detach
shmctl(shmid, IPC_RMID, NULL);     // Remove
```

### Synchronization with Shared Memory
Shared memory provides **no synchronization** — you must use:
- POSIX semaphores (`sem_t` in shared memory)
- Mutexes with `PTHREAD_PROCESS_SHARED` attribute
- Atomic operations
- File locks (flock/fcntl)

```bash
# View shared memory segments
ls /dev/shm/            # POSIX shared memory objects
ipcs -m                  # System V shared memory
```

---

## Message Queues

### POSIX Message Queues

```c
#include <mqueue.h>

// Create/open queue
struct mq_attr attr = {
    .mq_maxmsg = 10,       // Max messages in queue
    .mq_msgsize = 256      // Max message size
};
mqd_t mq = mq_open("/my_queue", O_CREAT | O_RDWR, 0666, &attr);

// Send message
const char *msg = "Hello";
mq_send(mq, msg, strlen(msg) + 1, 0);  // 0 = priority

// Receive message
char buf[256];
unsigned int prio;
mq_receive(mq, buf, sizeof(buf), &prio);

// Cleanup
mq_close(mq);
mq_unlink("/my_queue");
```

### Message Queue vs Pipe
- **Message boundaries**: Each send/receive is a complete message (unlike pipe's byte stream)
- **Priority**: Messages can have priorities
- **Persistence**: Queue persists until explicitly removed
- **Non-blocking**: Can use `O_NONBLOCK` or timed operations

---

## Semaphores

### POSIX Named Semaphore

```c
#include <semaphore.h>

// Create (works across processes)
sem_t *sem = sem_open("/my_sem", O_CREAT, 0644, 1);  // Initial value = 1

// Lock (decrement, blocks if 0)
sem_wait(sem);

// Critical section...

// Unlock (increment)
sem_post(sem);

// Try without blocking
if (sem_trywait(sem) == 0) {
    // Got the semaphore
    sem_post(sem);
}

// Cleanup
sem_close(sem);
sem_unlink("/my_sem");
```

### Unnamed Semaphore (Thread or Process)

```c
sem_t sem;

// For threads
sem_init(&sem, 0, 1);    // 0 = thread-shared

// For processes (must be in shared memory)
sem_init(&sem, 1, 1);    // 1 = process-shared
```

---

## Signals as IPC

```c
#include <signal.h>

// Send signal to a process
kill(pid, SIGTERM);       // Send SIGTERM
kill(pid, SIGUSR1);       // Send custom signal
kill(-pgid, SIGTERM);     // Send to process group
kill(0, SIGTERM);         // Send to all processes in same group

// Real-time signals (with data payload)
union sigval sv;
sv.sival_int = 42;
sigqueue(pid, SIGRTMIN, sv);  // Send signal with integer data
```

### Limitations of Signals for IPC
- Very limited data (just signal number or sigval)
- Standard signals can be merged (only one pending per type)
- Async-signal-safety constraints in handlers
- Real-time signals are queued but still limited
- Better alternatives: sockets, shared memory, message queues

---

## Memory-Mapped Files

### Shared State with Persistence

```c
// Multiple processes can map the same file
int fd = open("/tmp/shared_state.dat", O_RDWR | O_CREAT, 0666);
ftruncate(fd, sizeof(struct state));

// MAP_SHARED: changes visible to other processes AND written to file
struct state *s = mmap(NULL, sizeof(struct state),
    PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);

// Access shared state
s->counter++;

// Changes persist to file
// (async by default, or use msync for explicit flush)
msync(s, sizeof(struct state), MS_SYNC);

munmap(s, sizeof(struct state));
```

### Comparison: Shared Memory vs Memory-Mapped File

| Feature | POSIX Shared Memory | Memory-Mapped File |
|---|---|---|
| Backing | RAM (+ swap) | File on disk |
| Persistence | Until shm_unlink() or reboot | Permanent (file) |
| Path | /dev/shm/ | Any file path |
| Speed | Fastest (RAM only) | Depends on page cache |
| Use case | Temporary sharing | Persistent shared state |

---

## Choosing the Right IPC

```
Need structured messages? → Message Queue
Need bulk data transfer?  → Shared Memory + Semaphore
Need bidirectional comm?  → Unix Domain Socket
Need cross-network?       → TCP/UDP Socket
Need simple parent→child? → Pipe
Need unrelated process?   → Named Pipe, Unix Socket, or Shared Memory
Need FD passing?          → Unix Domain Socket (SCM_RIGHTS)
Need persistence?         → Memory-Mapped File
Need notification only?   → Signal
```

---

## Interview Tips

1. **Fastest IPC mechanism?**
   → Shared memory. No kernel involvement for data transfer after setup. Must handle synchronization yourself.

2. **Unix socket vs TCP loopback?**
   → Unix socket: no network stack (no checksums, no TCP state), FD/credential passing. 2-3x faster.

3. **Pipe vs message queue?**
   → Pipe: byte stream, no boundaries. MQ: discrete messages with priority, persist until removed.

4. **When to use shared memory?**
   → High-throughput data sharing between processes on same host. E.g., database buffer pool, shared cache. Need separate sync (semaphore/mutex).

5. **How does FD passing work?**
   → Send file descriptor over Unix domain socket using SCM_RIGHTS ancillary data. Receiving process gets a new FD referencing the same kernel file object.

---

*Last updated: March 13, 2026*
