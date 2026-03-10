# Chapter 30 — Garbage Collector (GC)

Go's GC is a **concurrent, tri-color mark-and-sweep** collector optimized for low latency (sub-millisecond pauses). Unlike Java, it's non-generational and non-compacting — objects don't move in memory.

```
┌──────────────────────────────────────────────────────────┐
│           Go GC — Tri-Color Mark-and-Sweep              │
│                                                          │
│  Phase 1: Mark Setup (STW ~10-30µs)                     │
│  └─ Enable write barrier, find roots                    │
│                                                          │
│  Phase 2: Concurrent Mark (runs with app)               │
│  └─ Traverse heap using tri-color algorithm:            │
│                                                          │
│     ┌───────┐     ┌───────┐     ┌───────┐               │
│     │ WHITE │────►│ GREY  │────►│ BLACK │               │
│     │ maybe │     │found  │     │fully  │               │
│     │garbage│     │not yet │     │scanned│               │
│     └───────┘     │scanned│     └───────┘               │
│       ▲           └───────┘       │                     │
│       │                           │                     │
│       └── remaining WHITE = garbage ─┘                  │
│                                                          │
│  Phase 3: Mark Termination (STW ~10-30µs)               │
│  └─ Finish marking, disable write barrier               │
│                                                          │
│  Phase 4: Concurrent Sweep (runs with app)              │
│  └─ Free WHITE objects, return pages to OS              │
│                                                          │
│  Tuning:                                                │
│  GOGC=100 (default) → GC when heap doubles              │
│  GOMEMLIMIT=1GiB    → hard memory ceiling (Go 1.19+)   │
│                                                          │
│  STW = Stop The World (all goroutines paused)           │
│  Total STW < 1ms for most applications                  │
└──────────────────────────────────────────────────────────┘
```

## GC Overview — Tri-Color Mark-and-Sweep

**Tutorial: Reading GC Runtime Statistics**

This example shows how to inspect the garbage collector's state using `runtime.ReadMemStats`. The `MemStats` struct gives you live data about heap usage, GC cycle count, and CPU fraction spent on garbage collection. These metrics are your first stop when diagnosing memory issues. Note that `ReadMemStats` briefly stops the world to get a consistent snapshot.

```
┌──────────────────────────────────────────────────────┐
│  runtime.ReadMemStats — GC Snapshot                  │
│                                                      │
│  Program Heap                                        │
│  ┌────────────────────────────────┐                   │
│  │ objects  objects  objects      │                   │
│  │ (live)   (live)   (garbage)    │                   │
│  └──────────────┬─────────────────┘                   │
│                 │                                     │
│                 ▼                                     │
│  runtime.ReadMemStats(&stats)                        │
│    ├── HeapAlloc   → currently allocated bytes       │
│    ├── NumGC       → completed GC cycles so far      │
│    └── GCCPUFraction → fraction of CPU used by GC    │
│                                                      │
│  Typical healthy values:                             │
│    GCCPUFraction < 0.05 (under 5%)                   │
│    NumGC grows slowly over time                      │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // Go uses a CONCURRENT, TRI-COLOR MARK-AND-SWEEP garbage collector.
    //
    // Key properties:
    // - Non-generational (no young/old gen like Java)
    // - Non-compacting (objects don't move in memory)
    // - Concurrent (runs mostly alongside the application)
    // - Low-latency (sub-millisecond pauses, typically <1ms)
    //
    // The GC's goal: reclaim heap memory no longer reachable from roots
    // (stack variables, globals, registers).

    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)
    fmt.Printf("HeapAlloc: %d KB\n", stats.HeapAlloc/1024)
    fmt.Printf("NumGC: %d\n", stats.NumGC)
    fmt.Printf("GCCPUFraction: %.4f\n", stats.GCCPUFraction)
}
```

---

## Tri-Color Algorithm Explained

**Tutorial: The Three Colors of Object Reachability**

The tri-color algorithm is the heart of Go's concurrent GC. Every heap object is logically colored WHITE (unknown/potentially garbage), GREY (discovered but not fully scanned), or BLACK (fully scanned and reachable). The GC iteratively promotes objects from white to grey to black. The critical invariant is that a BLACK object must never directly reference a WHITE object — the write barrier enforces this during concurrent marking.

```
┌──────────────────────────────────────────────────────┐
│  Tri-Color Marking — Step by Step                    │
│                                                      │
│  Step 1: All objects start WHITE                     │
│  ○ ○ ○ ○ ○ ○   (○ = white)                          │
│                                                      │
│  Step 2: Roots marked GREY                           │
│  ◐ ◐ ○ ○ ○ ○   (◐ = grey, found from stack/globals) │
│                                                      │
│  Step 3: Scan GREY, mark refs GREY, self BLACK       │
│  ● ● ◐ ◐ ○ ○   (● = black, fully scanned)          │
│                                                      │
│  Step 4: Repeat until no GREY remains                │
│  ● ● ● ● ○ ○                                        │
│                                                      │
│  Step 5: Remaining WHITE = garbage → free            │
│  ● ● ● ●       (○ objects reclaimed)                │
│                                                      │
│  Write Barrier (during concurrent mark):             │
│  BLACK obj ──ptr──► WHITE obj  ✗ FORBIDDEN           │
│  Barrier intercepts write, marks target GREY         │
└──────────────────────────────────────────────────────┘
```

```go
package main

// The tri-color algorithm classifies objects into three sets:
//
// WHITE — Potentially garbage. Not yet visited by the GC.
//         At the end of marking, white objects are collected.
//
// GREY  — Discovered but not fully scanned. The GC has found this
//         object is reachable but hasn't checked its references yet.
//
// BLACK — Fully scanned. This object is reachable AND all its
//         direct references have been discovered (turned grey).
//
// Algorithm steps:
// 1. Initially all objects are WHITE
// 2. GC roots (stack, globals) are marked GREY
// 3. Pick a GREY object, scan its pointers:
//    - Mark referenced WHITE objects as GREY
//    - Mark the scanned object as BLACK
// 4. Repeat step 3 until no GREY objects remain
// 5. All remaining WHITE objects are unreachable → free them
//
// Invariant (tri-color invariant):
//   A BLACK object must NEVER point directly to a WHITE object.
//   This is maintained using a WRITE BARRIER.

// Why concurrent GC needs a write barrier:
// While the GC is marking, the application (mutator) keeps running.
// If the mutator stores a pointer from a BLACK object to a WHITE object,
// the invariant would break. The write barrier intercepts pointer writes
// and ensures the referenced object gets marked GREY.

func main() {
    // The write barrier is active only during the marking phase.
    // Outside of GC marking, pointer writes have zero overhead.
}
```

---

## GC Phases

**Tutorial: The Four Phases of a GC Cycle**

Each GC cycle runs through four distinct phases. Two are brief Stop-the-World (STW) pauses (mark setup and mark termination, typically <100μs each), while marking and sweeping run concurrently with the application. The GC uses ~25% of CPU during concurrent marking. If a goroutine allocates faster than the GC can mark, it gets drafted into "mark assist" duty to prevent heap runaway.

```
┌──────────────────────────────────────────────────────┐
│  GC Cycle Timeline                                   │
│                                                      │
│  ──time──►                                           │
│                                                      │
│  App:  ████████░░██████████████░░████████████████     │
│  GC:          ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓                    │
│         ▲  ▲                    ▲  ▲                 │
│         │  │                    │  │                 │
│  Phase: 1  2                    3  4                 │
│                                                      │
│  1 ─ Mark Setup     (STW ~10-30μs)                   │
│      └─ enable write barrier, find roots             │
│                                                      │
│  2 ─ Concurrent Mark (runs with app, ~25% CPU)       │
│      └─ trace refs, mark assist if allocating fast   │
│                                                      │
│  3 ─ Mark Termination (STW ~10-30μs)                 │
│      └─ finish marking, disable write barrier        │
│                                                      │
│  4 ─ Concurrent Sweep (lazy, as spans needed)        │
│      └─ free WHITE objects, no STW                   │
│                                                      │
│  ████ = app running   ░░ = STW   ▓▓ = GC concurrent  │
└──────────────────────────────────────────────────────┘
```

```go
package main

// Go's GC runs in these phases:
//
// Phase 1: MARK SETUP (Stop-the-World — very brief)
//   - Enable write barrier
//   - All goroutines are briefly paused (typically <100μs)
//   - GC roots are identified
//
// Phase 2: MARKING (Concurrent)
//   - GC goroutines trace references from roots
//   - Runs CONCURRENTLY with the application
//   - Uses ~25% of available CPU (controlled by GOMAXPROCS)
//   - Write barrier is active — pointer writes are intercepted
//   - Mark assist: if a goroutine allocates too fast, it's forced
//     to help with marking (prevents outpacing the GC)
//
// Phase 3: MARK TERMINATION (Stop-the-World — very brief)
//   - Brief pause to finish marking
//   - Disable write barrier
//   - Clean up GC state
//   - Typically <100μs
//
// Phase 4: SWEEPING (Concurrent)
//   - Reclaim memory from WHITE (unreachable) objects
//   - Runs concurrently, triggered lazily as spans are needed
//   - No Stop-the-World pause for sweeping
//
// Total STW time per GC cycle: typically <1ms (often <200μs)

func main() {}
```

---

## GOGC — GC Trigger Percentage

**Tutorial: Controlling GC Frequency with GOGC**

GOGC sets the percentage of new heap growth that triggers the next GC cycle. The default `GOGC=100` means the GC runs when the heap has doubled since the last collection. Increasing GOGC (e.g., 200) reduces GC frequency at the cost of more memory, while decreasing it (e.g., 50) increases frequency to save memory at the cost of CPU. Use `debug.SetGCPercent` at runtime or set the `GOGC` environment variable.

```
┌──────────────────────────────────────────────────────┐
│  GOGC — Heap Growth Trigger                          │
│                                                      │
│  Live heap after last GC = 100 MB                    │
│                                                      │
│  GOGC=50:   next GC at 150 MB  (100 + 50%)          │
│  ├──────────┤·····┤                                  │
│  100MB       150MB                                   │
│                                                      │
│  GOGC=100:  next GC at 200 MB  (100 + 100%)         │
│  ├──────────┤···········┤                            │
│  100MB                 200MB                         │
│                                                      │
│  GOGC=200:  next GC at 300 MB  (100 + 200%)         │
│  ├──────────┤·····················┤                  │
│  100MB                          300MB                │
│                                                      │
│  Trade-off:                                          │
│  Higher GOGC ──► less GC CPU, more memory            │
│  Lower  GOGC ──► more GC CPU, less memory            │
│  GOGC=off    ──► disable (use with GOMEMLIMIT)       │
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
    // GOGC controls how often GC runs.
    // It sets the percentage of NEW heap growth that triggers GC.
    //
    // Default: GOGC=100
    // Meaning: GC triggers when heap grows 100% over the "live" heap
    //          (heap occupied after last GC).
    //
    // Example: if live heap = 100MB after last GC,
    //          next GC triggers at ~200MB total heap.

    // Read current GOGC
    prev := debug.SetGCPercent(100)
    fmt.Println("Previous GOGC:", prev)

    // Higher GOGC = less frequent GC = more memory, less CPU
    debug.SetGCPercent(200) // GC at 200% growth → uses more memory, less CPU

    // Lower GOGC = more frequent GC = less memory, more CPU
    debug.SetGCPercent(50) // GC at 50% growth → uses less memory, more CPU

    // GOGC=off — disable GC entirely (use with GOMEMLIMIT)
    // debug.SetGCPercent(-1) // disables GC

    // Set via environment variable:
    //   GOGC=200 ./myapp
    //   GOGC=off ./myapp

    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)
    fmt.Printf("HeapAlloc: %d KB\n", stats.HeapAlloc/1024)
    fmt.Printf("NextGC target: %d KB\n", stats.NextGC/1024)
}
```

---

## GOMEMLIMIT — Soft Memory Limit (Go 1.19+)

**Tutorial: Setting a Soft Memory Ceiling**

GOMEMLIMIT (Go 1.19+) tells the GC to work harder to keep total memory under a target. It is a soft limit — Go can briefly exceed it, but the GC increases frequency to bring usage back down. The killer pattern for containers is `GOGC=off` + `GOMEMLIMIT=<value>`, which maximizes throughput within a memory budget. Always leave ~10% headroom below the container's hard memory limit for non-Go allocations (cgo, OS overhead).

```
┌──────────────────────────────────────────────────────┐
│  GOMEMLIMIT — Soft Memory Ceiling                    │
│                                                      │
│  Container Memory: 512 MB (hard kill at 512)         │
│  GOMEMLIMIT:       450 MiB (soft target)             │
│                                                      │
│  Memory ▲                                            │
│  512 MB ┤ ─ ─ ─ ─ ─ ─ container OOM kill ─ ─ ─      │
│         │                                            │
│  450 MB ┤═══════════ GOMEMLIMIT ═══════════          │
│         │     ╱╲    ╱╲                               │
│         │    ╱  ╲  ╱  ╲   ◄── GC works harder here  │
│         │   ╱    ╲╱    ╲                             │
│         │  ╱              ╲                          │
│    0 MB ┤─╱────────────────╲──────── time ──►        │
│         └────────────────────────────────────         │
│                                                      │
│  GOGC=off + GOMEMLIMIT pattern:                      │
│  • GC only runs when approaching the limit           │
│  • Maximizes throughput in memory-bounded envs       │
│  • GC still runs — just triggered by limit, not %    │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "math"
    "runtime/debug"
)

func main() {
    // GOMEMLIMIT sets a soft memory limit for the Go runtime.
    // The GC works harder to stay under this limit.
    //
    // This is NOT a hard limit — Go can exceed it temporarily,
    // but the GC will increase its frequency to bring memory down.
    //
    // Default: math.MaxInt64 (no limit)

    // Set programmatically
    limit := debug.SetMemoryLimit(512 * 1024 * 1024) // 512 MB
    fmt.Printf("Previous limit: %d bytes\n", limit)

    // Read current limit
    current := debug.SetMemoryLimit(math.MaxInt64) // reset to no limit
    _ = current

    // Set via environment variable (recommended):
    //   GOMEMLIMIT=512MiB ./myapp
    //   GOMEMLIMIT=1GiB ./myapp

    // Supported units: B, KiB, MiB, GiB, TiB

    // Best practice for containers:
    //   Container memory limit = 512MB
    //   GOMEMLIMIT = 450MiB (leave headroom for non-Go memory)
    //   GOGC = 100 (or even GOGC=off with GOMEMLIMIT)

    // GOGC=off + GOMEMLIMIT pattern:
    //   - Disables percentage-based GC triggering
    //   - GC only runs when approaching the memory limit
    //   - Maximizes throughput in memory-constrained environments
    //   - The GC will still run — it's triggered by GOMEMLIMIT

    fmt.Println("GOMEMLIMIT is a soft target, not a hard cap")
}
```

---

## GC Pacing — How the GC Decides When to Run

**Tutorial: The GC Pacer Algorithm**

The GC pacer is the internal algorithm that decides exactly when to start each GC cycle and how many resources to devote to it. It balances allocation rate against marking speed to complete marking before the heap exceeds the trigger target. When a goroutine allocates too fast, it is forced into "mark assist" — it must help the GC mark objects before it can proceed with its own allocation. This prevents any single hot goroutine from outpacing the collector.

```
┌──────────────────────────────────────────────────────┐
│  GC Pacer Decision Flow                              │
│                                                      │
│  Inputs:                                             │
│  ┌──────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ Live heap    │  │ GOGC /     │  │ Allocation   │  │
│  │ after GC     │  │ GOMEMLIMIT │  │ rate         │  │
│  └──────┬───────┘  └─────┬──────┘  └──────┬───────┘  │
│         │                │                │          │
│         └────────┬───────┘────────────────┘          │
│                  ▼                                   │
│         ┌────────────────┐                           │
│         │   GC Pacer     │                           │
│         └───┬────────┬───┘                           │
│             │        │                               │
│             ▼        ▼                               │
│  ┌──────────────┐  ┌────────────────────┐            │
│  │ When to start│  │ Mark assist needed?│            │
│  │ next GC      │  │ (goroutine drafted │            │
│  └──────────────┘  │  to help marking)  │            │
│                    └────────────────────┘            │
└──────────────────────────────────────────────────────┘
```

```go
package main

// The GC pacer determines when to start a new GC cycle.
//
// Goal: complete marking before the heap reaches the trigger target
//       while using minimal CPU.
//
// Inputs to the pacer:
// 1. Live heap size after last GC
// 2. GOGC percentage
// 3. GOMEMLIMIT (if set)
// 4. Rate of allocation
// 5. Rate of marking (how fast GC can scan)
//
// The pacer dynamically adjusts:
// - When to start the next GC cycle
// - How many GC workers to use (background marking)
// - Whether goroutines need "mark assist" duty

// Mark Assist:
// If a goroutine is allocating faster than GC can mark,
// the runtime forces it to assist with marking before
// allowing the allocation. This prevents runaway heap growth.
//
// Observe mark assist in traces:
//   go test -trace=trace.out && go tool trace trace.out
//   Look for "GC mark assist" spans on goroutine timelines.

func main() {}
```

---

## runtime.GC() — Forcing Garbage Collection

**Tutorial: Manually Triggering a GC Cycle**

Calling `runtime.GC()` forces an immediate, complete garbage collection cycle. This is rarely needed in production since the GC pacer handles timing automatically. It's useful in benchmarks (to get a clean starting state), tests (to verify finalizers), or after releasing a very large batch of data. Watch the before/after `HeapAlloc` values to see how much memory was reclaimed. Avoid calling this in regular application code — it usually degrades performance.

```
┌──────────────────────────────────────────────────────┐
│  runtime.GC() — Manual Collection                    │
│                                                      │
│  ┌───────────────────────────────────┐               │
│  │ Heap before GC                    │               │
│  │ ████████████░░░░░░░░░             │               │
│  │ live data    garbage              │               │
│  └───────────────────────────────────┘               │
│         │                                            │
│    data = nil    ◄── remove reference                │
│    runtime.GC()  ◄── force collection                │
│         │                                            │
│         ▼                                            │
│  ┌───────────────────────────────────┐               │
│  │ Heap after GC                     │               │
│  │ ████████████                      │               │
│  │ live data only                    │               │
│  └───────────────────────────────────┘               │
│                                                      │
│  Use for: benchmarks, tests, post-bulk-release       │
│  Avoid:   regular app code (trust the pacer)         │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // runtime.GC() forces a full garbage collection cycle.
    // Rarely needed — the GC is designed to run automatically.

    var stats runtime.MemStats

    // Allocate some memory
    data := make([]byte, 10*1024*1024) // 10 MB
    _ = data

    runtime.ReadMemStats(&stats)
    fmt.Printf("Before GC — HeapAlloc: %d MB\n", stats.HeapAlloc/1024/1024)

    data = nil      // remove reference
    runtime.GC()    // force GC

    runtime.ReadMemStats(&stats)
    fmt.Printf("After GC  — HeapAlloc: %d MB\n", stats.HeapAlloc/1024/1024)
    fmt.Printf("NumGC: %d\n", stats.NumGC)

    // When runtime.GC() might be appropriate:
    // - Benchmarks: ensure consistent starting state
    // - Tests: verify finalizers run
    // - After releasing a large batch of memory (rare)
    //
    // When NOT to use:
    // - Regular application code (trust the GC pacer)
    // - Performance optimization (it usually makes things worse)
}
```

---

## runtime.MemStats — GC Metrics

**Tutorial: Deep Dive into Memory Statistics**

The `runtime.MemStats` struct provides a comprehensive view of the Go runtime's memory state. Key fields include `HeapAlloc` (live heap bytes), `HeapSys` (total heap from OS), `HeapIdle`/`HeapInuse` (unused vs active spans), `NumGC` (cycle count), `PauseTotalNs` (cumulative STW time), and `GCCPUFraction`. Note that `ReadMemStats` briefly stops the world, so call it sparingly in production — prefer `debug.ReadGCStats` for lighter-weight GC info.

```
┌──────────────────────────────────────────────────────┐
│  runtime.MemStats — Key Fields                       │
│                                                      │
│  OS Memory (Sys)                                     │
│  ┌──────────────────────────────────────────────┐    │
│  │  HeapSys (heap from OS)                      │    │
│  │  ┌──────────────────┬───────────────────┐    │    │
│  │  │  HeapInuse       │    HeapIdle        │    │    │
│  │  │  (active spans)  │  (unused spans)    │    │    │
│  │  │  ┌────────────┐  │  ┌──────────────┐  │    │    │
│  │  │  │ HeapAlloc  │  │  │ HeapReleased │  │    │    │
│  │  │  │ (live obj) │  │  │ (back to OS) │  │    │    │
│  │  │  └────────────┘  │  └──────────────┘  │    │    │
│  │  └──────────────────┴───────────────────┘    │    │
│  │                                              │    │
│  │  StackInuse / StackSys — goroutine stacks    │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  GC metrics: NumGC, PauseTotalNs, GCCPUFraction      │
│  Alloc metrics: TotalAlloc, Mallocs, Frees           │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)

    // Heap metrics
    fmt.Printf("HeapAlloc:    %d MB  (currently allocated heap)\n", stats.HeapAlloc/1024/1024)
    fmt.Printf("HeapSys:      %d MB  (heap obtained from OS)\n", stats.HeapSys/1024/1024)
    fmt.Printf("HeapIdle:     %d MB  (heap spans not in use)\n", stats.HeapIdle/1024/1024)
    fmt.Printf("HeapInuse:    %d MB  (heap spans in use)\n", stats.HeapInuse/1024/1024)
    fmt.Printf("HeapReleased: %d MB  (heap released to OS)\n", stats.HeapReleased/1024/1024)
    fmt.Printf("HeapObjects:  %d     (live heap objects)\n", stats.HeapObjects)

    // GC metrics
    fmt.Printf("NumGC:        %d     (completed GC cycles)\n", stats.NumGC)
    fmt.Printf("PauseTotalNs: %d ms  (total STW pause time)\n", stats.PauseTotalNs/1_000_000)
    fmt.Printf("GCCPUFraction: %.4f (fraction of CPU used by GC)\n", stats.GCCPUFraction)
    fmt.Printf("NextGC:       %d MB  (target heap for next GC)\n", stats.NextGC/1024/1024)

    // Last pause duration
    if stats.NumGC > 0 {
        lastPause := stats.PauseNs[(stats.NumGC+255)%256]
        fmt.Printf("Last GC pause: %d μs\n", lastPause/1000)
    }

    // Allocation metrics
    fmt.Printf("TotalAlloc:   %d MB  (cumulative bytes allocated)\n", stats.TotalAlloc/1024/1024)
    fmt.Printf("Mallocs:      %d     (cumulative alloc count)\n", stats.Mallocs)
    fmt.Printf("Frees:        %d     (cumulative free count)\n", stats.Frees)

    // System memory
    fmt.Printf("Sys:          %d MB  (total memory from OS)\n", stats.Sys/1024/1024)

    // Stack metrics
    fmt.Printf("StackInuse:   %d KB  (stack memory in use)\n", stats.StackInuse/1024)
    fmt.Printf("StackSys:     %d KB  (stack memory from OS)\n", stats.StackSys/1024)
}
```

---

## GODEBUG=gctrace — GC Trace Logging

**Tutorial: Interpreting GC Trace Output**

Setting `GODEBUG=gctrace=1` when launching your application prints one line per GC cycle to stderr. This is the most accessible way to observe GC behavior without adding instrumentation to your code. The trace shows STW pause durations, concurrent mark time, heap sizes before/after, CPU usage breakdown (assist, background, idle marking), and the GC goal. Learn to read these numbers to diagnose whether your app is GC-bound.

```
┌──────────────────────────────────────────────────────┐
│  gctrace output anatomy:                             │
│                                                      │
│  gc 1 @0.012s 2%: 0.015+0.41+0.008 ms clock         │
│  │  │  │     │    │     │    │                       │
│  │  │  │     │    │     │    └─ STW mark termination │
│  │  │  │     │    │     └── concurrent mark (wall) │
│  │  │  │     │    └───── STW mark setup          │
│  │  │  │     └───────── GC CPU %                │
│  │  │  └─────────────── time since start        │
│  │  └──────────────────── cycle number            │
│                                                      │
│  4->4->1 MB, 4 MB goal, 8 P                         │
│  │  │  │       │          │                         │
│  │  │  │       │          └─ GOMAXPROCS             │
│  │  │  │       └────────── heap target              │
│  │  │  └───────────────── live heap after mark     │
│  │  └──────────────────── heap at end of mark      │
│  └─────────────────────── heap at start of mark    │
└──────────────────────────────────────────────────────┘
```

```go
package main

// Run your application with GC tracing enabled:
//
//   GODEBUG=gctrace=1 ./myapp
//
// Output format (one line per GC cycle):
//
//   gc 1 @0.012s 2%: 0.015+0.41+0.008 ms clock,
//   0.12+0.25/0.33/0+0.066 ms cpu, 4->4->1 MB,
//   4 MB goal, 0 MB stacks, 0 MB globals, 8 P
//
// Reading the trace:
//   gc 1          — GC cycle number
//   @0.012s       — time since program start
//   2%            — percentage of CPU used by GC
//   0.015+0.41+0.008 ms clock:
//     0.015 ms    — STW mark setup (sweep termination)
//     0.41 ms     — concurrent marking (wall clock)
//     0.008 ms    — STW mark termination
//   0.12+0.25/0.33/0+0.066 ms cpu:
//     0.12 ms     — STW sweep termination CPU
//     0.25 ms     — GC mark assist CPU (goroutines helping)
//     0.33 ms     — GC background mark CPU
//     0 ms        — GC idle mark CPU
//     0.066 ms    — STW mark termination CPU
//   4->4->1 MB:
//     4 MB        — heap size at start of marking
//     4 MB        — heap size at end of marking
//     1 MB        — live heap after marking
//   4 MB goal     — target heap size for this cycle
//   8 P           — number of processors (GOMAXPROCS)

func main() {
    // Also useful: GODEBUG=gcpacertrace=1 for pacer decisions
    // Combine: GODEBUG=gctrace=1,gcpacertrace=1 ./myapp
}
```

---

## Escape Analysis — Stack vs Heap

**Tutorial: How the Compiler Decides Stack vs Heap Allocation**

Escape analysis is the compiler's decision process for whether a variable stays on the stack (fast, no GC) or escapes to the heap (GC-managed). A variable escapes if its address outlives the function: returning a pointer, passing to an `interface{}` parameter, or being captured by an escaping closure. Run `go build -gcflags="-m"` to see escape decisions. Reducing escapes is the most effective way to lower GC pressure.

```
┌──────────────────────────────────────────────────────┐
│  Escape Analysis — Stack vs Heap Decision             │
│                                                      │
│  Stack (fast, auto-freed)    Heap (GC-managed)       │
│  ┌──────────────────┐      ┌──────────────────┐    │
│  │  x := 42        │      │  x := 42        │    │
│  │  return x        │      │  return &x       │    │
│  │                  │      │  ▲ escapes!      │    │
│  │  ✔ stays on stack│      │  ptr outlives fn │    │
│  └──────────────────┘      └──────────────────┘    │
│                                                      │
│  Common escape causes:                               │
│  1. return &localVar      ◄── pointer escapes       │
│  2. fmt.Println(x)        ◄── boxed to interface{}  │
│  3. closure captures var   ◄── closure escapes      │
│  4. large/dynamic slice    ◄── can't prove size     │
│  5. store in global var    ◄── outlives function    │
│                                                      │
│  Check: go build -gcflags="-m" .                     │
└──────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Check escape analysis:
//   go build -gcflags="-m" .
//   go build -gcflags="-m -m" .   (more verbose)

// Stack allocation: fast, auto-freed when function returns
// Heap allocation: GC-managed, slower

//go:noinline
func stackAllocation() int {
    x := 42 // stays on stack — doesn't escape
    return x
}

//go:noinline
func heapAllocation() *int {
    x := 42 // ESCAPES to heap — returned pointer outlives function
    return &x
}

//go:noinline
func interfaceEscape() {
    x := 42
    fmt.Println(x) // x escapes — passed to interface{} parameter
    // fmt.Println accepts any (interface{}), which boxes the value
}

//go:noinline
func sliceEscape() {
    s := make([]int, 0, 100) // may escape depending on usage
    s = append(s, 1)
    _ = s
}

//go:noinline
func closureCapture() func() int {
    x := 42 // escapes — captured by closure that outlives the function
    return func() int {
        return x
    }
}

func main() {
    _ = stackAllocation()
    _ = heapAllocation()
    interfaceEscape()
    sliceEscape()
    _ = closureCapture()

    // Common causes of escape:
    // 1. Returning a pointer to a local variable
    // 2. Passing to interface{}/any parameter (fmt.Println, etc.)
    // 3. Variable captured by a closure that escapes
    // 4. Slice/map grows beyond known-at-compile-time size
    // 5. Storing in a package-level variable
    //
    // Reducing escapes → fewer heap allocations → less GC work
}
```

---

## Finalizers — runtime.SetFinalizer

**Tutorial: Cleanup Callbacks Before Collection**

Finalizers are functions called by the GC just before an object is collected. You register them with `runtime.SetFinalizer(obj, cleanupFunc)`. However, finalizers have significant caveats: they defer collection by one cycle, aren't guaranteed to run before program exit, and can accidentally "resurrect" objects. The idiomatic Go pattern is explicit `Close()` methods with `defer`, using finalizers only as a safety net for missed cleanup.

```
┌──────────────────────────────────────────────────────┐
│  Finalizer Lifecycle                                 │
│                                                      │
│  ┌──────────────────┐                                  │
│  │  obj := &Res{}  │                                  │
│  │  SetFinalizer(  │                                  │
│  │    obj, cleanup)│                                  │
│  └────────┬─────────┘                                  │
│           │                                           │
│           ▼                                           │
│  obj = nil (remove reference)                        │
│           │                                           │
│           ▼                                           │
│  GC Cycle 1: ──► runs finalizer, obj NOT freed yet    │
│           │                                           │
│           ▼                                           │
│  GC Cycle 2: ──► object actually freed               │
│                                                      │
│  Best practice:                                      │
│  • Use defer obj.Close() for cleanup                 │
│  • Finalizers = safety net, not primary cleanup      │
│  • SetFinalizer(obj, nil) removes finalizer           │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

type Resource struct {
    Name string
}

func main() {
    // Finalizers run when an object is about to be garbage collected.
    // They are NOT guaranteed to run — and have caveats.

    r := &Resource{Name: "db-connection"}

    runtime.SetFinalizer(r, func(res *Resource) {
        fmt.Printf("Finalizer called for: %s\n", res.Name)
        // Cleanup: close file, release handle, etc.
    })

    r = nil           // remove reference
    runtime.GC()      // force GC to trigger finalizer
    time.Sleep(100 * time.Millisecond)

    // Finalizer caveats:
    // 1. Not guaranteed to run before program exits
    // 2. Order is not guaranteed
    // 3. Finalizer prevents collection in first GC cycle —
    //    object is collected in the NEXT cycle after finalizer runs
    // 4. Finalizer can "resurrect" an object (make it reachable again)
    // 5. Only one finalizer per object — last SetFinalizer wins
    // 6. SetFinalizer(obj, nil) removes the finalizer
    //
    // Best practice:
    // - Use explicit Close() methods instead of finalizers
    // - Use defer obj.Close() for cleanup
    // - Finalizers are a safety net, not the primary cleanup mechanism
}
```

---

## GC Tuning Strategies

**Tutorial: Practical GC Optimization Playbook**

GC tuning follows a strict priority order: measure first, reduce allocations second, then tune knobs. The most effective optimization is always reducing allocations — using `sync.Pool`, pre-allocating slices, avoiding interface boxing in hot paths. GOGC and GOMEMLIMIT are coarse-grained tools for trading memory against CPU. The ballast technique (pre-GOMEMLIMIT) raised the live heap baseline to reduce GC frequency. For most applications, Go's default GC settings work well without tuning.

```
┌──────────────────────────────────────────────────────┐
│  GC Tuning Priority (most to least effective)        │
│                                                      │
│  1. MEASURE                                          │
│     └► GODEBUG=gctrace=1, pprof, go tool trace       │
│              │                                       │
│              ▼                                       │
│  2. REDUCE ALLOCATIONS  (biggest impact)             │
│     ├► sync.Pool for buffer reuse                    │
│     ├► make([]T, 0, cap) pre-allocate               │
│     ├► strings.Builder not + concatenation           │
│     ├► []T instead of []*T                          │
│     └► avoid boxing in hot paths                     │
│              │                                       │
│              ▼                                       │
│  3. TUNE GOGC / GOMEMLIMIT                          │
│     ├► GOGC=200 → half as many GC cycles             │
│     └► GOMEMLIMIT=450MiB (90% of container)         │
│              │                                       │
│              ▼                                       │
│  4. ADVANCED (rarely needed)                         │
│     ├► GOGC=off + GOMEMLIMIT                        │
│     └► Off-heap / arena (experimental)               │
└──────────────────────────────────────────────────────┘
```

```go
package main

// Practical GC tuning guidelines:
//
// 1. MEASURE FIRST
//    - Use GODEBUG=gctrace=1 to see GC frequency and pause times
//    - Use pprof heap profile to find allocation hotspots
//    - Use go tool trace to see GC impact on latency
//
// 2. REDUCE ALLOCATIONS (most effective)
//    - Reuse objects with sync.Pool
//    - Pre-allocate slices: make([]T, 0, estimatedCap)
//    - Use strings.Builder instead of string concatenation
//    - Avoid unnecessary pointer indirection
//    - Use value types instead of pointers when struct is small
//    - Avoid boxing into interface{} in hot paths
//    - Use arrays instead of slices for fixed-size data
//
// 3. TUNE GOGC
//    - GOGC=200: halve GC frequency, double memory usage
//    - GOGC=50: double GC frequency, halve memory usage
//    - Trade-off: CPU time vs memory
//
// 4. USE GOMEMLIMIT (Go 1.19+)
//    - Set to ~90% of container memory limit
//    - Prevents OOM in containerized environments
//    - GOGC=off + GOMEMLIMIT for maximum throughput
//
// 5. BALLAST TECHNIQUE (pre-GOMEMLIMIT)
//    - Allocate a large unused byte slice at startup:
//      var ballast = make([]byte, 1<<30) // 1 GB
//    - Raises the "live heap" baseline, reducing GC frequency
//    - Superseded by GOMEMLIMIT in Go 1.19+
//
// 6. OFF-HEAP / ARENA (experimental)
//    - arena package (Go 1.20 experimental) for manual management
//    - mmap for large datasets outside GC's view
//    - Use with caution — Go's GC is usually sufficient
//
// Remember: Go's GC is designed for low latency, not maximum throughput.
// If you need throughput-oriented GC, consider Java or C#.
// If you need no GC, consider Rust or C.

func main() {}
```

---

## GC-Friendly Code Patterns

**Tutorial: Writing Code That Minimizes GC Pressure**

These five patterns reduce heap allocations and GC work. `sync.Pool` recycles temporary objects (like buffers) across GC cycles instead of allocating fresh ones. Pre-allocating slice capacity avoids repeated grow-and-copy operations. Value receivers on small structs keep them on the stack. Using `[]Item` instead of `[]*Item` creates one contiguous allocation with better cache locality. Struct of Arrays (SoA) layout improves batch-processing throughput over Array of Structs (AoS).

```
┌──────────────────────────────────────────────────────┐
│  GC-Friendly Patterns                                │
│                                                      │
│  Pattern 1: sync.Pool                                │
│  ┌───────────┐  Get   ┌──────────┐  Put   ┌─────────┐  │
│  │sync.Pool │ ──► │  use buf │ ──► │sync.Pool│  │
│  └───────────┘       └──────────┘       └─────────┘  │
│    (reuse, no alloc each time)                        │
│                                                      │
│  Pattern 2: Pre-allocate capacity                    │
│  make([]int, 0, n) ─► append without regrowth        │
│                                                      │
│  Pattern 3: Value receiver (small struct)             │
│  func (p Point) Dist() ─► no heap alloc              │
│                                                      │
│  Pattern 4: []Item vs []*Item                        │
│  []Item  ─► ┌─┬─┬─┬─┐ one contiguous block          │
│  []*Item ─► ┌┐┌┐┌┐┌┐ N separate heap allocs        │
│              │ │ │ │                                │
│              ▼ ▼ ▼ ▼ (poor cache locality)          │
│                                                      │
│  Pattern 5: Struct of Arrays (SoA)                   │
│  X[] Y[] Z[] ─► batch-process one field at a time    │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

// Pattern 1: sync.Pool for reusing buffers
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func processWithPool(data string) string {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    buf.WriteString("processed: ")
    buf.WriteString(data)
    return buf.String()
}

// Pattern 2: Pre-allocated slices
func collectResults(n int) []int {
    results := make([]int, 0, n) // pre-allocate capacity
    for i := 0; i < n; i++ {
        results = append(results, i*i)
    }
    return results
}

// Pattern 3: Value receivers for small structs (avoid heap alloc)
type Point struct {
    X, Y float64
}

func (p Point) Distance() float64 { // value receiver — no heap alloc
    return p.X*p.X + p.Y*p.Y
}

// Pattern 4: Avoid slice-of-pointers when possible
// []*Item → each item is a separate heap allocation
// []Item  → one contiguous allocation (better cache locality, less GC work)

// Pattern 5: Struct of Arrays vs Array of Structs
type ParticlesAoS struct {
    Particles []struct{ X, Y, Z, VX, VY, VZ float64 } // AoS
}
type ParticlesSoA struct { // SoA — better for batch processing
    X, Y, Z, VX, VY, VZ []float64
}

func main() {
    fmt.Println(processWithPool("hello"))
    fmt.Println(collectResults(5))
    fmt.Println(Point{3, 4}.Distance())
}
```

---

## Monitoring GC in Production

**Tutorial: Production GC Observability**

This example demonstrates three approaches to monitor the GC in production. `runtime.ReadMemStats` provides the most detail but briefly stops the world. `debug.ReadGCStats` gives GC-specific info (pause history, last GC time) with less overhead. `expvar` exposes runtime stats over HTTP at `/debug/vars` for integration with monitoring systems like Prometheus or Grafana. Watch `GCCPUFraction` (alert if >5%) and pause times (alert if consistently >1ms).

```
┌──────────────────────────────────────────────────────┐
│  Production Monitoring Methods                        │
│                                                      │
│  Method 1: runtime.ReadMemStats (detailed, STW)      │
│  ┌─────────────────┐                                   │
│  │ HeapAlloc       │                                   │
│  │ NumGC           ├──► richest data, brief STW        │
│  │ GCCPUFraction   │                                   │
│  └─────────────────┘                                   │
│                                                      │
│  Method 2: debug.ReadGCStats (GC-focused)            │
│  ┌─────────────────┐                                   │
│  │ LastGC          │                                   │
│  │ Pause[]         ├──► lighter weight                  │
│  │ NumGC           │                                   │
│  └─────────────────┘                                   │
│                                                      │
│  Method 3: expvar (HTTP endpoint)                    │
│  GET /debug/vars ──► JSON metrics ──► Grafana        │
│                                                      │
│  Alerts:                                             │
│  • GCCPUFraction > 0.05  ──► investigate              │
│  • Pause > 1ms           ──► investigate              │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "expvar"
    "fmt"
    "net/http"
    "runtime"
    "runtime/debug"
    "time"
)

func gcMonitor() {
    // Method 1: runtime.MemStats (detailed, but causes STW to read)
    var stats runtime.MemStats
    runtime.ReadMemStats(&stats) // note: briefly stops the world
    fmt.Printf("HeapAlloc: %d MB, NumGC: %d\n",
        stats.HeapAlloc/1024/1024, stats.NumGC)

    // Method 2: debug.ReadGCStats (GC-specific stats)
    var gcStats debug.GCStats
    debug.ReadGCStats(&gcStats)
    fmt.Printf("Last GC: %v ago\n", time.Since(gcStats.LastGC))
    fmt.Printf("Num GC: %d\n", gcStats.NumGC)
    if len(gcStats.Pause) > 0 {
        fmt.Printf("Last pause: %v\n", gcStats.Pause[0])
    }

    // Method 3: expvar (expose via HTTP for monitoring)
    // Access at http://localhost:8080/debug/vars
    expvar.Publish("num_goroutines", expvar.Func(func() any {
        return runtime.NumGoroutine()
    }))
}

func main() {
    gcMonitor()

    // For production monitoring:
    // - Export runtime.MemStats to Prometheus/Grafana
    // - Watch HeapAlloc, NumGC, PauseTotalNs, GCCPUFraction
    // - Alert if GCCPUFraction > 0.05 (5% CPU on GC is high)
    // - Alert if pause times > 1ms consistently
    // - Use GODEBUG=gctrace=1 in staging for detailed analysis

    _ = http.ListenAndServe // expvar registers on default mux
}
```

---

## Interview Questions

1. **What type of garbage collector does Go use?**
   - Go uses a concurrent, tri-color mark-and-sweep GC. It is non-generational and non-compacting. It prioritizes low latency (sub-millisecond STW pauses) over maximum throughput.

2. **Explain the tri-color marking algorithm.**
   - Objects are classified as white (unvisited, potentially garbage), grey (discovered but not fully scanned), and black (fully scanned and reachable). GC starts by marking roots grey, then iteratively scans grey objects, marking their references grey and themselves black. Remaining white objects are garbage.

3. **What are the Stop-the-World (STW) phases in Go's GC?**
   - There are two brief STW pauses: (1) Mark Setup — enables write barrier, identifies roots. (2) Mark Termination — finishes marking, disables write barrier. Each is typically <100μs. Marking and sweeping run concurrently.

4. **What is a write barrier and why is it needed?**
   - A write barrier intercepts pointer writes during the concurrent marking phase. It ensures the tri-color invariant (no black-to-white pointer) isn't violated when the mutator modifies the heap while the GC is scanning. Active only during marking.

5. **What is `GOGC` and how does it affect GC behavior?**
   - `GOGC` sets the heap growth percentage that triggers GC. Default 100 means GC runs when heap doubles over live data. `GOGC=200` halves GC frequency (more memory, less CPU). `GOGC=off` disables percentage-based triggering (use with `GOMEMLIMIT`).

6. **What is `GOMEMLIMIT` and when should you use it?**
   - A soft memory limit (Go 1.19+) that makes the GC work harder to stay under the target. Set it to ~90% of container memory. Combine with `GOGC=off` for maximum throughput within a memory budget. Not a hard cap — Go can briefly exceed it.

7. **How does escape analysis relate to GC?**
   - The compiler's escape analysis determines if a variable can stay on the stack (no GC involvement) or must be heap-allocated (GC-managed). Reducing escapes reduces GC pressure. Check with `go build -gcflags="-m"`.

8. **What is mark assist?**
   - When a goroutine allocates faster than the GC can mark, the runtime forces it to assist with marking before allowing the allocation. This prevents heap growth from outpacing the GC. Visible in `go tool trace` as "GC mark assist" spans.

9. **How do you monitor GC in production?**
   - Use `GODEBUG=gctrace=1` for trace logs, `runtime.ReadMemStats()` for metrics, `debug.ReadGCStats()` for pause data, `pprof` heap profiles for allocation hotspots, and `go tool trace` for latency analysis. Export `GCCPUFraction` and pause times to monitoring dashboards.

10. **What are the best strategies to reduce GC pressure?**
    - Reduce allocations: reuse objects with `sync.Pool`, pre-allocate slices, use `strings.Builder`, avoid interface boxing in hot paths. Use value types over pointers for small structs. Use `[]T` instead of `[]*T`. Tune `GOGC`/`GOMEMLIMIT` for your workload.

11. **What is the difference between `runtime.GC()` and automatic GC?**
    - `runtime.GC()` forces an immediate GC cycle. Automatic GC is triggered by the pacer based on `GOGC` and `GOMEMLIMIT`. Forcing GC is rarely needed — appropriate only for benchmarks, tests, or after releasing large memory batches.

12. **Why doesn't Go use a generational garbage collector?**
    - Go's concurrent mark-and-sweep achieves consistently low pause times (<1ms) which suits its target workloads (servers, CLI tools). A generational GC adds complexity (write barriers for all pointer writes, promotion cost) with less benefit for Go's allocation patterns where escape analysis already keeps short-lived objects on the stack.
