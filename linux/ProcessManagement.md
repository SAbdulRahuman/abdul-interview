# Process Management

> Linux process lifecycle, creation, termination, signals, and the /proc filesystem.

---

## Process Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│  Process State Machine (Linux)                               │
│                                                              │
│  fork()          exec()                                      │
│    │               │                                         │
│    ▼               ▼                                         │
│  CREATED ──────► READY ◄────── WAITING (I/O, signal, etc.)  │
│                   │  ▲              ▲                         │
│            schedule│  │preempt      │ blocked                │
│                   ▼  │              │                         │
│                RUNNING ────────────►│                         │
│                   │                                          │
│              exit()│                                          │
│                   ▼                                          │
│                ZOMBIE ──► TERMINATED (parent calls wait())   │
└──────────────────────────────────────────────────────────────┘
```

### Process States in `/proc/<pid>/status`

| State | Code | Description |
|---|---|---|
| Running | R | Executing or on run queue |
| Sleeping (interruptible) | S | Waiting for event, can wake on signal |
| Sleeping (uninterruptible) | D | Waiting for I/O, cannot be interrupted |
| Stopped | T | Stopped by signal (SIGSTOP/SIGTSTP) |
| Zombie | Z | Exited but parent hasn't called wait() |
| Dead | X | Being removed from process table |
| Idle | I | Kernel thread idle state |

---

## fork() — Process Creation

```c
#include <unistd.h>
#include <sys/wait.h>
#include <stdio.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // Child process
        printf("Child PID: %d, Parent PID: %d\n", getpid(), getppid());
        // Do child work...
        _exit(0);  // Use _exit() in child after fork (avoids flushing parent's buffers)
    } else {
        // Parent process
        printf("Parent PID: %d, Child PID: %d\n", getpid(), pid);
        int status;
        waitpid(pid, &status, 0);  // Wait for child
        if (WIFEXITED(status)) {
            printf("Child exited with status %d\n", WEXITSTATUS(status));
        }
    }
    return 0;
}
```

### Copy-on-Write (COW) Semantics

- `fork()` does NOT copy the parent's entire address space
- Both parent and child share the **same physical pages**, marked read-only
- When either process **writes** to a page, a **page fault** occurs
- The kernel then copies that specific page → "copy-on-write"
- This makes `fork()` very fast (just copy page tables, not data)

```
Before fork:
  Parent: [Page A] [Page B] [Page C]   → Physical: [P1] [P2] [P3]

After fork (COW):
  Parent: [Page A] [Page B] [Page C]   ─┐
                                         ├→ Physical: [P1] [P2] [P3] (shared, read-only)
  Child:  [Page A] [Page B] [Page C]   ─┘

After child writes to Page B:
  Parent: [Page A] [Page B] [Page C]   → Physical: [P1] [P2] [P3]
  Child:  [Page A] [Page B'] [Page C]  → Physical: [P1] [P2'] [P3]
                    ↑ copied on write         ↑ new copy
```

---

## exec() Family — Replace Process Image

| Function | Path | Args | Environment |
|---|---|---|---|
| `execve()` | Full path | Array | Array (explicit) |
| `execv()` | Full path | Array | Inherited |
| `execvp()` | Uses $PATH | Array | Inherited |
| `execl()` | Full path | Variadic | Inherited |
| `execlp()` | Uses $PATH | Variadic | Inherited |
| `execle()` | Full path | Variadic | Array (explicit) |

```c
// Common pattern: fork + exec
pid_t pid = fork();
if (pid == 0) {
    // Child: replace with new program
    char *args[] = {"/bin/ls", "-la", "/tmp", NULL};
    execv("/bin/ls", args);
    // If exec returns, it failed
    perror("exec failed");
    _exit(1);
}
// Parent continues...
```

### Key Points
- `exec()` replaces the **entire process image** (text, data, heap, stack)
- PID remains the same
- Open file descriptors are preserved (unless `FD_CLOEXEC` / `O_CLOEXEC` is set)
- The `exec()` call **never returns** on success

---

## wait() / waitpid() — Reaping Child Processes

```c
#include <sys/wait.h>

// Wait for any child
int status;
pid_t child = wait(&status);

// Wait for specific child
pid_t child = waitpid(specific_pid, &status, 0);

// Non-blocking wait (check if any child exited)
pid_t child = waitpid(-1, &status, WNOHANG);
if (child > 0) {
    // Child exited
} else if (child == 0) {
    // No child has exited yet
}

// Inspect exit status
if (WIFEXITED(status)) {
    int exit_code = WEXITSTATUS(status);     // Normal exit
}
if (WIFSIGNALED(status)) {
    int sig = WTERMSIG(status);               // Killed by signal
    bool core = WCOREDUMP(status);            // Core dump generated?
}
if (WIFSTOPPED(status)) {
    int sig = WSTOPSIG(status);               // Stopped by signal
}
```

---

## Zombie and Orphan Processes

### Zombie Process
- Child has exited but parent hasn't called `wait()`
- Shows as state `Z` in `ps aux`
- Occupies a process table entry (PID not reusable)
- Cannot be killed (already dead); only the parent can reap it
- If parent ignores `SIGCHLD` (`signal(SIGCHLD, SIG_IGN)`), zombies are auto-reaped

```bash
# Find zombie processes
ps aux | awk '$8 ~ /^Z/'

# Find parent of zombie
ps -o ppid= -p <zombie_pid>
# Kill the parent (or fix it to call wait())
```

### Orphan Process
- Parent exits before child
- Child is **re-parented** to `init` (PID 1) or `systemd`
- `init` automatically calls `wait()` for orphans → no zombie accumulation
- This is intentional in the double-fork daemon pattern

---

## Process Groups and Sessions

```
┌───────────────────────────────────────────────┐
│  Session (sid = session leader PID)           │
│                                               │
│  ┌───────────────────────────────────────┐    │
│  │  Foreground Process Group (pgid)      │    │
│  │  ┌──────┐ ┌──────┐ ┌──────┐          │    │
│  │  │ Proc │ │ Proc │ │ Proc │          │    │
│  │  └──────┘ └──────┘ └──────┘          │    │
│  └───────────────────────────────────────┘    │
│                                               │
│  ┌───────────────────────────────────────┐    │
│  │  Background Process Group (pgid)      │    │
│  │  ┌──────┐ ┌──────┐                   │    │
│  │  │ Proc │ │ Proc │                   │    │
│  │  └──────┘ └──────┘                   │    │
│  └───────────────────────────────────────┘    │
│                                               │
│  Controlling Terminal: /dev/pts/0             │
└───────────────────────────────────────────────┘
```

- **Process group**: Collection of related processes (e.g., pipeline `ls | grep | wc`)
- **Session**: Collection of process groups, led by a session leader (typically the shell)
- **Controlling terminal**: Terminal attached to session; sends signals (Ctrl+C → `SIGINT` to foreground group)
- `setpgid(pid, pgid)` — set process group
- `setsid()` — create new session (detach from terminal)

---

## Daemon Creation — Double Fork Pattern

```c
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>

void daemonize() {
    // Step 1: First fork — parent exits, child continues
    pid_t pid = fork();
    if (pid > 0) _exit(0);   // Parent exits
    if (pid < 0) _exit(1);   // Error

    // Step 2: Create new session (detach from terminal)
    setsid();

    // Step 3: Second fork — ensure daemon cannot acquire terminal
    pid = fork();
    if (pid > 0) _exit(0);   // First child exits
    if (pid < 0) _exit(1);   // Error

    // Step 4: Set working directory
    chdir("/");

    // Step 5: Reset file mode mask
    umask(0);

    // Step 6: Close all inherited file descriptors
    for (int fd = sysconf(_SC_OPEN_MAX); fd >= 0; fd--)
        close(fd);

    // Step 7: Redirect stdin/stdout/stderr to /dev/null
    open("/dev/null", O_RDWR);  // stdin  (fd 0)
    dup(0);                     // stdout (fd 1)
    dup(0);                     // stderr (fd 2)

    // Now running as a proper daemon
}
```

### Why Double Fork?
1. **First fork**: Parent exits → child is no longer a process group leader
2. **`setsid()`**: Creates new session (requires not being a group leader)
3. **Second fork**: Session leader exits → grandchild can never acquire a controlling terminal

Modern alternative: `systemd` manages daemon lifecycle, so double fork is often unnecessary.

---

## /proc Filesystem

### Per-Process Information

| Path | Content |
|---|---|
| `/proc/<pid>/status` | Human-readable status (name, state, memory, threads) |
| `/proc/<pid>/stat` | Machine-readable status (single line) |
| `/proc/<pid>/maps` | Memory mappings (virtual address ranges) |
| `/proc/<pid>/smaps` | Detailed memory maps (RSS, PSS, shared/private) |
| `/proc/<pid>/fd/` | Open file descriptors (symlinks to files/sockets) |
| `/proc/<pid>/cmdline` | Null-separated command line arguments |
| `/proc/<pid>/environ` | Null-separated environment variables |
| `/proc/<pid>/cgroup` | Cgroup membership |
| `/proc/<pid>/ns/` | Namespace symlinks (pid, net, mnt, etc.) |
| `/proc/<pid>/io` | I/O statistics (rchar, wchar, read_bytes, write_bytes) |
| `/proc/<pid>/oom_score` | OOM killer score (higher = more likely to be killed) |
| `/proc/<pid>/limits` | Resource limits (open files, memory, etc.) |

```bash
# Process info examples
cat /proc/self/status | grep -i "vm\|rss\|threads\|state"
cat /proc/self/maps | head -20
ls -la /proc/self/fd/ | wc -l       # Count open FDs
cat /proc/self/io                     # I/O counters
cat /proc/<pid>/oom_score_adj         # Adjust OOM priority (-1000 to 1000)
```

### System-Wide /proc Entries

| Path | Content |
|---|---|
| `/proc/meminfo` | System memory details |
| `/proc/cpuinfo` | CPU information |
| `/proc/loadavg` | Load averages |
| `/proc/stat` | System statistics |
| `/proc/interrupts` | Interrupt counts per CPU |
| `/proc/sys/` | Tunable kernel parameters (sysctl) |

---

## Signals

### Signal Table

| Signal | Number | Default Action | Common Use |
|---|---|---|---|
| `SIGTERM` | 15 | Terminate | Graceful shutdown request |
| `SIGKILL` | 9 | Terminate (uncatchable) | Force kill |
| `SIGINT` | 2 | Terminate | Ctrl+C |
| `SIGQUIT` | 3 | Core dump | Ctrl+\ (Go: goroutine dump) |
| `SIGHUP` | 1 | Terminate | Terminal hangup / config reload |
| `SIGUSR1` | 10 | User-defined | Custom actions (log rotation, debug) |
| `SIGUSR2` | 12 | User-defined | Custom actions |
| `SIGPIPE` | 13 | Terminate | Write to pipe with no reader |
| `SIGCHLD` | 17 | Ignore | Child process status change |
| `SIGSTOP` | 19 | Stop (uncatchable) | Pause process |
| `SIGCONT` | 18 | Continue | Resume stopped process |
| `SIGALRM` | 14 | Terminate | Timer expiration (alarm()) |
| `SIGSEGV` | 11 | Core dump | Invalid memory access |
| `SIGBUS` | 7 | Core dump | Bus error (alignment, mmap beyond file) |
| `SIGFPE` | 8 | Core dump | Arithmetic error (division by zero) |
| `SIGABRT` | 6 | Core dump | abort() called |

### Signal Handling in C

```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

volatile sig_atomic_t got_signal = 0;

void handler(int signo) {
    // ONLY use async-signal-safe functions here
    // (write, _exit, signal — NOT printf, malloc, etc.)
    got_signal = signo;
}

int main() {
    // Proper way: sigaction (avoid signal() — behavior varies across systems)
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);      // Don't block other signals during handler
    sa.sa_flags = SA_RESTART;       // Restart interrupted syscalls
    sigaction(SIGTERM, &sa, NULL);
    sigaction(SIGINT, &sa, NULL);

    // Block SIGPIPE (common in network programs)
    signal(SIGPIPE, SIG_IGN);

    while (!got_signal) {
        // Main work loop
        pause();  // Wait for a signal
    }
    printf("Received signal %d, shutting down...\n", got_signal);
    return 0;
}
```

### Signal Handling in Go

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    // Create context that cancels on SIGTERM/SIGINT
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGTERM, syscall.SIGINT)
    defer stop()

    // Or use channel-based approach
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT, syscall.SIGHUP)

    select {
    case sig := <-sigCh:
        fmt.Printf("Received signal: %v\n", sig)
        // Graceful shutdown
    case <-ctx.Done():
        fmt.Println("Context cancelled")
    }
}
```

### Async-Signal-Safe Functions

Only these functions are safe to call inside a signal handler (POSIX):
- `write()`, `_exit()`, `signal()`, `sigaction()`
- `open()`, `close()`, `read()`, `lseek()`
- `fork()`, `execve()`, `kill()`, `raise()`
- `getpid()`, `getppid()`
- **NOT safe**: `printf()`, `malloc()`, `free()`, `syslog()`, `pthread_*`

Best practice: Set a flag in the handler, check it in the main loop.

---

## Interview Tips

### Common Questions

1. **What happens when you call fork()?**
   → Copy process table entry, share pages as COW, return 0 to child, child PID to parent.

2. **How to avoid zombie processes?**
   → Call `wait()`/`waitpid()`, use `SIGCHLD` handler, set `SIG_IGN` for `SIGCHLD`, or double fork.

3. **Difference between SIGTERM and SIGKILL?**
   → `SIGTERM` can be caught/handled (graceful shutdown); `SIGKILL` cannot be caught (immediate termination).

4. **What is a daemon? How to create one?**
   → Background process without a controlling terminal. Created via double fork + setsid + close FDs. Modern: use systemd.

5. **Zombie vs orphan?**
   → Zombie: child exited, parent hasn't waited. Orphan: parent exited, child re-parented to init.

---

*Last updated: March 13, 2026*
