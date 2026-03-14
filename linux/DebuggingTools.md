# Debugging Tools Deep Dive

> GDB, strace, ltrace, tcpdump, core dumps, valgrind, sanitizers, and systematic debugging approaches.

---

## GDB (GNU Debugger)

### Getting Started

```bash
# Compile with debug symbols (essential)
gcc -g -O0 -o myapp myapp.c           # C
g++ -g -O0 -o myapp myapp.cpp         # C++

# Start debugging
gdb ./myapp                             # Interactive debug
gdb --args ./myapp --flag value         # Debug with arguments
gdb -p <pid>                            # Attach to running process
gdb ./myapp core.12345                  # Analyze core dump
```

### Essential Commands

```
(gdb) run                           # Start program
(gdb) run arg1 arg2                  # Start with arguments
(gdb) kill                           # Kill running program

# --- Breakpoints ---
(gdb) break main                     # Break at function
(gdb) break main.c:42               # Break at file:line
(gdb) break myfile.c:100 if x > 10  # Conditional breakpoint
(gdb) tbreak main                    # Temporary breakpoint (one-time)
(gdb) info breakpoints               # List all breakpoints
(gdb) delete 3                       # Delete breakpoint #3
(gdb) disable 2                      # Disable breakpoint #2
(gdb) enable 2                       # Enable breakpoint #2

# --- Execution Control ---
(gdb) next (n)                       # Step over (execute line, don't enter functions)
(gdb) step (s)                       # Step into (enter functions)
(gdb) finish                         # Run until current function returns
(gdb) continue (c)                   # Continue to next breakpoint
(gdb) until 100                      # Run until line 100
(gdb) advance main.c:50             # Run until reaching line 50

# --- Inspection ---
(gdb) print var                      # Print variable value
(gdb) print *ptr                     # Dereference pointer
(gdb) print arr[0]@10               # Print 10 elements of array
(gdb) print sizeof(struct mystruct)  # Print size
(gdb) print/x var                    # Print in hex
(gdb) print/t var                    # Print in binary
(gdb) display var                    # Auto-print on every stop
(gdb) info locals                    # All local variables
(gdb) info args                      # Function arguments
(gdb) ptype var                      # Print type of variable

# --- Stack ---
(gdb) backtrace (bt)                 # Full stack trace
(gdb) bt full                        # Stack trace with local variables
(gdb) frame 3                        # Switch to frame #3
(gdb) up / down                      # Move up/down the stack
(gdb) info frame                     # Current frame details

# --- Memory ---
(gdb) x/10x 0xADDR                  # 10 hex words at address
(gdb) x/10s 0xADDR                  # 10 strings at address
(gdb) x/10i $pc                     # 10 instructions at current PC
(gdb) x/10b 0xADDR                  # 10 bytes at address

# --- Watchpoints ---
(gdb) watch var                      # Break when var changes (write)
(gdb) rwatch var                     # Break when var is read
(gdb) awatch var                     # Break on read or write
(gdb) info watchpoints
```

### Thread Debugging

```
(gdb) info threads                   # List all threads
# Num  Id     Target Id            Frame
# * 1  1234   Thread 0x7ff... "main" at main.c:42
#   2  1235   Thread 0x7ff... "worker" at worker.c:100

(gdb) thread 2                       # Switch to thread 2
(gdb) thread apply all bt            # Stack traces for ALL threads
(gdb) thread apply all bt full       # Detailed stack traces
(gdb) set scheduler-locking on       # Only step current thread (others frozen)
(gdb) set scheduler-locking off      # All threads run
```

### GDB Scripts and Automation

```bash
# Run GDB with commands from file
gdb -x commands.gdb ./myapp

# commands.gdb:
# break main
# run
# backtrace
# quit

# Batch mode (non-interactive)
gdb -batch -ex "bt" -ex "info threads" -p <pid>
```

### GDB for Go Programs

```bash
# Go programs include DWARF debug info by default
gdb ./mygoapp

# But Delve is preferred for Go — see go/DebuggingAndTroubleshooting.md
# GDB limitations with Go:
# - Goroutine awareness is limited
# - Can't inspect channels easily
# - Doesn't understand Go runtime internals
```

---

## strace — System Call Tracer

### Basic Usage

```bash
# Trace a command
strace ./myapp                         # All syscalls
strace -f ./myapp                      # Follow forks (all threads/children)

# Attach to running process
strace -p <pid>                        # Single thread
strace -f -p <pid>                     # All threads
```

### Filtering Syscalls

```bash
# By category
strace -e trace=file ./myapp           # File operations (open, read, write, stat...)
strace -e trace=network ./myapp        # Network (socket, bind, connect, accept...)
strace -e trace=memory ./myapp         # Memory (mmap, brk, mprotect...)
strace -e trace=process ./myapp        # Process (fork, exec, exit, wait...)
strace -e trace=signal ./myapp         # Signal delivery

# By specific syscalls
strace -e trace=open,read,write,close ./myapp
strace -e trace=openat,read,write,close,fsync -f -p <pid>
```

### Timing and Statistics

```bash
# Show time spent in each syscall
strace -T ./myapp
# open("file", O_RDONLY) = 3 <0.000012>    ← 12μs

# Timestamp each syscall
strace -t ./myapp                       # HH:MM:SS
strace -tt ./myapp                      # HH:MM:SS.microseconds
strace -r ./myapp                       # Relative timestamp (since last syscall)

# Summary statistics
strace -c ./myapp
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- ----------------
#  45.00    0.045000          45      1000           read
#  30.00    0.030000          30      1000           write
#  15.00    0.015000          15      1000           open
#   5.00    0.005000           5      1000       200 stat
#   5.00    0.005000           5      1000           close
# ------ ----------- ----------- --------- --------- ----------------
# 100.00    0.100000                  5000       200 total

# Combined: statistics + trace
strace -c -S calls ./myapp             # Sort by call count
```

### Practical Debugging Scenarios

```bash
# What files is an app opening?
strace -e trace=openat -f -p <pid> 2>&1 | grep -v ENOENT

# Why is a process hanging?
strace -p <pid>
# Shows which syscall it's stuck in:
# futex(0x..., FUTEX_WAIT, ...) → waiting for lock (deadlock?)
# read(5, ...                   → waiting for data on FD 5
# epoll_wait(3, ...             → waiting for events (normal for event loop)

# Why is a process slow?
strace -c -f -p <pid>
# → Which syscalls consume the most time?
# Common: too many read/write calls (small I/O without buffering)
#         too many stat calls (unnecessary file checks)

# What network connections?
strace -e trace=network -f ./myapp
# Shows: socket(), connect(), bind(), accept(), sendto(), recvfrom()

# File permission issues
strace -e trace=openat,access ./myapp 2>&1 | grep "EACCES\|EPERM"
```

---

## ltrace — Library Function Tracer

```bash
# Trace shared library calls
ltrace ./myapp
# malloc(1024) = 0x55abcdef
# free(0x55abcdef)
# strlen("hello") = 5
# printf("Count: %d\n", 42) = 10

# Trace specific functions
ltrace -e malloc+free ./myapp           # Only malloc/free
ltrace -e '*@libssl*' ./myapp           # SSL library calls

# With timing
ltrace -T ./myapp

# Attach to running process
ltrace -p <pid>
```

### strace vs ltrace

| Feature | strace | ltrace |
|---|---|---|
| Traces | System calls (kernel boundary) | Library calls (user-space) |
| Overhead | Lower (ptrace on syscalls only) | Higher (every library call) |
| Visibility | Kernel interaction | User-space behavior |
| Use case | I/O, network, file issues | Memory allocation, string ops |

---

## tcpdump — Network Packet Capture

### Basic Capture

```bash
# Capture on interface
tcpdump -i eth0                         # All traffic on eth0
tcpdump -i any                          # All interfaces

# Common filters
tcpdump -i eth0 port 8080              # Specific port
tcpdump -i eth0 host 10.0.0.1          # Specific host
tcpdump -i eth0 src 10.0.0.1           # Source only
tcpdump -i eth0 dst 10.0.0.1           # Destination only
tcpdump -i eth0 tcp                     # TCP only
tcpdump -i eth0 udp                     # UDP only
tcpdump -i eth0 icmp                    # ICMP (ping)

# Combined filters
tcpdump -i eth0 'host 10.0.0.1 and port 8080'
tcpdump -i eth0 'port 8080 or port 9090'
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'  # SYN packets only
tcpdump -i eth0 'tcp[tcpflags] & (tcp-rst) != 0'  # RST packets only
```

### Output Options

```bash
# Don't resolve names (faster, clearer)
tcpdump -nn -i eth0 port 8080

# Limit capture count
tcpdump -c 100 -i eth0                  # Stop after 100 packets

# Save to file (for Wireshark analysis)
tcpdump -w capture.pcap -i eth0 port 8080

# Read from file
tcpdump -r capture.pcap -nn

# Show packet contents
tcpdump -X -i eth0 port 8080 -c 5      # Hex + ASCII
tcpdump -A -i eth0 port 8080 -c 5      # ASCII only (good for HTTP)

# Verbose output
tcpdump -v -i eth0 port 8080           # Verbose
tcpdump -vv -i eth0 port 8080          # More verbose
```

### Practical Scenarios

```bash
# Check if traffic is reaching server
tcpdump -nn -i eth0 port 8080 -c 10

# Diagnose connection issues (look for SYN without SYN-ACK)
tcpdump -nn 'tcp[tcpflags] & (tcp-syn) != 0' -i eth0

# Check for RST (connection reset)
tcpdump -nn 'tcp[tcpflags] & (tcp-rst) != 0' -i eth0

# iSCSI traffic
tcpdump -nn -i eth0 port 3260

# NFS traffic
tcpdump -nn -i eth0 port 2049

# DNS resolution issues
tcpdump -nn -i eth0 port 53
```

---

## Core Dump Analysis

### Enabling Core Dumps

```bash
# Check current limit
ulimit -c                              # 0 = disabled

# Enable (unlimited size)
ulimit -c unlimited

# Set core file pattern
echo "/tmp/core.%e.%p.%t" > /proc/sys/kernel/core_pattern
# %e = executable name
# %p = PID
# %t = timestamp
# %h = hostname

# For systemd systems
echo "kernel.core_pattern=/tmp/core.%e.%p" >> /etc/sysctl.conf
sysctl -p

# Or use coredumpctl (systemd)
coredumpctl list                        # List recent core dumps
coredumpctl info <pid>                  # Info about specific dump
coredumpctl gdb <pid>                   # Open in GDB
```

### Analyzing Core Dumps

```bash
# Open core dump with GDB
gdb ./myapp /tmp/core.myapp.12345

# Essential commands
(gdb) bt                               # Where did it crash?
(gdb) bt full                           # With local variables
(gdb) info threads                      # All threads at crash time
(gdb) thread apply all bt               # Stack traces for ALL threads
(gdb) thread apply all bt full          # Detailed traces for ALL threads

# Examine crash context
(gdb) frame 0                           # Go to crash frame
(gdb) info locals                       # Local variables
(gdb) info args                         # Function arguments
(gdb) print *this                       # Object state (C++)
(gdb) x/10x $sp                        # Examine stack memory

# Common crash causes
# SIGSEGV (11): NULL pointer dereference, buffer overflow, use-after-free
# SIGABRT (6):  assert() failed, abort() called, double free
# SIGBUS (7):   Unaligned access, mmap beyond file size
# SIGFPE (8):   Division by zero
```

### Go Core Dumps

```bash
# Enable Go core dumps
GOTRACEBACK=crash ./mygoapp             # Crash → core dump

# Analyze with Delve
dlv core ./mygoapp core.12345
(dlv) goroutines                        # List all goroutines
(dlv) goroutine <id>                    # Switch to goroutine
(dlv) bt                                # Stack trace
```

---

## Valgrind — Memory Error Detection

### Memcheck (Default Tool)

```bash
# Detect memory errors
valgrind ./myapp
valgrind --leak-check=full ./myapp
valgrind --leak-check=full --show-leak-kinds=all ./myapp
valgrind --track-origins=yes ./myapp    # Track origin of uninitialized values

# Common errors detected:
# - Memory leak (allocated, never freed)
# - Use after free (access freed memory)
# - Buffer overflow (write beyond allocation)
# - Double free (free same pointer twice)
# - Uninitialized value use
# - Invalid read/write (wrong address)
# - Mismatched free (malloc/delete or new/free)
```

### Example Output

```
==12345== Invalid read of size 4
==12345==    at 0x400543: main (myapp.c:10)
==12345==  Address 0x5203048 is 8 bytes inside a block of size 10 free'd
==12345==    at 0x4C2BDEC: free (vg_replace_malloc.c:530)
==12345==    by 0x40053E: main (myapp.c:9)
```

### Other Valgrind Tools

```bash
# Heap profiler
valgrind --tool=massif ./myapp
ms_print massif.out.<pid>              # Heap usage over time

# Thread error detector
valgrind --tool=helgrind ./myapp       # Detect lock ordering issues, data races

# Cache profiler
valgrind --tool=cachegrind ./myapp     # Cache hit/miss simulation
cg_annotate cachegrind.out.<pid>
```

### Valgrind Limitations
- **10-50x slower** than native execution
- Cannot attach to running processes
- Not suitable for production debugging
- Alternative: sanitizers (compile-time, much faster)

---

## Sanitizers (Compile-Time Instrumentation)

### AddressSanitizer (ASan)

```bash
# Compile with ASan
gcc -fsanitize=address -g -O1 -o myapp myapp.c
# Or: clang -fsanitize=address -g -O1 -o myapp myapp.c

# Run (2x slower than native, much faster than valgrind)
./myapp

# Detects:
# - Buffer overflow (stack, heap, global)
# - Use-after-free
# - Use-after-return
# - Double free / invalid free
# - Memory leaks (with detect_leaks)

# Options via environment
ASAN_OPTIONS=detect_leaks=1:halt_on_error=0 ./myapp
```

### MemorySanitizer (MSan)

```bash
# Detect uninitialized memory reads
clang -fsanitize=memory -g -O1 -o myapp myapp.c
./myapp
```

### ThreadSanitizer (TSan)

```bash
# Detect data races
gcc -fsanitize=thread -g -O1 -o myapp myapp.c
./myapp

# Also works with Go:
go run -race main.go
go test -race ./...
```

### UndefinedBehaviorSanitizer (UBSan)

```bash
# Detect undefined behavior
gcc -fsanitize=undefined -g -O1 -o myapp myapp.c
# Detects: integer overflow, null dereference, misaligned access, etc.
```

### Sanitizer Comparison

| Sanitizer | Overhead | Detects |
|---|---|---|
| ASan | 2x slower, 3x memory | Buffer overflow, use-after-free, leaks |
| MSan | 3x slower | Uninitialized reads |
| TSan | 5-15x slower, 5-10x memory | Data races |
| UBSan | Minimal | Undefined behavior |
| Valgrind | 10-50x slower | Everything ASan + cache hits |

---

## Systematic Debugging Approaches

### 1. Application Crash

```
1. Get core dump or crash log
2. gdb ./app core → bt → identify crash frame
3. Check: NULL pointer? Buffer overflow? Use-after-free?
4. Reproduce with ASan if possible
5. Fix → test with sanitizers enabled
```

### 2. Application Hang

```
1. strace -p <pid> → which syscall is it stuck in?
   - futex(): likely deadlock → gdb → thread apply all bt → find lock cycle
   - read()/epoll_wait(): waiting for I/O → check network/disk
   - nanosleep(): sleeping (correct or bug?)
2. If deadlock: gdb → info threads → find threads holding locks
3. For Go: send SIGQUIT → goroutine dump with stack traces
```

### 3. High CPU Usage

```
1. top -H -p <pid> → which thread?
2. perf top -p <pid> → which function?
3. perf record -g -p <pid> -- sleep 30 → flame graph
4. Common: infinite loop, busy-wait, excessive GC, regex backtracking
```

### 4. High Memory Usage

```
1. cat /proc/<pid>/status → VmRSS
2. pmap -x <pid> → which regions are large?
3. Over time: pidstat -r -p <pid> 1 → is RSS growing?
4. If leak: valgrind --leak-check=full or ASan
5. For Go: go tool pprof → heap profile
```

### 5. Slow I/O

```
1. iostat -xz 1 → which device? await high?
2. iotop → which process?
3. strace -c -p <pid> → too many small I/Os?
4. blktrace / biolatency → I/O latency distribution
5. Check: I/O scheduler, queue depth, disk health (smartctl)
```

### 6. Network Issues

```
1. ss -tnp → connection states
2. tcpdump -nn port <port> → is traffic flowing?
3. Look for:
   - SYN without SYN-ACK (connection refused or firewall)
   - RST (connection reset)
   - Retransmissions (nstat -az | grep retrans)
   - CLOSE_WAIT accumulation (app not closing sockets)
   - TIME_WAIT exhaustion (too many short connections)
4. strace -e trace=network -p <pid>
```

---

## Interview Tips

1. **How would you debug a segfault?**
   → Get core dump (ulimit -c unlimited), open with gdb (gdb ./app core), backtrace, examine crash frame, check for NULL deref or buffer overflow. Reproduce with ASan for detailed report.

2. **Process is hanging — how to investigate?**
   → strace -p <pid> to see what syscall it's stuck in. If futex → likely deadlock, use gdb to see thread stacks and lock state. If I/O → check network/disk.

3. **strace vs GDB — when to use which?**
   → strace: observe system call behavior without stopping the process. GDB: inspect internal state, set breakpoints, examine variables. Use strace first (less invasive), GDB for deeper investigation.

4. **How to detect memory leaks in production?**
   → Monitor RSS growth (pidstat -r). For C: ASan or valgrind. For Go: pprof heap profile. BCC memleak tool can trace without restart.

5. **Valgrind vs sanitizers?**
   → Valgrind: no recompile needed, 10-50x slower. Sanitizers: need recompile, 2-5x slower, better for CI/testing. Use sanitizers in CI, valgrind for one-off analysis.

---

*Last updated: March 13, 2026*
