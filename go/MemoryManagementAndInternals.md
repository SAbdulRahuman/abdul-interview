# Chapter 23 — Memory Management & Internals

Understanding how Go manages memory helps you write efficient programs and debug performance issues. Go uses a **garbage collector** (GC) and **escape analysis** to decide whether data lives on the stack or heap.

```
┌──────────────────────────────────────────────────────────┐
│              Stack vs Heap in Go                        │
│                                                          │
│  Stack (per goroutine):       Heap (shared):            │
│  ┌─────────────────┐          ┌──────────────────┐       │
│  │ Fast allocation  │          │ Slow allocation   │      │
│  │ (~2KB, grows)    │          │ GC managed         │     │
│  │ Auto-freed on    │          │ Survives function  │     │
│  │ function return  │          │ scope              │     │
│  │ No GC overhead   │          │ GC pauses (µs)     │     │
│  └─────────────────┘          └──────────────────┘       │
│                                                          │
│  Escape Analysis decides:                               │
│  func noEscape() {                                       │
│      x := 42         → stays on STACK (local, small)    │
│      _ = x                                               │
│  }                                                       │
│  func escapes() *int {                                   │
│      x := 42         → moved to HEAP (returned pointer) │
│      return &x                                           │
│  }                                                       │
│                                                          │
│  What causes escape to heap:                            │
│  • Returning pointer to local variable                  │
│  • Sending pointer into channel                         │
│  • Storing in interface value (dynamic type)            │
│  • Closure capturing pointer to local var               │
│  • Variable too large for stack                         │
│                                                          │
│  Check: go build -gcflags="-m" ./...                    │
└──────────────────────────────────────────────────────────┘
```

## Stack vs Heap Allocation

**Tutorial: Stack vs Heap — Where Does Your Data Live?**

This example contrasts two functions: one that returns a value (stays on the stack) and one that returns a pointer (escapes to the heap). The compiler uses escape analysis at build time to decide placement. Watch how `stackAllocation` returns a copy of `x` (stack-safe), while `heapAllocation` returns `&x`, forcing the compiler to allocate `x` on the heap so it survives the function return.

```
┌─────────────────────────────────────────────────┐
│           stackAllocation()                     │
│  ┌──────────────┐                               │
│  │ Stack Frame   │   x := 42                    │
│  │  x: 42       │──► return x (copy)            │
│  └──────────────┘   Frame freed ✓               │
│                                                 │
│           heapAllocation()                      │
│  ┌──────────────┐       ┌────────────┐          │
│  │ Stack Frame   │       │   Heap     │          │
│  │  x ─────────►│──────►│  x: 42     │          │
│  └──────────────┘       └────────────┘          │
│   Frame freed ✓          Survives! GC managed   │
│                          return &x (pointer)    │
└─────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// stackAllocation — stays on the stack (no escape)
func stackAllocation() int {
    x := 42 // allocated on stack, freed when function returns
    return x
}

// heapAllocation — escapes to the heap (returned pointer)
func heapAllocation() *int {
    x := 42 // compiler detects x escapes → allocates on heap
    return &x
}

func main() {
    a := stackAllocation()
    b := heapAllocation()
    fmt.Println(a, *b)
}
```

**Key differences:**
- **Stack**: fast, automatic cleanup on function return, goroutine-local
- **Heap**: managed by GC, slower allocation, survives function scope

---

## Escape Analysis

Use `go build -gcflags="-m"` to see what escapes to the heap.

```bash
$ go build -gcflags="-m" main.go
# main.go:12:2: moved to heap: x        ← heapAllocation's x escapes
# main.go:8:2: x does not escape          ← stackAllocation's x is stack-only
```

**Tutorial: Common Escape Scenarios in Practice**

This code demonstrates three common escape patterns: a slice that stays on the stack because it never leaves the function, a slice that escapes because it's returned, and a value that escapes because `fmt.Println` accepts `interface{}`. Understanding these patterns helps you write allocation-efficient code. Run `go build -gcflags="-m"` on your own code to see what the compiler decides.

```
┌────────────────────────────────────────────────────┐
│  noEscape()              escapeToHeap()            │
│  ┌────────────┐          ┌────────────┐            │
│  │ Stack      │          │ Stack      │            │
│  │ s []int    │          │ s ─────────┼──► Heap    │
│  │ (local use)│          │ (returned) │   []int    │
│  └────────────┘          └────────────┘            │
│  ✓ No escape             ✗ Escapes                 │
│                                                    │
│  escapeViaInterface()                              │
│  ┌────────────┐   fmt.Println(x)                   │
│  │ Stack      │        │                           │
│  │ x: 42 ────►│───► interface{} ──► Heap           │
│  └────────────┘   (boxing to interface)            │
│  ✗ Escapes (interface conversion)                  │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func noEscape() {
    s := make([]int, 100) // might stay on stack if compiler proves safe
    s[0] = 1
    _ = s
}

func escapeToHeap() []int {
    s := make([]int, 100) // escapes — returned to caller
    s[0] = 1
    return s
}

// Interface conversion can cause escape
func escapeViaInterface() {
    x := 42
    fmt.Println(x) // x escapes because fmt.Println takes interface{}
}

func main() {
    noEscape()
    _ = escapeToHeap()
    escapeViaInterface()
}
```

**Common causes of escape:**
1. Returning a pointer to a local variable
2. Sending to a channel
3. Assigning to interface (e.g., `fmt.Println`)
4. Closure captures
5. Large allocations (too big for stack)

---

## Garbage Collector

Go uses a **tri-color mark-and-sweep** concurrent garbage collector.

**Tutorial: Inspecting and Tuning the Garbage Collector**

This example shows how to read runtime memory statistics, force garbage collection, and tune GC behavior with `GOGC` and `GOMEMLIMIT`. The `runtime.MemStats` struct exposes allocation counts, heap size, and GC run counts. Adjusting `GOGC` trades memory usage for CPU time — higher values mean less frequent GC. `GOMEMLIMIT` (Go 1.19+) sets a soft cap on total memory, useful in containers.

```
┌──────────────────────────────────────────────────────┐
│         Tri-Color Mark & Sweep GC                    │
│                                                      │
│  Phase 1: Mark Setup (STW)                           │
│  ┌──────┐  Enable write barrier                      │
│  │ Root │                                            │
│  └──┬───┘                                            │
│     ▼                                                │
│  Phase 2: Concurrent Marking                         │
│  ┌─────┐    ┌─────┐    ┌─────┐                       │
│  │White│───►│Grey │───►│Black│                        │
│  │(?)  │    │(scan)│   │(done)│                       │
│  └─────┘    └─────┘    └─────┘                       │
│  unreached   queued    reachable                     │
│     ▼                                                │
│  Phase 3: Mark Termination (STW)                     │
│     ▼                                                │
│  Phase 4: Concurrent Sweep                           │
│  White objects ──► freed (reclaimed)                 │
│                                                      │
│  GOGC=100: GC when heap doubles    (default)         │
│  GOGC=200: GC when heap triples    (less GC)         │
│  GOMEMLIMIT=512MB: soft cap        (Go 1.19+)        │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
)

func main() {
    // Read memory stats
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Alloc: %d KB\n", m.Alloc/1024)
    fmt.Printf("TotalAlloc: %d KB\n", m.TotalAlloc/1024)
    fmt.Printf("Sys: %d KB\n", m.Sys/1024)
    fmt.Printf("NumGC: %d\n", m.NumGC)

    // Force garbage collection
    runtime.GC()

    // Tuning GOGC — sets the GC target percentage
    // GOGC=100 (default): GC runs when heap size doubles
    // GOGC=200: GC runs when heap triples (less frequent GC)
    // GOGC=50: GC runs when heap grows by 50% (more frequent GC)
    debug.SetGCPercent(200)

    // GOMEMLIMIT (Go 1.19+) — soft memory limit
    // Prevents excessive memory use, GC becomes more aggressive near limit
    debug.SetMemoryLimit(512 * 1024 * 1024) // 512 MB

    // Free OS memory
    debug.FreeOSMemory()
}
```

**GC phases:**
1. **Mark Setup** → stop the world briefly, enable write barrier
2. **Marking** → concurrent, trace reachable objects (tri-color: white/grey/black)
3. **Mark Termination** → stop the world briefly, finish marking
4. **Sweeping** → concurrent, reclaim unreachable (white) objects

---

## Memory Alignment & Struct Padding

**Tutorial: How Field Order Affects Struct Size**

The CPU requires data to be aligned to specific boundaries (e.g., `int64` on 8-byte boundaries). When struct fields aren't ordered by alignment, the compiler inserts invisible padding bytes to satisfy these constraints. This example shows that reordering the same four fields can save 8 bytes per struct — significant when allocating millions of instances. Use `unsafe.Sizeof` to verify struct sizes.

```
┌──────────────────────────────────────────────────────┐
│  BadStruct (24 bytes)        GoodStruct (16 bytes)   │
│  Offset  Field   Size       Offset  Field   Size     │
│  ┌────────────────────┐     ┌────────────────────┐   │
│  │ 0: a bool   [1]   │     │ 0: b int64  [8]   │   │
│  │ 1: padding  [7]   │     │ 8: d int32  [4]   │   │
│  │ 8: b int64  [8]   │     │12: a bool   [1]   │   │
│  │16: c bool   [1]   │     │13: c bool   [1]   │   │
│  │17: padding  [3]   │     │14: padding  [2]   │   │
│  │20: d int32  [4]   │     └────────────────────┘   │
│  └────────────────────┘     Total: 16 bytes ✓        │
│  Total: 24 bytes ✗          (saved 8 bytes!)         │
│                                                      │
│  Rule: Order fields largest → smallest alignment     │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "unsafe"
)

// BAD ordering — wasted padding
type BadStruct struct {
    a bool    // 1 byte + 7 bytes padding
    b int64   // 8 bytes
    c bool    // 1 byte + 3 bytes padding
    d int32   // 4 bytes
}
// Total: 24 bytes

// GOOD ordering — minimize padding
type GoodStruct struct {
    b int64   // 8 bytes
    d int32   // 4 bytes
    a bool    // 1 byte
    c bool    // 1 byte + 2 bytes padding
}
// Total: 16 bytes

func main() {
    fmt.Println("BadStruct size:", unsafe.Sizeof(BadStruct{}))   // 24
    fmt.Println("GoodStruct size:", unsafe.Sizeof(GoodStruct{})) // 16

    // Alignment of each field
    fmt.Println("Align of int64:", unsafe.Alignof(int64(0)))  // 8
    fmt.Println("Align of int32:", unsafe.Alignof(int32(0)))  // 4
    fmt.Println("Align of bool:", unsafe.Alignof(false))       // 1
}
```

**Rule of thumb:** Order struct fields from largest to smallest alignment.

---

## unsafe Package

**Tutorial: Low-Level Memory Inspection with unsafe**

The `unsafe` package lets you bypass Go's type system to inspect memory layout, compute field offsets, and reinterpret raw pointer bits. This example demonstrates `Sizeof`, `Alignof`, `Offsetof` for struct introspection, `unsafe.Pointer` for type-punning between pointer types, and `unsafe.Slice` for creating slices from raw pointers. These operations are inherently unsafe — they bypass compile-time type checks and can cause undefined behavior if misused.

```
┌────────────────────────────────────────────────────┐
│           unsafe Package Operations                │
│                                                    │
│  Sizeof(x)     ──► bytes occupied by value         │
│  Alignof(x)    ──► alignment requirement           │
│  Offsetof(f)   ──► byte offset of struct field     │
│                                                    │
│  unsafe.Pointer ──► generic pointer (bridges types)│
│                                                    │
│  *int64 ──► unsafe.Pointer ──► *float64            │
│   &i          (bridge)          (*float64)(p)      │
│   42      raw bits reinterpreted as float64        │
│                                                    │
│  unsafe.Slice(&arr[0], 3)                          │
│  [10, 20, 30, 40, 50]                              │
│   ▲──────────▲                                     │
│   ptr        len=3  ──► []int{10, 20, 30}          │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    // unsafe.Sizeof — size of value in bytes
    var x int64
    fmt.Println("Sizeof int64:", unsafe.Sizeof(x)) // 8

    // unsafe.Alignof — alignment requirement
    fmt.Println("Alignof int64:", unsafe.Alignof(x)) // 8

    // unsafe.Offsetof — offset of struct field
    type Example struct {
        A bool
        B int64
        C int32
    }
    fmt.Println("Offset of B:", unsafe.Offsetof(Example{}.B)) // 8

    // unsafe.Pointer — convert between pointer types
    // Only for VERY specific use cases (e.g., interop, performance-critical code)
    var i int64 = 42
    p := unsafe.Pointer(&i)
    fp := (*float64)(p) // reinterpret bits as float64
    fmt.Println("Reinterpreted:", *fp)

    // unsafe.Slice (Go 1.17+) — create slice from pointer + length
    arr := [5]int{10, 20, 30, 40, 50}
    slice := unsafe.Slice(&arr[0], 3)
    fmt.Println("Unsafe slice:", slice) // [10 20 30]
}
```

> **Warning:** `unsafe` bypasses Go's type safety. Avoid unless absolutely necessary (systems/interop code).

---

## cgo Basics

**Tutorial: Calling C Code from Go with cgo**

Go can call C functions directly using the `cgo` tool. You embed C code in a comment block immediately before `import "C"`, then call functions via the `C` pseudo-package. This example calls a C function, passes arguments, and handles C strings. Note that C memory (`C.CString`) must be freed manually — Go's garbage collector doesn't manage C allocations. Be aware of significant performance overhead: Go↔C calls are ~100x slower than Go↔Go calls.

```
┌────────────────────────────────────────────────────┐
│           cgo Call Flow                            │
│                                                    │
│  Go Code                    C Code                 │
│  ┌──────────────┐           ┌──────────────┐       │
│  │ main()       │           │ hello_from_c()│       │
│  │              │──cgo──►   │ printf(...)   │       │
│  │ C.hello()    │  bridge   │               │       │
│  │              │◄────────  │ return        │       │
│  │              │           └──────────────┘       │
│  │ C.add(3,4)  │──cgo──►   add(a,b) → 7          │
│  │              │◄────────  return 7               │
│  └──────────────┘                                  │
│                                                    │
│  C.CString("hello")  ──► malloc (C heap)           │
│  C.free(cStr)        ──► free   (manual!)          │
│  ⚠ Go GC does NOT manage C memory                 │
└────────────────────────────────────────────────────┘
```

```go
package main

/*
#include <stdio.h>
#include <stdlib.h>

void hello_from_c() {
    printf("Hello from C!\n");
}

int add(int a, int b) {
    return a + b;
}
*/
import "C"
import "fmt"

func main() {
    // Call C function
    C.hello_from_c()

    // Call C function with arguments
    result := C.add(3, 4)
    fmt.Println("C add(3, 4) =", result)

    // C string handling
    cStr := C.CString("Hello, cgo!") // allocates C memory
    defer C.free(C.CGoVoidPtr(cStr)) // must free manually!

    fmt.Println("cgo string done")
}
```

**cgo overhead:**
- Function calls between Go and C are ~100x slower than Go-to-Go calls
- Goroutine blocked in C uses a real OS thread
- Complicates cross-compilation

---

## Go Runtime — Goroutine Stacks

**Tutorial: Runtime Introspection — Goroutines, CPUs, and Scheduling**

This example uses the `runtime` package to inspect goroutine count, available CPUs, and `GOMAXPROCS` (the number of OS threads that can execute Go code simultaneously). It also demonstrates `runtime.Gosched()` to voluntarily yield the processor, and `runtime.Goexit()` to terminate a goroutine while still running deferred functions. Goroutine stacks start at ~2-8 KB and grow dynamically — enabling millions of concurrent goroutines.

```
┌────────────────────────────────────────────────────┐
│       Go Runtime: GMP Scheduler Model              │
│                                                    │
│  G = Goroutine    M = OS Thread    P = Processor   │
│                                                    │
│  ┌───┐ ┌───┐ ┌───┐    ┌───┐ ┌───┐                 │
│  │ G │ │ G │ │ G │    │ G │ │ G │  (many G's)     │
│  └─┬─┘ └─┬─┘ └─┬─┘    └─┬─┘ └─┬─┘                 │
│    └──────┼─────┘        └──┬──┘                    │
│       ┌───▼───┐         ┌───▼───┐                   │
│       │  P-0  │         │  P-1  │  (GOMAXPROCS Ps)  │
│       └───┬───┘         └───┬───┘                   │
│       ┌───▼───┐         ┌───▼───┐                   │
│       │  M-0  │         │  M-1  │  (OS threads)     │
│       └───────┘         └───────┘                   │
│                                                    │
│  Gosched()  ──► yield current G, let others run    │
│  Goexit()   ──► terminate G, defers still execute  │
│  Stack: 2-8KB initial ──► grows up to 1GB          │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // Goroutine stacks start small (~2-8 KB)
    // They grow dynamically (contiguous stack since Go 1.4)
    // Stack growth is transparent and automatic

    fmt.Println("NumGoroutine:", runtime.NumGoroutine())
    fmt.Println("NumCPU:", runtime.NumCPU())
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))

    // runtime.Gosched — yield processor
    go func() {
        runtime.Gosched() // let other goroutines run
        fmt.Println("After Gosched")
    }()

    // runtime.Goexit — terminate current goroutine
    // Deferred functions still run
    go func() {
        defer fmt.Println("Deferred in exiting goroutine")
        runtime.Goexit()
        fmt.Println("This never prints")
    }()

    runtime.Gosched()
}
```

---

## Memory Profiling

**Tutorial: Measuring Allocation Impact with MemStats**

This example captures `runtime.MemStats` snapshots before and after a known allocation workload, then compares them to measure total bytes allocated and malloc/free counts. This technique is useful for micro-benchmarking allocation-heavy code paths. For production profiling, use `pprof` which provides heap profiles, flame graphs, and goroutine analysis.

```
┌────────────────────────────────────────────────────┐
│        Memory Profiling Flow                       │
│                                                    │
│  ReadMemStats(&before)                             │
│        │                                           │
│        ▼                                           │
│  ┌──────────────────┐                              │
│  │ allocateSlices() │  1000 × make([]byte, 1024)   │
│  │  loop 1000x      │  = ~1MB total allocation     │
│  └──────────────────┘                              │
│        │                                           │
│        ▼                                           │
│  ReadMemStats(&after)                              │
│        │                                           │
│        ▼                                           │
│  Diff: TotalAlloc, Mallocs, Frees                  │
│  ┌────────────────────────────────┐                 │
│  │ Alloc diff ≈ 1024 KB          │                 │
│  │ Mallocs    ≈ 1000             │                 │
│  │ Frees      ≈ (GC dependent)   │                 │
│  └────────────────────────────────┘                 │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func allocateSlices() {
    for i := 0; i < 1000; i++ {
        s := make([]byte, 1024) // each allocation = 1KB
        _ = s
    }
}

func main() {
    var before, after runtime.MemStats

    runtime.ReadMemStats(&before)
    allocateSlices()
    runtime.ReadMemStats(&after)

    fmt.Printf("Alloc diff: %d KB\n", (after.TotalAlloc-before.TotalAlloc)/1024)
    fmt.Printf("Mallocs: %d\n", after.Mallocs-before.Mallocs)
    fmt.Printf("Frees: %d\n", after.Frees-before.Frees)
}
```

For production profiling, use `pprof` (see Chapter 26).

---

## Interview Questions

1. **How does Go's garbage collector work?**
   - Go uses a concurrent, tri-color mark-and-sweep GC. It runs concurrently with the program (low pause times). Tuned via `GOGC` (default 100 = GC when heap doubles). No generational collection.

2. **What is the difference between stack and heap allocation in Go?**
   - Local variables are stack-allocated (fast, auto-freed). If a variable escapes (returned pointer, stored in interface), it's heap-allocated (GC-managed). The compiler decides via escape analysis.

3. **What is escape analysis?**
   - The compiler analyzes whether a variable's lifetime extends beyond its scope. If it does, the variable "escapes" to the heap. Check with `go build -gcflags='-m'`. Reducing escapes improves performance.

4. **What is `GOGC` and how does it affect performance?**
   - `GOGC` sets the GC target percentage. Default 100 means GC triggers when heap grows 100% since last collection. Lower = more frequent GC (less memory). Higher = less frequent (more memory, less CPU).

5. **What is `GOMEMLIMIT` (Go 1.19+)?**
   - Sets a soft memory limit for the Go runtime. The GC works harder to stay under this limit. Helps prevent OOM in containerized environments. Complements `GOGC`.

6. **How does Go's scheduler (GMP model) work?**
   - G (goroutine), M (OS thread), P (processor/context). P holds a run queue. M executes G using P. `GOMAXPROCS` sets the number of Ps. Work-stealing balances load across Ps.

7. **What is a goroutine's stack size?**
   - Starts at 2-8 KB (much smaller than OS thread's ~1MB). Grows dynamically by copying to a larger allocation. Can grow to 1GB by default. This enables millions of concurrent goroutines.

8. **What is `unsafe.Pointer` and when should you use it?**
   - Allows bypassing Go's type system for raw memory access. Converts between pointer types and `uintptr`. Used in performance-critical code and runtime internals. Avoid in application code.

9. **What are finalizers in Go?**
   - `runtime.SetFinalizer(obj, func)` registers a function called when obj is garbage collected. Unreliable—no guarantee of when/if it runs. Use `defer` or explicit `Close()` instead.

10. **How do you diagnose memory leaks in Go?**
    - Use `pprof`: `go tool pprof http://localhost:6060/debug/pprof/heap`. Look for growing allocations. Common causes: unclosed resources, goroutine leaks, global caches, forgotten timers.
