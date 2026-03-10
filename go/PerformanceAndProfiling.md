# Chapter 26 — Performance & Profiling

Go has built-in profiling and tracing tools that require zero external setup. The `pprof` tool is your primary weapon for diagnosing CPU hotspots, memory leaks, and goroutine issues.

```
┌──────────────────────────────────────────────────────────┐
│            Performance Debugging Workflow                │
│                                                          │
│   1. Observe    → metrics, logs, latency spikes          │
│   2. Profile    → pprof (CPU, heap, goroutine, mutex)    │
│   3. Analyze    → go tool pprof (top, list, web)         │
│   4. Benchmark  → go test -bench=. -benchmem            │
│   5. Trace      → go tool trace (scheduler, GC events)  │
│   6. Fix & Verify → re-benchmark to confirm improvement │
│                                                          │
│  Profile Types:                                          │
│  ┌──────────────┬────────────────────────────────────┐   │
│  │ CPU          │ Where is time spent?               │   │
│  │ Heap         │ What's allocating memory?          │   │
│  │ Goroutine    │ What are goroutines doing?         │   │
│  │ Block        │ Where do goroutines block?         │   │
│  │ Mutex        │ Where is lock contention?          │   │
│  │ Trace        │ Timeline of scheduler events       │   │
│  └──────────────┴────────────────────────────────────┘   │
│                                                          │
│  pprof commands:                                        │
│  top     → hottest functions                            │
│  list fn → annotated source code                        │
│  web     → interactive graph in browser                 │
│  png     → export flame graph as image                  │
└──────────────────────────────────────────────────────────┘
```

## pprof — CPU, Memory, Goroutine Profiles

### HTTP Server Profiling

**Tutorial: Enabling Live Profiling via HTTP**

Adding `net/http/pprof` as a blank import automatically registers profiling endpoints on the DefaultServeMux. This code shows how a running HTTP server can expose CPU, memory, goroutine, block, and mutex profiles for live debugging. Note that you must use the DefaultServeMux (pass `nil` to `ListenAndServe`) rather than a custom mux for the pprof routes to be accessible.

```
┌────────────────────────────────────────┐
│         HTTP Server (:8080)           │
│                                        │
│  DefaultServeMux                       │
│  ├── GET /              → handler      │
│  └── /debug/pprof/*     → pprof routes │
│       ├── /profile      → CPU profile  │
│       ├── /heap          → heap allocs  │
│       ├── /goroutine     → goroutine    │
│       ├── /block         → blocking     │
│       ├── /mutex         → mutex        │
│       └── /trace         → exec trace   │
│                                        │
│  _ "net/http/pprof"  ◄── blank import  │
│  registers routes automatically        │
└────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "net/http"
    _ "net/http/pprof" // registers /debug/pprof endpoints
)

func main() {
    // Just importing net/http/pprof adds these endpoints:
    //   /debug/pprof/              — index page
    //   /debug/pprof/profile       — CPU profile (30s default)
    //   /debug/pprof/heap          — heap memory profile
    //   /debug/pprof/goroutine     — goroutine stack dump
    //   /debug/pprof/block         — blocking profile
    //   /debug/pprof/mutex         — mutex contention profile
    //   /debug/pprof/trace         — execution trace

    mux := http.NewServeMux()
    mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello!")
    })

    fmt.Println("Server on :8080 (pprof at /debug/pprof/)")
    http.ListenAndServe(":8080", nil) // use DefaultServeMux which has pprof
}
```

```bash
# Collect a 30-second CPU profile:
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30

# Collect heap profile:
go tool pprof http://localhost:8080/debug/pprof/heap

# Interactive commands inside pprof:
#   top        — show top functions by CPU/memory
#   list foo   — show annotated source of function foo
#   web        — open SVG graph in browser
#   png        — export as PNG
```

### Non-Server Profiling (runtime/pprof)

**Tutorial: Profiling CLI Tools with runtime/pprof**

For CLI tools, batch jobs, or any non-HTTP program, use `runtime/pprof` directly. You explicitly create profile output files, start/stop CPU profiling, and write heap profiles at specific points. The generated `.prof` files can be analyzed offline with `go tool pprof`.

```
┌──────────────────────────────────────┐
│     runtime/pprof Direct Usage      │
│                                      │
│  1. Create file   → os.Create(...)   │
│  2. Start CPU     → pprof.Start...   │
│  3. Run code      → doWork()         │
│  4. Stop CPU      → pprof.Stop...    │
│  5. Write heap    → pprof.Write...   │
│                                      │
│  Output:                             │
│  ┌──────────┐    ┌──────────┐        │
│  │ cpu.prof │    │ mem.prof │        │
│  └────┬─────┘    └────┬─────┘        │
│       └──────┬────────┘              │
│              ▼                       │
│     go tool pprof <file>            │
└──────────────────────────────────────┘
```

```go
package main

import (
    "os"
    "runtime/pprof"
)

func doWork() {
    s := make([]byte, 0)
    for i := 0; i < 1000000; i++ {
        s = append(s, byte(i%256))
    }
}

func main() {
    // CPU Profile
    cpuFile, _ := os.Create("cpu.prof")
    defer cpuFile.Close()
    pprof.StartCPUProfile(cpuFile)
    defer pprof.StopCPUProfile()

    doWork()

    // Memory Profile
    memFile, _ := os.Create("mem.prof")
    defer memFile.Close()
    pprof.WriteHeapProfile(memFile)
}
```

```bash
# Analyze profile:
go tool pprof cpu.prof
go tool pprof mem.prof
```

---

## go tool trace

**Tutorial: Capturing Execution Traces**

`runtime/trace` captures fine-grained execution events — goroutine scheduling, GC pauses, system calls, and network I/O. Unlike pprof which aggregates samples, trace records a timeline of events. The resulting trace file opens in a browser-based viewer showing exactly when each goroutine ran, blocked, or was scheduled.

```
┌──────────────────────────────────────┐
│       Execution Trace Flow          │
│                                      │
│  trace.Start(f) ──► Recording        │
│       │                              │
│       ▼                              │
│  ┌─────────────────────────────┐     │
│  │  Timeline (trace.out)       │     │
│  │  G1: ████░░████░░░████     │     │
│  │  G2:    ░░░████░░░░░░░     │     │
│  │  GC:       ██      ██     │     │
│  │  █=running ░=waiting      │     │
│  └─────────────────────────────┘     │
│       │                              │
│  trace.Stop() ──► Flush to file      │
│       ▼                              │
│  go tool trace trace.out → browser   │
└──────────────────────────────────────┘
```

```go
package main

import (
    "os"
    "runtime/trace"
)

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()

    trace.Start(f)
    defer trace.Stop()

    // Your program logic here
    ch := make(chan int)
    go func() { ch <- 42 }()
    <-ch
}
```

```bash
# View trace in browser:
go tool trace trace.out
# Opens browser with timeline showing goroutine scheduling, GC events, etc.
```

---

## Benchmarking

**Tutorial: Writing and Running Go Benchmarks**

Go's `testing.B` type provides built-in benchmarking with automatic iteration tuning. The framework adjusts `b.N` to run enough iterations for statistically meaningful results. Use `b.ReportAllocs()` for allocation counts, sub-benchmarks with `b.Run()` for parameterized tests, and `benchstat` to compare before/after performance across multiple runs.

```
┌───────────────────────────────────────────┐
│           Benchmark Lifecycle            │
│                                           │
│  go test -bench=. -benchmem              │
│       │                                   │
│       ▼                                   │
│  ┌─────────────────────────────────┐      │
│  │  b.N = 1                       │      │
│  │  Run loop → too fast? increase  │      │
│  │  b.N = 100                      │      │
│  │  Run loop → too fast? increase  │      │
│  │  b.N = 10000                    │      │
│  │  Run loop → stable timing ✓    │      │
│  └─────────────────────────────────┘      │
│       │                                   │
│       ▼                                   │
│  Output: BenchmarkXxx-8  10000           │
│          123 ns/op  48 B/op  2 allocs/op │
└───────────────────────────────────────────┘
```

```go
// bench_test.go
package main

import (
    "fmt"
    "strings"
    "testing"
)

// Basic benchmark
func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "x"
        }
    }
}

// Benchmark with Builder (much faster)
func BenchmarkStringBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("x")
        }
        _ = sb.String()
    }
}

// Report allocations
func BenchmarkWithAllocs(b *testing.B) {
    b.ReportAllocs() // include alloc stats in output
    for i := 0; i < b.N; i++ {
        _ = fmt.Sprintf("hello %d", i)
    }
}

// Sub-benchmarks for different sizes
func BenchmarkSliceAppend(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                s := make([]int, 0)
                for j := 0; j < size; j++ {
                    s = append(s, j)
                }
            }
        })
    }
}

// Benchmark with pre-allocated slice
func BenchmarkSlicePrealloc(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                s := make([]int, 0, size) // pre-allocate!
                for j := 0; j < size; j++ {
                    s = append(s, j)
                }
            }
        })
    }
}
```

```bash
# Run benchmarks:
go test -bench=. -benchmem

# Compare benchmarks with benchstat:
go test -bench=. -count=5 > old.txt
# (make changes)
go test -bench=. -count=5 > new.txt
benchstat old.txt new.txt
```

---

## Common Optimizations

### Pre-allocate Slices

**Tutorial: Avoiding Repeated Allocations with Pre-allocation**

When the final size of a slice is known (or can be estimated), pre-allocating with `make([]T, 0, capacity)` avoids costly repeated allocations and copies during `append`. Without pre-allocation, Go doubles the backing array when capacity runs out, causing O(log n) allocations instead of one. This is one of the highest-impact optimizations in Go.

```
┌──────────────────────────────────────────────┐
│   Without Pre-allocation (repeated growth)  │
│                                              │
│   cap=1  [x]         → copy, alloc          │
│   cap=2  [x][x]      → copy, alloc          │
│   cap=4  [x][x][x][ ] → copy, alloc         │
│   cap=8  [x][x][x][x][x][ ][ ][ ] → ...    │
│   Total: O(log n) allocations + copies       │
│                                              │
│   With Pre-allocation (single alloc)        │
│                                              │
│   cap=10000  [ ][ ][ ]...[ ] → one alloc    │
│   append:    [x][x][x]...[ ] → no copies    │
│   Total: 1 allocation, 0 copies              │
└──────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // BAD — repeated allocations as slice grows
    bad := make([]int, 0)
    for i := 0; i < 10000; i++ {
        bad = append(bad, i)
    }

    // GOOD — single allocation
    good := make([]int, 0, 10000)
    for i := 0; i < 10000; i++ {
        good = append(good, i)
    }

    fmt.Println(len(bad), len(good))
}
```

### strings.Builder for Concatenation

**Tutorial: Efficient String Building with strings.Builder**

String concatenation with `+=` creates a new string on every iteration because Go strings are immutable — each concatenation copies all previous bytes. `strings.Builder` uses an internal `[]byte` buffer that grows efficiently, converting to string only once via `String()`. This reduces time complexity from O(n²) to O(n).

```
┌──────────────────────────────────────────┐
│    String += "x" (O(n²))                │
│                                          │
│    Iter 1: alloc "x"          (1 byte)   │
│    Iter 2: alloc "xx"         (2 bytes)  │
│    Iter 3: alloc "xxx"        (3 bytes)  │
│    ...                                   │
│    Total bytes copied: 1+2+3+...+n = n²/2│
│                                          │
│    strings.Builder (O(n))               │
│                                          │
│    ┌──────────────────────────┐          │
│    │ internal []byte buffer   │          │
│    │ [x][x][x][x][...][ ][ ] │          │
│    └──────────────────────────┘          │
│    .String() → single conversion         │
└──────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // BAD — O(n²) string concatenation
    s := ""
    for i := 0; i < 1000; i++ {
        s += "x" // creates new string each iteration
    }

    // GOOD — O(n) with Builder
    var sb strings.Builder
    for i := 0; i < 1000; i++ {
        sb.WriteString("x")
    }
    result := sb.String()

    fmt.Println(len(s), len(result)) // both 1000
}
```

### sync.Pool for Object Reuse

**Tutorial: Reducing GC Pressure with sync.Pool**

`sync.Pool` is a cache of temporary objects that reduces GC pressure by reusing allocations. You `Get()` an object from the pool (or create a new one via `New`), use it, `Reset()` it, and `Put()` it back. The pool may be cleared between GC cycles, so never rely on objects persisting. This is ideal for short-lived, frequently allocated objects like buffers.

```
┌───────────────────────────────────────────┐
│          sync.Pool Lifecycle             │
│                                           │
│  Get()                                    │
│   ├── Pool has item? → return it          │
│   └── Pool empty?    → call New()         │
│          │                                │
│          ▼                                │
│   ┌─────────────┐                         │
│   │  Use buffer  │ ◄── buf.WriteString()  │
│   └──────┬──────┘                         │
│          │                                │
│          ▼                                │
│   buf.Reset()  → clear contents           │
│   Put(buf)     → return to pool           │
│                                           │
│   ⚠ GC may clear pool at any time        │
└───────────────────────────────────────────┘
```

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data string) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    buf.WriteString("processed: ")
    buf.WriteString(data)
    return buf.String()
}

func main() {
    fmt.Println(process("hello"))
    fmt.Println(process("world"))
}
```

### Struct Field Ordering

**Tutorial: Minimizing Struct Padding with Field Ordering**

Go aligns struct fields according to their size, inserting padding bytes to ensure proper memory alignment. Ordering fields from largest to smallest minimizes padding waste. This can significantly reduce struct size — especially when your program allocates millions of structs. Use `unsafe.Sizeof()` to verify layout.

```
┌──────────────────────────────────────────────┐
│        Memory Layout Comparison             │
│                                              │
│  BadLayout (24 bytes):                       │
│  ┌──┬───────┬────────┬──┬─────┬────┬────┐   │
│  │a │padding│   b    │c │pad  │ d  │pad │   │
│  │1 │  7    │   8    │1 │ 3   │ 4  │    │   │
│  └──┴───────┴────────┴──┴─────┴────┴────┘   │
│                                              │
│  GoodLayout (16 bytes):                      │
│  ┌────────┬────┬──┬──┬────┐                  │
│  │   b    │ d  │a │c │pad │                  │
│  │   8    │ 4  │1 │1 │ 2  │                  │
│  └────────┴────┴──┴──┴────┘                  │
│                                              │
│  Rule: order fields large → small            │
└──────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "unsafe"
)

// BAD — 24 bytes (padding waste)
type BadLayout struct {
    a bool    // 1 + 7 padding
    b int64   // 8
    c bool    // 1 + 3 padding
    d int32   // 4
}

// GOOD — 16 bytes (minimal padding)
type GoodLayout struct {
    b int64   // 8
    d int32   // 4
    a bool    // 1
    c bool    // 1 + 2 padding
}

func main() {
    fmt.Println("BadLayout:", unsafe.Sizeof(BadLayout{}))   // 24
    fmt.Println("GoodLayout:", unsafe.Sizeof(GoodLayout{})) // 16
}
```

---

## GODEBUG Environment Variable

```bash
# Trace GC behavior:
GODEBUG=gctrace=1 ./myapp

# Trace scheduler:
GODEBUG=schedtrace=1000 ./myapp  # print every 1000ms

# More flags:
# allocfreetrace=1  — print every alloc/free
# cgocheck=2        — extra cgo pointer checks
# efence=1          — allocate each object on its own page (slow, for debugging)
```

---

## Inlining and BCE

```bash
# See inlining decisions:
go build -gcflags="-m" .
# Output like:
#   ./main.go:10:6: can inline add
#   ./main.go:15:6: inlining call to add

# See bounds check elimination:
go build -gcflags="-d=ssa/check_bce/debug=1" .
```

**Tutorial: Inlining and Bounds Check Elimination**

The Go compiler applies inlining and bounds check elimination (BCE) as optimizations. Small functions are inlined at call sites, removing function call overhead. BCE removes array/slice bounds checks when the compiler can prove access is safe (e.g., sequential iteration with `i < len(s)`). Use `-gcflags="-m"` to see what gets inlined.

```
┌──────────────────────────────────────────┐
│       Compiler Optimizations            │
│                                          │
│  Inlining:                               │
│  func add(a,b int) int { return a+b }   │
│       │                                  │
│       ▼  (compiler replaces call)        │
│  result := a + b  ◄── no function call   │
│                                          │
│  Bounds Check Elimination:               │
│  for i := 0; i < len(s); i++ {          │
│      s[i]  ◄── compiler proves i < len  │
│  }          ◄── bounds check removed     │
│                                          │
│  Check: go build -gcflags="-m" .        │
└──────────────────────────────────────────┘
```

```go
package main

func add(a, b int) int { // small function — likely inlined
    return a + b
}

func sum(s []int) int {
    total := 0
    for i := 0; i < len(s); i++ { // BCE: bounds check eliminated
        total += s[i]
    }
    return total
}

func main() {
    _ = add(1, 2) // inlined at call site
    _ = sum([]int{1, 2, 3, 4, 5})
}
```

---

## Interview Questions

1. **What profiling tools does Go provide?**
   - `pprof` for CPU, memory, goroutine, block, and mutex profiles. `go test -bench -cpuprofile`. `net/http/pprof` for live servers. `go tool trace` for execution traces. `go tool pprof` for analysis.

2. **How do you profile CPU usage in Go?**
   - Run `go test -bench=. -cpuprofile=cpu.prof` then `go tool pprof cpu.prof`. Or import `runtime/pprof` and call `pprof.StartCPUProfile(f)` / `pprof.StopCPUProfile()`. Use `top`, `list`, `web` commands.

3. **How do you profile memory allocations?**
   - `go test -bench=. -memprofile=mem.prof -memprofilerate=1`. Or use `net/http/pprof` endpoint. Analyze with `go tool pprof -alloc_space mem.prof`. Focus on `alloc_objects` vs `alloc_space`.

4. **What is `go tool trace`?**
   - Captures and visualizes execution events: goroutine scheduling, GC, syscalls, network. Run `go test -trace=trace.out` then `go tool trace trace.out`. Shows timeline in browser.

5. **How do you reduce allocations for better performance?**
   - Reuse buffers (`sync.Pool`), pre-allocate slices (`make([]T, 0, cap)`), use `strings.Builder`, avoid interface boxing, pass structs by pointer, use `append` wisely.

6. **What is benchmarking in Go and how do you write benchmarks?**
   - `func BenchmarkXxx(b *testing.B)` with a loop `for i := 0; i < b.N; i++`. b.N auto-adjusts. Use `b.ReportAllocs()` for allocation stats. `b.ResetTimer()` excludes setup.

7. **How do you detect and fix goroutine leaks?**
   - Monitor `runtime.NumGoroutine()`. Use `pprof` goroutine profile. Use `goleak` package in tests. Fix by ensuring all goroutines have exit paths via context, done channels, or closed channels.

8. **What is inlining and how does it affect performance?**
   - The compiler replaces small function calls with the function body. Check with `go build -gcflags='-m'`. Functions too complex (loops, defer) aren't inlined. `//go:noinline` disables it.

9. **What is the `race` detector and when should you use it?**
   - `go test -race` or `go run -race`. Instruments memory accesses at runtime to detect data races. ~10x slowdown. Use in CI and during development. Not for production.

10. **How do you optimize JSON performance in Go?**
    - Use streaming (`json.Decoder`) for large data. Use `json.RawMessage` to defer parsing. Consider `easyjson`, `jsoniter`, or `sonic` for faster encoding. Pre-generate code with `ffjson`.
