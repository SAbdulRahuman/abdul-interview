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
