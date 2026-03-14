# Chapter 38 — Debugging & Troubleshooting in Go

Production debugging and troubleshooting is a critical skill for systems engineers building storage appliances, distributed systems, and Kubernetes-native services. This chapter covers the full debugging toolkit for Go — from interactive debuggers to production diagnostics, crash analysis, and distributed system observability.

---

## Debugging Methodology

```
┌──────────────────────────────────────────────────────────────────────┐
│  Systematic Debugging Framework                                      │
│                                                                      │
│  1. REPRODUCE    → Can you trigger the bug reliably?                │
│  2. ISOLATE      → Narrow down the failing component                │
│  3. INSTRUMENT   → Add logging, metrics, traces, or breakpoints     │
│  4. ROOT CAUSE   → Identify the fundamental issue                   │
│  5. FIX          → Apply the minimal correct fix                    │
│  6. VERIFY       → Confirm fix resolves the issue                   │
│  7. PREVENT      → Add tests, monitoring, or guards                 │
│                                                                      │
│  Common Pitfall: Jumping to step 5 without steps 1-4               │
│  → Leads to whack-a-mole debugging and regressions                 │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Debugging Principles

1. **Reproduce first** — if you can't reproduce it, you can't verify the fix
2. **Minimize the reproduction** — strip away unrelated code until you have the simplest failing case
3. **Check recent changes** — `git log --oneline -20`, `git bisect` to find the introducing commit
4. **Read the error message carefully** — Go errors are usually precise
5. **Don't assume** — verify each assumption with data (logs, traces, metrics)
6. **Rubber duck debugging** — explain the problem out loud to find logic gaps

---

## Delve — The Go Debugger

Delve (`dlv`) is the standard debugger for Go. Unlike GDB, Delve understands goroutines, channels, interfaces, and Go's runtime.

### Installation

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

### Starting a Debug Session

```bash
# Debug current package
dlv debug

# Debug a specific binary
dlv debug ./cmd/server

# Attach to running process
dlv attach <pid>

# Debug a test
dlv test ./pkg/storage/...

# Debug a core dump
dlv core <binary> <core-file>

# Run with arguments
dlv debug ./cmd/server -- --port 8080 --config /etc/app.yaml

# Headless mode (for IDE integration)
dlv debug --headless --api-version=2 --listen=:2345
```

### Essential Delve Commands

```
┌─────────────────────────────────────────────────────────────────────┐
│  Delve Command Reference                                            │
│                                                                     │
│  Navigation:                                                        │
│  ─────────                                                          │
│  break (b)     → Set breakpoint: b main.go:42, b pkg.FuncName     │
│  continue (c)  → Run until next breakpoint                         │
│  next (n)      → Step over (execute current line, move to next)    │
│  step (s)      → Step into function call                           │
│  stepout (so)  → Step out of current function                      │
│  restart (r)   → Restart the program                               │
│                                                                     │
│  Inspection:                                                        │
│  ──────────                                                         │
│  print (p)     → Print variable: p myVar, p myStruct.Field        │
│  locals        → Show all local variables                          │
│  args          → Show function arguments                           │
│  whatis        → Show type of expression: whatis myVar             │
│  set           → Set variable value: set myVar = 42                │
│                                                                     │
│  Goroutines:                                                        │
│  ───────────                                                        │
│  goroutines    → List all goroutines                               │
│  goroutine N   → Switch to goroutine N                             │
│  goroutines -t → Show goroutines with stack traces                 │
│                                                                     │
│  Stack:                                                             │
│  ──────                                                             │
│  stack (bt)    → Print stack trace                                 │
│  frame N       → Switch to stack frame N                           │
│  up / down     → Move up/down the call stack                       │
│                                                                     │
│  Breakpoints:                                                       │
│  ────────────                                                       │
│  breakpoints   → List all breakpoints                              │
│  clear N       → Delete breakpoint N                               │
│  clearall      → Delete all breakpoints                            │
│  condition N   → Conditional breakpoint: cond 1 i > 100           │
│  on N          → Execute command on breakpoint hit                 │
│                                                                     │
│  Advanced:                                                          │
│  ─────────                                                          │
│  call          → Call function: call myFunc(arg1, arg2)            │
│  regs          → Show CPU registers                                │
│  disassemble   → Show assembly                                     │
│  examinemem    → Examine raw memory                                │
└─────────────────────────────────────────────────────────────────────┘
```

### Conditional Breakpoints

```bash
(dlv) break main.go:85
Breakpoint 1 set at 0x4a2b3c for main.processRequest() ./main.go:85

# Only break when condition is true
(dlv) condition 1 req.Method == "POST" && req.ContentLength > 1048576

# Break after N hits (skip first 99)
(dlv) condition 1 hitcount > 99
```

### Debugging Goroutines

```bash
# List all goroutines with their status
(dlv) goroutines
  Goroutine 1 - User: ./main.go:45 main.main (0x4a2380)
  Goroutine 6 - User: ./server.go:112 server.handleConn (0x4a5b20) [IO wait]
  Goroutine 7 - User: ./worker.go:33 worker.process (0x4a7c40) [chan receive]
  Goroutine 18 - User: ./storage.go:67 storage.flush (0x4a8d60) [select]

# Switch to goroutine and inspect
(dlv) goroutine 7
Switched to goroutine 7

(dlv) bt
0  0x00000000004a7c40 in worker.process
   at ./worker.go:33
1  0x00000000004a7a80 in worker.Run
   at ./worker.go:18
2  0x0000000000451c20 in runtime.goexit
   at /usr/local/go/src/runtime/asm_amd64.s:1695

(dlv) p job
*worker.Job {
    ID: "abc-123",
    Type: "backup",
    Status: "pending",
    CreatedAt: 2026-03-13T10:30:00Z,
}
```

### Debugging Tests

```bash
# Debug a specific test function
dlv test ./pkg/storage/ -- -test.run TestSnapshotConsistency

# Debug with build tags
dlv test --build-flags="-tags=integration" ./pkg/storage/

# Debug a benchmark
dlv test ./pkg/cache/ -- -test.bench BenchmarkLRUEviction -test.run ^$
```

### Remote Debugging (Containers / Kubernetes)

```bash
# In the container: run delve in headless mode
dlv exec /app/server --headless --listen=:2345 --api-version=2 --accept-multiclient

# From your machine: connect to remote delve
dlv connect <pod-ip>:2345

# For Kubernetes: port-forward first
kubectl port-forward pod/my-app-pod 2345:2345
dlv connect localhost:2345
```

**Dockerfile for debug builds:**

```dockerfile
# Debug stage
FROM golang:1.23 AS debug
RUN go install github.com/go-delve/delve/cmd/dlv@latest
WORKDIR /app
COPY . .
RUN go build -gcflags="all=-N -l" -o /server ./cmd/server
EXPOSE 2345 8080
CMD ["dlv", "exec", "/server", "--headless", "--listen=:2345", "--api-version=2"]
```

**Key build flags for debugging:**

```bash
# Disable optimizations and inlining (required for accurate debugging)
go build -gcflags="all=-N -l" -o myapp ./cmd/myapp

# -N = disable optimizations
# -l = disable inlining
# all= applies to all packages (not just main)
```

---

## Goroutine Dumps (SIGQUIT)

When a Go program is stuck (deadlock, hang), send `SIGQUIT` to get a full goroutine dump:

```bash
# Send SIGQUIT to running process
kill -QUIT <pid>

# Or: press Ctrl+\ in the terminal running the program

# The output goes to stderr — capture it:
kill -QUIT $(pgrep myapp) 2>&1 | tee goroutine-dump.txt
```

### Reading Goroutine Dumps

```
goroutine 1 [running]:
main.main()
        /app/main.go:45 +0x1a8

goroutine 18 [chan send, 5 minutes]:          ← STUCK for 5 minutes!
main.producer(0xc000106000)
        /app/producer.go:28 +0x94

goroutine 19 [semacquire, 3 minutes]:         ← Waiting on mutex for 3 minutes
sync.runtime_SemacquireMutex(0xc000108004, 0x0, 0x1)
        /usr/local/go/src/runtime/sema.go:77 +0x25
sync.(*Mutex).lockSlow(0xc000108000)
        /usr/local/go/src/sync/mutex.go:171 +0x15d
main.(*Storage).Write(0xc000108000, {0xc0001a4000, 0x1000})
        /app/storage.go:89 +0x4f
```

### Common Goroutine States

| State | Meaning | Debugging Action |
|---|---|---|
| `running` | Currently executing on a thread | Normal |
| `runnable` | Ready to run, waiting for a P | Check GOMAXPROCS, CPU saturation |
| `chan send` | Blocked sending on a channel | Find the receiver — is it dead? |
| `chan receive` | Blocked receiving from a channel | Find the sender — did it close? |
| `select` | Blocked in select statement | Check all channel cases |
| `semacquire` | Waiting on mutex/semaphore | Potential deadlock — find the holder |
| `IO wait` | Waiting on I/O (network, file) | Normal for servers; check if stuck |
| `sleep` | In `time.Sleep` | Normal |
| `syscall` | In a system call | Check for slow syscalls (disk, network) |

### Programmatic Goroutine Dumps

```go
import "runtime"

func dumpGoroutines() {
    buf := make([]byte, 1<<20) // 1MB buffer
    n := runtime.Stack(buf, true) // true = all goroutines
    fmt.Fprintf(os.Stderr, "=== GOROUTINE DUMP ===\n%s\n", buf[:n])
}

// Expose via HTTP endpoint for production debugging
http.HandleFunc("/debug/goroutines", func(w http.ResponseWriter, r *http.Request) {
    buf := make([]byte, 1<<20)
    n := runtime.Stack(buf, true)
    w.Header().Set("Content-Type", "text/plain")
    w.Write(buf[:n])
})
```

---

## Race Detector

Go's built-in race detector finds data races at runtime. It's based on ThreadSanitizer.

```bash
# Run with race detector
go run -race ./cmd/server

# Test with race detector
go test -race ./...

# Build with race detector (for production-like testing)
go build -race -o myapp-race ./cmd/server
```

### Understanding Race Reports

```
==================
WARNING: DATA RACE
Write at 0x00c000108000 by goroutine 7:
  main.(*Counter).Increment()
      /app/counter.go:15 +0x64

Previous read at 0x00c000108000 by goroutine 6:
  main.(*Counter).Value()
      /app/counter.go:20 +0x3e

Goroutine 7 (running) created at:
  main.main()
      /app/main.go:30 +0x1a8

Goroutine 6 (running) created at:
  main.main()
      /app/main.go:28 +0x170
==================
```

**Reading the report:**
1. **What:** Write-read race on address `0x00c000108000`
2. **Who:** Goroutine 7 writing, Goroutine 6 reading
3. **Where:** `counter.go:15` (write) vs `counter.go:20` (read)
4. **Origin:** Both goroutines spawned from `main.go`

**Limitations:**
- ~2-10x slowdown, ~5-10x memory overhead
- Only detects races that actually execute during the run
- Not suitable for production (performance overhead)
- False negatives possible (race path not hit during test)

---

## pprof — Production Profiling

### Enabling pprof in Your Application

```go
import (
    "net/http"
    _ "net/http/pprof" // Register pprof handlers
)

func main() {
    // Start pprof server on a separate port
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // ... your application code
}
```

### Profile Types

```
┌───────────────────────────────────────────────────────────────────┐
│  pprof Profile Types                                              │
│                                                                   │
│  Profile          │ URL                        │ What It Shows    │
│  ─────────────────┼────────────────────────────┼────────────────  │
│  CPU              │ /debug/pprof/profile?s=30  │ CPU hotspots     │
│  Heap             │ /debug/pprof/heap          │ Memory allocs    │
│  Goroutine        │ /debug/pprof/goroutine     │ Goroutine stacks │
│  Allocs           │ /debug/pprof/allocs        │ Past allocations │
│  Block            │ /debug/pprof/block         │ Blocking ops     │
│  Mutex            │ /debug/pprof/mutex         │ Mutex contention │
│  Threadcreate     │ /debug/pprof/threadcreate  │ OS thread create │
│  Trace            │ /debug/pprof/trace?s=5     │ Execution trace  │
└───────────────────────────────────────────────────────────────────┘
```

### Collecting and Analyzing Profiles

```bash
# CPU profile (30 seconds)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile (current allocations)
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Mutex contention profile
go tool pprof http://localhost:6060/debug/pprof/mutex

# Interactive commands in pprof:
(pprof) top 20          # Top 20 functions by resource usage
(pprof) list FuncName   # Show source code with profile data
(pprof) web             # Open flame graph in browser
(pprof) peek FuncName   # Show callers and callees
(pprof) tree            # Show call tree
(pprof) svg             # Generate SVG flame graph

# Compare two profiles (before/after optimization)
go tool pprof -base baseline.prof current.prof
```

### Heap Profile Analysis

```bash
# alloc_objects = total allocations (cumulative)
# alloc_space = total bytes allocated (cumulative)
# inuse_objects = current live allocations
# inuse_space = current live bytes

go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
(pprof) top 10
Showing nodes accounting for 142.50MB, 89.2% of 159.72MB total
      flat  flat%   sum%        cum   cum%
   52.00MB 32.55% 32.55%    52.00MB 32.55%  bytes.makeSlice
   30.50MB 19.10% 51.65%    30.50MB 19.10%  encoding/json.(*decodeState).literalStore
   18.00MB 11.27% 62.92%    88.50MB 55.42%  main.(*Storage).processBlock
```

### Memory Leak Detection Pattern

```go
import "runtime"

// Periodically log memory stats
func monitorMemory(interval time.Duration) {
    ticker := time.NewTicker(interval)
    for range ticker.C {
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        log.Printf("Alloc=%vMB TotalAlloc=%vMB Sys=%vMB NumGC=%v Goroutines=%v",
            m.Alloc/1024/1024,
            m.TotalAlloc/1024/1024,
            m.Sys/1024/1024,
            m.NumGC,
            runtime.NumGoroutine(),
        )
    }
}
```

**Memory leak checklist:**
1. Growing goroutine count → goroutine leak (forgotten cancel, blocking channel)
2. Growing heap with no GC relief → reference leak (global map, growing slice)
3. Growing Sys memory → OS memory not returned (fragmentation, cgo)
4. Growing TotalAlloc but stable Alloc → high allocation rate (GC pressure)

---

## Execution Tracer

The Go execution tracer captures fine-grained runtime events — goroutine scheduling, GC pauses, syscalls, network I/O.

```bash
# Collect a 5-second trace
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5

# Analyze in browser
go tool trace trace.out
```

### Programmatic Tracing

```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()

    // ... your code
}
```

### What the Tracer Shows

```
┌─────────────────────────────────────────────────────────────────┐
│  go tool trace — Visual Timeline                                │
│                                                                 │
│  Goroutine Analysis:                                            │
│  • Execution time vs wait time per goroutine                   │
│  • Blocking durations (channel, mutex, syscall, network)       │
│  • Goroutine creation/destruction timeline                     │
│                                                                 │
│  Scheduler Analysis:                                            │
│  • P (processor) utilization over time                         │
│  • Goroutine scheduling latency                                │
│  • Work stealing events                                        │
│                                                                 │
│  GC Analysis:                                                   │
│  • GC pause durations (STW phases)                             │
│  • Mark assist time (goroutines helping GC)                    │
│  • Sweep time                                                  │
│                                                                 │
│  Network & Syscall:                                             │
│  • Network poll durations                                      │
│  • Blocking syscall durations                                  │
│  • Thread creation events                                      │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Tracer vs pprof

| Scenario | Use |
|---|---|
| "Which function is slow?" | pprof CPU profile |
| "Where is memory going?" | pprof heap profile |
| "Why is my goroutine stuck?" | pprof goroutine + tracer |
| "Why is latency p99 high?" | Tracer (scheduling, GC pauses) |
| "Is GC causing pauses?" | Tracer (GC view) |
| "Are goroutines load-balanced?" | Tracer (P utilization) |
| "Why is throughput limited?" | Tracer + pprof CPU |

---

## Core Dump Analysis

### Enabling Core Dumps

```bash
# Set environment variable to enable core dumps on crash
export GOTRACEBACK=crash

# GOTRACEBACK values:
# none    — no goroutine stacktraces on panic
# single  — (default) only panicking goroutine's stack
# all     — all goroutines' stacks
# system  — all goroutines + runtime frames
# crash   — all goroutines + write core dump + crash

# Ensure OS allows core dumps
ulimit -c unlimited
```

### Analyzing Core Dumps

```bash
# After crash, core file is generated (e.g., core.12345)
# Use delve to analyze
dlv core ./myapp core.12345

# Inside delve
(dlv) goroutines
(dlv) goroutine 1
(dlv) bt
(dlv) frame 3
(dlv) locals
(dlv) p myVariable
```

---

## Runtime Diagnostics Environment Variables

```
┌──────────────────────────────────────────────────────────────────────┐
│  Go Runtime Diagnostic Environment Variables                         │
│                                                                      │
│  Variable                  │ Purpose                                 │
│  ──────────────────────────┼──────────────────────────────────────── │
│  GOTRACEBACK=all           │ Print all goroutine stacks on panic     │
│  GOGC=100                  │ GC target percentage (default 100)      │
│  GOMEMLIMIT=4GiB           │ Soft memory limit                      │
│  GOMAXPROCS=8              │ Max OS threads for goroutines           │
│  GODEBUG=gctrace=1         │ Print GC stats to stderr               │
│  GODEBUG=schedtrace=1000   │ Print scheduler stats every 1000ms     │
│  GODEBUG=madvdontneed=1    │ Return memory to OS more aggressively  │
│  GODEBUG=allocfreetrace=1  │ Print alloc/free traces                │
│  GODEBUG=inittrace=1       │ Print init function timings            │
│  GODEBUG=asyncpreemptoff=1 │ Disable async preemption (debug only)  │
└──────────────────────────────────────────────────────────────────────┘
```

### Reading GODEBUG=gctrace=1 Output

```
gc 1 @0.012s 2%: 0.018+1.2+0.012 ms clock, 0.14+0.35/1.0/0.25+0.096 ms cpu, 4->4->2 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P
```

| Field | Meaning |
|---|---|
| `gc 1` | GC cycle number |
| `@0.012s` | Time since program start |
| `2%` | Percentage of CPU spent in GC |
| `0.018+1.2+0.012 ms clock` | STW sweep-termination + concurrent mark + STW mark-termination |
| `4->4->2 MB` | Heap before GC → heap at GC trigger → live heap after GC |
| `4 MB goal` | Target heap size |
| `8 P` | Number of processors |

### Reading GODEBUG=schedtrace=1000 Output

```
SCHED 1003ms: gomaxprocs=8 idleprocs=2 threads=10 spinningthreads=1 needspinning=0
  idlethreads=3 runqueue=5 [2 0 1 0 0 0 3 1]
```

| Field | Meaning |
|---|---|
| `gomaxprocs=8` | Configured P count |
| `idleprocs=2` | Idle Ps (not running goroutines) |
| `threads=10` | Total OS threads |
| `runqueue=5` | Global run queue length |
| `[2 0 1 0 0 0 3 1]` | Local run queue length per P |

---

## Network Debugging in Go

### TCP Connection Debugging

```go
import "net"

// Dial with timeout to detect connection issues
conn, err := net.DialTimeout("tcp", "storage-node:9090", 5*time.Second)
if err != nil {
    if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
        log.Printf("Connection timeout to storage-node:9090")
    } else {
        log.Printf("Connection error: %v", err)
    }
}

// Check for connection reset, broken pipe, EOF
func isConnectionError(err error) bool {
    if err == nil {
        return false
    }
    if errors.Is(err, io.EOF) || errors.Is(err, io.ErrUnexpectedEOF) {
        return true
    }
    if errors.Is(err, syscall.ECONNRESET) || errors.Is(err, syscall.EPIPE) {
        return true
    }
    var opErr *net.OpError
    return errors.As(err, &opErr)
}
```

### HTTP Client Debugging

```go
import "net/http/httptrace"

// Trace HTTP request lifecycle
trace := &httptrace.ClientTrace{
    DNSStart:          func(info httptrace.DNSStartInfo) { log.Printf("DNS lookup: %s", info.Host) },
    DNSDone:           func(info httptrace.DNSDoneInfo) { log.Printf("DNS done: %v, err=%v", info.Addrs, info.Err) },
    ConnectStart:      func(network, addr string) { log.Printf("Connecting to %s", addr) },
    ConnectDone:       func(network, addr string, err error) { log.Printf("Connected: err=%v", err) },
    TLSHandshakeStart: func() { log.Printf("TLS handshake start") },
    TLSHandshakeDone:  func(state tls.ConnectionState, err error) { log.Printf("TLS done: err=%v", err) },
    GotConn:           func(info httptrace.GotConnInfo) { log.Printf("Got conn: reused=%v, idle=%v", info.Reused, info.IdleTime) },
    GotFirstResponseByte: func() { log.Printf("First response byte received") },
}

req, _ := http.NewRequestWithContext(httptrace.WithClientTrace(ctx, trace), "GET", url, nil)
resp, err := client.Do(req)
```

---

## Distributed System Debugging Patterns

### Structured Logging with Correlation

```go
import "log/slog"

// Create a request-scoped logger with correlation ID
func requestLogger(ctx context.Context) *slog.Logger {
    traceID := ctx.Value("trace-id").(string)
    return slog.With(
        "trace_id", traceID,
        "service", "storage-engine",
        "node_id", nodeID,
    )
}

// Use throughout the request path
func (s *Storage) WriteBlock(ctx context.Context, block *Block) error {
    log := requestLogger(ctx)
    log.Info("write block started",
        "block_id", block.ID,
        "size_bytes", len(block.Data),
        "volume_id", block.VolumeID,
    )
    
    startTime := time.Now()
    err := s.backend.Write(ctx, block)
    duration := time.Since(startTime)
    
    if err != nil {
        log.Error("write block failed",
            "block_id", block.ID,
            "duration_ms", duration.Milliseconds(),
            "error", err,
        )
        return fmt.Errorf("write block %s: %w", block.ID, err)
    }
    
    log.Info("write block completed",
        "block_id", block.ID,
        "duration_ms", duration.Milliseconds(),
    )
    return nil
}
```

### Distributed Tracing with OpenTelemetry

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func (s *Storage) ReplicateBlock(ctx context.Context, block *Block, targets []string) error {
    ctx, span := otel.Tracer("storage").Start(ctx, "ReplicateBlock",
        trace.WithAttributes(
            attribute.String("block.id", block.ID),
            attribute.Int("target.count", len(targets)),
        ),
    )
    defer span.End()

    for _, target := range targets {
        if err := s.sendToReplica(ctx, block, target); err != nil {
            span.RecordError(err)
            span.SetStatus(codes.Error, "replication failed")
            return err
        }
    }
    return nil
}
```

### Health Check and Readiness Patterns

```go
// Comprehensive health check endpoint
type HealthStatus struct {
    Status    string            `json:"status"` // "healthy", "degraded", "unhealthy"
    Checks   map[string]Check  `json:"checks"`
    Uptime   string            `json:"uptime"`
    Version  string            `json:"version"`
    Goroutines int             `json:"goroutines"`
}

type Check struct {
    Status   string        `json:"status"`
    Latency  string        `json:"latency,omitempty"`
    Error    string        `json:"error,omitempty"`
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    checks := map[string]Check{}
    
    // Check database
    start := time.Now()
    if err := db.PingContext(r.Context()); err != nil {
        checks["database"] = Check{Status: "unhealthy", Error: err.Error()}
    } else {
        checks["database"] = Check{Status: "healthy", Latency: time.Since(start).String()}
    }
    
    // Check storage backend
    start = time.Now()
    if err := storage.Ping(r.Context()); err != nil {
        checks["storage"] = Check{Status: "unhealthy", Error: err.Error()}
    } else {
        checks["storage"] = Check{Status: "healthy", Latency: time.Since(start).String()}
    }
    
    // Check goroutine count (leak detection)
    goroutines := runtime.NumGoroutine()
    if goroutines > 10000 {
        checks["goroutines"] = Check{Status: "degraded", Error: fmt.Sprintf("high count: %d", goroutines)}
    }
    
    status := HealthStatus{
        Status:     overallStatus(checks),
        Checks:     checks,
        Uptime:     time.Since(startTime).String(),
        Goroutines: goroutines,
    }
    
    json.NewEncoder(w).Encode(status)
}
```

---

## Common Production Issues — Diagnosis Playbook

### 1. Goroutine Leak

**Symptoms:** Growing memory, increasing goroutine count over time

```bash
# Monitor goroutine count
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | head -1
# Output: goroutine profile: total 15234    ← growing over time

# Get full goroutine dump to find leaked goroutines
curl -s http://localhost:6060/debug/pprof/goroutine?debug=2 > goroutine-dump.txt
# Look for goroutines with long wait times
grep -A5 "minutes\|hours" goroutine-dump.txt
```

**Common causes:**
- Channel send/receive with no partner goroutine
- Missing `context.WithCancel` / `context.WithTimeout`
- HTTP response body not closed (`defer resp.Body.Close()`)
- Ticker not stopped (`defer ticker.Stop()`)

### 2. Mutex Deadlock

**Symptoms:** Application hangs, no CPU usage, goroutines stuck in `semacquire`

```bash
# Enable mutex profiling
import "runtime"
runtime.SetMutexProfileFraction(5) // Sample 1 in 5 mutex contention events

# Analyze mutex contention
go tool pprof http://localhost:6060/debug/pprof/mutex
```

**Prevention:**
- Always acquire locks in consistent order
- Use `defer mu.Unlock()` immediately after `mu.Lock()`
- Prefer `sync.RWMutex` when reads dominate
- Use lock-free patterns (channels, atomics) where possible

### 3. High Latency / Slow Requests

**Symptoms:** p99 latency spikes, some requests much slower than others

```bash
# Check if GC is causing pauses
GODEBUG=gctrace=1 ./myapp 2>&1 | grep "gc"

# Collect CPU profile during slow period
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Collect execution trace to see scheduling delays
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=10
go tool trace trace.out
```

**Common causes:**
- GC pauses (tune GOGC/GOMEMLIMIT, reduce allocations)
- Mutex contention (hot lock)
- Goroutine scheduling delay (too many goroutines, too few CPUs)
- Network latency to downstream services 
- Disk I/O blocking (especially with cgo or syscalls)

### 4. OOM (Out of Memory) Crash

**Symptoms:** Process killed by OOM killer, `signal: killed`

```bash
# Check OOM killer logs
dmesg | grep -i "oom\|killed" | tail -20

# Set GOMEMLIMIT to prevent OOM
GOMEMLIMIT=3GiB ./myapp

# Heap profile to find memory hogs
go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
(pprof) top 20
(pprof) list BigAllocFunction
```

### 5. Connection Pool Exhaustion

**Symptoms:** "too many open files", "connection refused", increasing connection wait time

```go
// Monitor connection pool
stats := db.Stats()
log.Printf("Open=%d InUse=%d Idle=%d WaitCount=%d WaitDuration=%v",
    stats.OpenConnections,
    stats.InUse,
    stats.Idle,
    stats.WaitCount,
    stats.WaitDuration,
)

// Proper pool configuration
db.SetMaxOpenConns(100)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(30 * time.Second)
```

---

## Linux Tools for Go Application Debugging

### System-Level Debugging

```bash
# Trace system calls made by Go program
strace -f -e trace=network,write,read -p $(pgrep myapp)

# Trace a specific goroutine's thread
strace -f -p $(pgrep myapp) -e trace=futex   # mutex/channel operations

# Watch file descriptors
ls -la /proc/$(pgrep myapp)/fd | wc -l       # Count open FDs
ls -la /proc/$(pgrep myapp)/fd               # List open FDs

# Monitor network connections
ss -tnp | grep myapp                          # TCP connections
ss -tnp state established | grep myapp        # Established connections
ss -tnp state time-wait | grep myapp          # Connections in TIME_WAIT

# Monitor memory
cat /proc/$(pgrep myapp)/status | grep -i "vmrss\|vmsize\|threads"

# CPU and memory real-time monitoring
top -p $(pgrep myapp) -H                     # Per-thread CPU usage
pidstat -p $(pgrep myapp) -t 1               # Thread-level stats every 1s

# Disk I/O monitoring
iostat -xz 1                                  # Per-device I/O stats
iotop -p $(pgrep myapp)                      # Per-process I/O
```

### BPF/eBPF Tracing

```bash
# Trace Go function calls with bpftrace
bpftrace -e 'uprobe:/path/to/myapp:main.processBlock { printf("processBlock called\n"); }'

# Trace all goroutine creations
bpftrace -e 'uprobe:/path/to/myapp:runtime.newproc1 { printf("new goroutine\n"); }'

# Trace mutex contention
bpftrace -e 'uprobe:/path/to/myapp:sync.(*Mutex).Lock { printf("lock acquired by tid %d\n", tid); }'
```

---

## Debugging Build Tags and Conditional Code

```go
//go:build debug

package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"
)

func init() {
    go func() {
        log.Println("Debug pprof server starting on :6060")
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
}
```

```bash
# Build with debug endpoints
go build -tags debug -o myapp-debug ./cmd/server

# Build without debug endpoints (production)
go build -o myapp ./cmd/server
```

---

## Interview Questions

1. **How would you debug a Go service that's slowly leaking memory in production?**
   - Enable pprof, collect heap profiles at intervals
   - Compare profiles with `go tool pprof -base`
   - Look for growing `inuse_space`, identify the allocation site
   - Check goroutine count for goroutine leaks
   - Monitor `runtime.MemStats` (Alloc, Sys, NumGoroutine)

2. **A Go service intermittently hangs under load. How do you diagnose it?**
   - Send `SIGQUIT` for goroutine dump, look for stuck goroutines
   - Check for deadlock patterns (`semacquire` state)
   - Collect mutex profile to find contention
   - Use execution tracer to see scheduling delays
   - Check connection pool exhaustion (`db.Stats()`)

3. **How do you profile CPU usage in a production Go service without restarting it?**
   - Pre-register `net/http/pprof` handler
   - `go tool pprof http://host:port/debug/pprof/profile?seconds=30`
   - Use `top`, `list`, `web` commands to find hotspots
   - Compare with baseline using `-base` flag

4. **How do you detect data races in a Go codebase?**
   - `go test -race ./...` in CI pipeline
   - `go build -race` for integration testing
   - Review code for shared mutable state without synchronization
   - Cannot use in production (10x overhead)

5. **A Go program crashes with "fatal error: concurrent map writes". How do you fix it?**
   - Use `sync.Mutex` or `sync.RWMutex` to protect map access
   - Or use `sync.Map` for simple use cases
   - Or redesign to confine map to a single goroutine (channel-based access)
   - Run `go test -race` to find other races

6. **How would you debug a networking issue between two Go microservices?**
   - Add `httptrace` to instrument connection lifecycle
   - Check `ss` / `netstat` for connection states
   - Use `strace` to trace syscalls
   - Add structured logging with trace/correlation IDs
   - Check for DNS resolution issues, connection pool limits

---

## Interview Problems

### Problem 1: Implement a Debug-Aware HTTP Middleware

Design an HTTP middleware that:
- Adds a unique request ID to every request
- Logs request start/end with duration
- Captures and reports panics without crashing the server
- Optionally records request/response bodies in debug mode

```go
func DebugMiddleware(debug bool, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()
        ctx := context.WithValue(r.Context(), "request-id", requestID)
        r = r.WithContext(ctx)
        
        w.Header().Set("X-Request-ID", requestID)
        
        start := time.Now()
        slog.Info("request started",
            "request_id", requestID,
            "method", r.Method,
            "path", r.URL.Path,
        )
        
        // Wrap ResponseWriter to capture status code
        wrapped := &statusWriter{ResponseWriter: w, statusCode: http.StatusOK}
        
        // Panic recovery
        defer func() {
            if rec := recover(); rec != nil {
                stack := make([]byte, 4096)
                n := runtime.Stack(stack, false)
                slog.Error("panic recovered",
                    "request_id", requestID,
                    "panic", rec,
                    "stack", string(stack[:n]),
                )
                http.Error(wrapped, "Internal Server Error", http.StatusInternalServerError)
            }
            
            slog.Info("request completed",
                "request_id", requestID,
                "status", wrapped.statusCode,
                "duration_ms", time.Since(start).Milliseconds(),
            )
        }()
        
        next.ServeHTTP(wrapped, r)
    })
}

type statusWriter struct {
    http.ResponseWriter
    statusCode int
}

func (w *statusWriter) WriteHeader(code int) {
    w.statusCode = code
    w.ResponseWriter.WriteHeader(code)
}
```

### Problem 2: Build a Goroutine Leak Detector

Design a utility that periodically samples goroutine counts and alerts when the count grows beyond a threshold:

```go
type LeakDetector struct {
    threshold    int
    interval     time.Duration
    samples      []int
    windowSize   int
    alertFunc    func(current int, trend string)
}

func NewLeakDetector(threshold, windowSize int, interval time.Duration, alertFn func(int, string)) *LeakDetector {
    return &LeakDetector{
        threshold:  threshold,
        interval:   interval,
        windowSize: windowSize,
        samples:    make([]int, 0, windowSize),
        alertFunc:  alertFn,
    }
}

func (ld *LeakDetector) Start(ctx context.Context) {
    ticker := time.NewTicker(ld.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            count := runtime.NumGoroutine()
            ld.samples = append(ld.samples, count)
            if len(ld.samples) > ld.windowSize {
                ld.samples = ld.samples[1:]
            }
            
            if count > ld.threshold {
                trend := ld.analyzeTrend()
                ld.alertFunc(count, trend)
            }
        }
    }
}

func (ld *LeakDetector) analyzeTrend() string {
    if len(ld.samples) < 2 {
        return "insufficient data"
    }
    first := ld.samples[0]
    last := ld.samples[len(ld.samples)-1]
    growth := float64(last-first) / float64(first) * 100
    if growth > 20 {
        return fmt.Sprintf("GROWING: %.1f%% increase over %d samples", growth, len(ld.samples))
    }
    return fmt.Sprintf("STABLE: %.1f%% change over %d samples", growth, len(ld.samples))
}
```

---

*Last updated: March 13, 2026*
