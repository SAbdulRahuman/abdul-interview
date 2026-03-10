# Chapter 11 — Goroutines & Concurrency

Concurrency is Go's killer feature. Goroutines are **lightweight user-space threads** managed by the Go runtime, not the OS. They start with ~2KB of stack (which grows dynamically), compared to ~1–8MB for OS threads. You can run millions of goroutines on a single machine.

```
┌──────────────────────────────────────────────────────────┐
│          Goroutines vs OS Threads                        │
│                                                          │
│  OS Threads:                 Goroutines:                 │
│  ┌──────────┐               ┌──────────┐                │
│  │ ~1-8MB   │               │ ~2KB     │ (grows on need)│
│  │ stack    │               │ stack    │                │
│  │ OS-      │               │ Go       │                │
│  │ scheduled│               │ runtime  │                │
│  │ expensive│               │ scheduled│                │
│  │ to create│               │ very cheap│               │
│  └──────────┘               └──────────┘                │
│  ~thousands max             ~millions possible           │
│                                                          │
│  Go Scheduler (M:N model):                              │
│  ┌────────────────────────────────────────────┐          │
│  │  G (goroutines)  ──mapped to──►  M (OS threads)│     │
│  │                                            │          │
│  │  G1 ─┐                    ┌─ M1 (thread 1) │         │
│  │  G2 ─┤──► P (processor) ──┤                 │         │
│  │  G3 ─┘    (run queue)    └─ M2 (thread 2) │         │
│  │  G4 ─┐                                     │         │
│  │  G5 ─┤──► P (processor) ──── M3 (thread 3)│         │
│  │  G6 ─┘                                     │         │
│  └────────────────────────────────────────────┘          │
│  GOMAXPROCS = number of P's (default = CPU cores)       │
└──────────────────────────────────────────────────────────┘
```

## Goroutine Basics

**Tutorial: Starting Goroutines — The go Keyword**

This example shows how to launch concurrent goroutines using the `go` keyword before a function call. Each `go` call spawns a new lightweight goroutine that runs concurrently with the caller. Note that the main goroutine does not wait for spawned goroutines — if `main()` returns, all goroutines are killed immediately, so coordination (via WaitGroup or channels) is essential.

```
┌──────────────────────────────────────────────────────────┐
│          Goroutine Launching with go Keyword             │
│                                                          │
│  main()                                                  │
│    │                                                     │
│    ├── go sayHello("goroutine-1") ──► G1 runs concurrently│
│    ├── go sayHello("goroutine-2") ──► G2 runs concurrently│
│    ├── go func(){...}()           ──► G3 runs concurrently│
│    │                                                     │
│    ▼                                                     │
│  sayHello("main")  ← main goroutine continues            │
│    │                                                     │
│    ▼                                                     │
│  main() returns → ALL goroutines killed                  │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    for i := 0; i < 3; i++ {
        fmt.Printf("Hello from %s (%d)\n", name, i)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    // Start a goroutine with the 'go' keyword
    go sayHello("goroutine-1")
    go sayHello("goroutine-2")

    // Anonymous goroutine
    go func() {
        fmt.Println("Hello from anonymous goroutine!")
    }()

    // Main goroutine continues executing
    sayHello("main")

    // If main exits, ALL goroutines are killed
    // Use sync.WaitGroup or channels to coordinate
}
```

---

## Goroutines vs OS Threads

**Tutorial: Spawning Thousands of Goroutines**

This example demonstrates goroutines' lightweight nature by spawning 1000 of them in a loop. Each goroutine uses only ~2KB of stack, so this is trivial for the Go runtime. A `sync.WaitGroup` coordinates completion, and `runtime.GOMAXPROCS(0)` reveals how many OS threads were actually used — far fewer than 1000.

```
┌──────────────────────────────────────────────────────────┐
│     Spawning 1000 Goroutines on Few OS Threads           │
│                                                          │
│  for i := 0..999 → wg.Add(1) + go func(id)              │
│                                                          │
│  ┌─────┐ ┌─────┐ ┌─────┐       ┌─────┐                  │
│  │ G0  │ │ G1  │ │ G2  │ . . . │G999 │  1000 goroutines │
│  └──┬──┘ └──┬──┘ └──┬──┘       └──┬──┘                  │
│     └───────┴───────┴────...──────┘                      │
│               │  multiplexed onto                        │
│       ┌───────┼────────┐                                 │
│       ▼       ▼        ▼                                 │
│     ┌───┐  ┌───┐   ┌───┐                                │
│     │ M1│  │ M2│   │ Mn│   ~GOMAXPROCS OS threads       │
│     └───┘  └───┘   └───┘                                │
│                                                          │
│  wg.Wait() blocks until counter → 0                     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    // Goroutines are LIGHTWEIGHT — ~2KB initial stack (grows as needed)
    // OS threads are ~1-8MB stack
    // You can run millions of goroutines, but only thousands of OS threads

    var wg sync.WaitGroup

    // Spawn 1000 goroutines easily
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // Each goroutine does some work
            _ = id * id
        }(i)
    }

    wg.Wait()
    fmt.Printf("Spawned 1000 goroutines using %d OS threads\n",
        runtime.GOMAXPROCS(0))
    fmt.Printf("Active goroutines: %d\n", runtime.NumGoroutine())
}
```

---

## Go Scheduler: M:N Model

**Tutorial: Exploring the Go Runtime Scheduler**

This example inspects the Go scheduler's configuration at runtime. The M:N scheduler multiplexes many goroutines (G) onto fewer OS threads (M) via logical processors (P). `GOMAXPROCS` controls the number of P's (defaulting to CPU cores), and work-stealing balances load across processors.

```
┌──────────────────────────────────────────────────────────┐
│            M:N Scheduler Internals                       │
│                                                          │
│  G = Goroutine    P = Processor    M = OS Thread         │
│                                                          │
│  ┌──── P0 ─────┐     ┌──── P1 ─────┐                    │
│  │Local RunQueue│     │Local RunQueue│                   │
│  │ G1  G2  G3  │     │ G4  G5  G6  │                    │
│  └──────┬──────┘     └──────┬──────┘                    │
│         ▼                   ▼                            │
│       ┌───┐              ┌───┐                           │
│       │ M0│              │ M1│    OS Threads             │
│       └───┘              └───┘                           │
│                                                          │
│  GOMAXPROCS(0) → returns current P count                 │
│  GOMAXPROCS(n) → sets P count to n                       │
│  NumCPU()      → number of CPU cores                     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // Go uses an M:N scheduler:
    // M = OS threads (managed by the OS)
    // N = Goroutines (managed by Go runtime)
    // P = Processors (logical processors, run queues)

    // GOMAXPROCS controls the number of OS threads that can execute goroutines
    // Default: number of CPU cores

    fmt.Println("CPUs available:", runtime.NumCPU())
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0)) // 0 returns current value

    // Set GOMAXPROCS (usually leave at default)
    old := runtime.GOMAXPROCS(2) // use 2 OS threads
    fmt.Println("Previous GOMAXPROCS:", old)
    fmt.Println("Current GOMAXPROCS:", runtime.GOMAXPROCS(0))

    // The scheduler multiplexes N goroutines onto M OS threads
    // Work stealing: idle P steals goroutines from busy P's queue
}
```

---

## sync.WaitGroup

**Tutorial: Coordinating Goroutines with WaitGroup**

This example demonstrates the standard pattern for waiting on multiple goroutines. Call `wg.Add(1)` before each `go` statement (never inside the goroutine), use `defer wg.Done()` to decrement the counter when the goroutine finishes, and `wg.Wait()` blocks until all goroutines complete. Always pass `*sync.WaitGroup` by pointer.

```
┌──────────────────────────────────────────────────────────┐
│          sync.WaitGroup Counter Flow                     │
│                                                          │
│  wg.Add(1)  wg.Add(1)  wg.Add(1)  wg.Add(1)  wg.Add(1) │
│  counter:  1 → 2 → 3 → 4 → 5                           │
│                                                          │
│   go worker(1)  go worker(2) ... go worker(5)            │
│      │              │              │                     │
│      ▼              ▼              ▼                     │
│   wg.Done()      wg.Done()     wg.Done()                │
│  counter: 5 → 4 → 3 → 2 → 1 → 0                       │
│                                                          │
│  wg.Wait()  ◄── blocks here until counter == 0          │
│      │                                                   │
│      ▼                                                   │
│  "All workers completed!"                                │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // Decrements counter when goroutine completes

    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Duration(id) * 100 * time.Millisecond)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1) // Increment counter BEFORE starting goroutine
        go worker(i, &wg)
    }

    wg.Wait() // Blocks until counter reaches zero
    fmt.Println("All workers completed!")

    // IMPORTANT:
    // - Call Add() BEFORE go statement (not inside goroutine)
    // - Pass *sync.WaitGroup (pointer), not copy
    // - Always pair Add(1) with Done()
}
```

---

## Race Conditions

**Tutorial: Data Races and the Mutex Fix**

This example first shows the classic data race: 1000 goroutines incrementing a shared counter without synchronization, producing an unpredictable result below 1000. The fix wraps the read-modify-write operation in `mu.Lock()` / `mu.Unlock()`, ensuring only one goroutine accesses the counter at a time.

```
┌──────────────────────────────────────────────────────────┐
│         Data Race: Unsynchronized counter++              │
│                                                          │
│  G1: READ counter (0) ─┐                                │
│  G2: READ counter (0) ─┤  Both read same value!         │
│  G1: WRITE counter = 1 │                                │
│  G2: WRITE counter = 1 ◄─ Lost update!                  │
│                                                          │
│  Fix with Mutex:                                         │
│  G1: Lock() → READ (0) → WRITE (1) → Unlock()          │
│                                        G2: Lock() →     │
│                                         READ (1) →      │
│                                         WRITE (2) →     │
│                                         Unlock()        │
│  Result: counter == 1000 ✓                               │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // BAD: Race condition — multiple goroutines modify shared state
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // RACE! Read-modify-write is not atomic
        }()
    }
    wg.Wait()
    fmt.Println("Unsafe counter:", counter) // Unpredictable! Could be < 1000

    // GOOD: Fix with mutex
    counter = 0
    var mu sync.Mutex

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Println("Safe counter:", counter) // Always 1000
}
```

---

## Race Detector

**Tutorial: Using Go's Built-In Race Detector**

Go ships with a powerful runtime race detector that instruments memory accesses to find unsynchronized concurrent reads/writes. Enable it with `-race` on any Go command. It reports exact source locations of conflicting accesses. Run it in CI pipelines — it adds ~2-10x overhead, so it's for development and testing, not production.

```
┌──────────────────────────────────────────────────────────┐
│          Go Race Detector Usage                          │
│                                                          │
│  Build/Run                  Test                         │
│  ┌──────────────────┐       ┌───────────────────┐        │
│  │ go run  -race .  │       │ go test -race ./..│        │
│  │ go build -race . │       └───────────────────┘        │
│  └──────────────────┘                                    │
│            │                                             │
│            ▼                                             │
│  ┌──────────────────────────────────┐                    │
│  │  WARNING: DATA RACE              │                    │
│  │  Read at 0x... by goroutine 7    │                    │
│  │  Previous write at 0x... by G8   │                    │
│  └──────────────────────────────────┘                    │
│  Instruments memory accesses at runtime                  │
│  ~2-10x slower — use in dev/CI, not production           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

// Detect race conditions at runtime:
//   go run -race main.go
//   go test -race ./...
//   go build -race -o app .

// The race detector instruments memory accesses and reports
// any unsynchronized concurrent access to shared variables.

// Example output when race detected:
// ==================
// WARNING: DATA RACE
// Read at 0x00c0000b4008 by goroutine 7:
//   main.main.func1()
//       /path/main.go:15 +0x3e
// Previous write at 0x00c0000b4008 by goroutine 8:
//   main.main.func1()
//       /path/main.go:15 +0x56
// ==================

// Always run tests with -race in CI/CD pipelines!
```

---

## Goroutine Leaks

**Tutorial: Preventing Goroutine Leaks with Context**

A goroutine that blocks forever and can never exit is a "leak" — it consumes memory indefinitely. This example contrasts a leaky goroutine (blocked on a channel nobody writes to) with the proper fix using `context.Context` for cancellation. Always ensure every goroutine has an exit path via context, done channels, or timeouts.

```
┌──────────────────────────────────────────────────────────┐
│         Goroutine Leak vs Proper Cancellation            │
│                                                          │
│  LEAK:                         FIX:                      │
│  ┌────────────┐                ┌────────────┐            │
│  │ goroutine  │                │ goroutine  │            │
│  │  val := <-ch│               │  select {  │            │
│  │  (blocks   │                │  case <-ch │            │
│  │   forever!)│                │  case <-ctx│            │
│  └────────────┘                │  }         │            │
│  ch never receives             └─────┬──────┘            │
│  → goroutine LEAKS                   │                   │
│                                ctx.Done() ──► goroutine  │
│                                  fires        exits ✓    │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
    "runtime"
    "time"
)

// BAD: Goroutine leak — goroutine blocks forever
func leakyFunction() {
    ch := make(chan int) // unbuffered channel
    go func() {
        val := <-ch // blocks forever — nobody sends to ch
        fmt.Println(val)
    }()
    // ch is never sent to, goroutine leaks!
}

// GOOD: Use context for cancellation
func properFunction(ctx context.Context) {
    ch := make(chan int)

    go func() {
        select {
        case val := <-ch:
            fmt.Println("Received:", val)
        case <-ctx.Done():
            fmt.Println("Goroutine cancelled, cleaning up")
            return // goroutine exits cleanly
        }
    }()
}

func main() {
    fmt.Println("Goroutines before leak:", runtime.NumGoroutine())

    leakyFunction()
    time.Sleep(10 * time.Millisecond)
    fmt.Println("Goroutines after leak:", runtime.NumGoroutine())
    // Count increased — goroutine is stuck!

    // With context — goroutine can be cancelled
    ctx, cancel := context.WithCancel(context.Background())
    properFunction(ctx)
    cancel() // signals goroutine to exit
    time.Sleep(10 * time.Millisecond)
    fmt.Println("Goroutines after cancel:", runtime.NumGoroutine())

    // Prevention tips:
    // 1. Always ensure goroutines can exit (use context, done channels)
    // 2. Close channels when done sending
    // 3. Use timeouts on blocking operations
    // 4. Monitor runtime.NumGoroutine() in production
}
```

---

## runtime Utilities

**Tutorial: Key runtime Package Functions**

This example showcases essential functions from the `runtime` package. `Gosched()` yields the processor to let other goroutines run, `NumGoroutine()` reports the count of active goroutines (useful for leak detection), and `LockOSThread()` pins a goroutine to a specific OS thread — required for some C libraries and GUI frameworks.

```
┌──────────────────────────────────────────────────────────┐
│          runtime Package Utilities                       │
│                                                          │
│  Gosched()         Yield to let others run               │
│  ┌──────┐ yield ┌──────┐ yield ┌──────┐                  │
│  │ main │──────►│ G1   │──────►│ main │ ...              │
│  └──────┘       └──────┘       └──────┘                  │
│                                                          │
│  NumGoroutine()  → count of active goroutines            │
│  GOMAXPROCS(0)   → get current parallelism               │
│  NumCPU()        → number of CPU cores                   │
│                                                          │
│  LockOSThread()  → pin goroutine to OS thread            │
│  ┌──────────┐ pinned ┌──────────┐                        │
│  │goroutine │═══════►│ OS Thread│  (for CGo, GUI, TLS)  │
│  └──────────┘        └──────────┘                        │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // runtime.Gosched() — yield processor, let other goroutines run
    go func() {
        for i := 0; i < 3; i++ {
            fmt.Println("goroutine:", i)
            runtime.Gosched() // give other goroutines a chance
        }
    }()

    for i := 0; i < 3; i++ {
        fmt.Println("main:", i)
        runtime.Gosched()
    }

    // runtime.NumGoroutine() — count active goroutines
    fmt.Println("\nActive goroutines:", runtime.NumGoroutine())

    // runtime.GOMAXPROCS
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
    fmt.Println("NumCPU:", runtime.NumCPU())

    // runtime.LockOSThread() — pin goroutine to current OS thread
    // Required for some C libraries, GUI frameworks, and thread-local storage
    runtime.LockOSThread()
    defer runtime.UnlockOSThread()
    // Now this goroutine will always execute on the same OS thread
}
```

---

## Data Race vs Race Condition

**Tutorial: Understanding Data Race vs Race Condition**

This example clarifies the critical distinction: a **data race** is unsynchronized concurrent memory access (undefined behavior, caught by `-race`), while a **race condition** is a logic bug where correctness depends on timing. Fixing a data race with atomics removes undefined behavior, but the program may still have race conditions if the logic is wrong.

```
┌──────────────────────────────────────────────────────────┐
│       Data Race vs Race Condition                        │
│                                                          │
│  DATA RACE (UB, detectable):                             │
│  G1: counter++  ─┐  unsynchronized writes to             │
│  G2: counter++  ─┘  same memory → undefined behavior    │
│  Fix: atomic.AddInt64(&counter, 1)                       │
│                                                          │
│  RACE CONDITION (logic bug, NOT detectable):             │
│  G1: if balance≥100 → withdraw(100) ─┐ both see         │
│  G2: if balance≥100 → withdraw(100) ─┘ balance=150      │
│  Result: balance = -50 (overdraft!)                      │
│  Even with locks around each check+act, interleaving     │
│  between check and act causes incorrect logic.           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    // DATA RACE: concurrent unsynchronized access to the same memory
    // where at least one access is a write.
    // Detected by Go's race detector (go run -race).
    // This is UNDEFINED BEHAVIOR in Go.

    // RACE CONDITION: a logic bug where program behavior depends
    // on the ordering/timing of events.
    // NOT detected by the race detector — it's a correctness issue.

    // Example: Data race (detectable)
    var counter int
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DATA RACE: unsynchronized write
        }()
    }
    wg.Wait()
    fmt.Println("Data race counter:", counter) // unpredictable

    // Fix the data race — but race CONDITION may still exist
    var atomicCounter int64
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&atomicCounter, 1) // no data race
        }()
    }
    wg.Wait()
    fmt.Println("Atomic counter:", atomicCounter) // always 100

    // Example: Race condition WITHOUT data race
    // Two goroutines check-then-act with proper locking, but the
    // interleaving of check + act produces wrong results.
    // E.g., two goroutines both check "balance >= 100" then withdraw,
    // even though total balance only supports one withdrawal.
}
```

---

## Runtime Deadlock Detection

**Tutorial: How Go Detects Deadlocks at Runtime**

Go's runtime automatically detects total deadlocks — when ALL goroutines are blocked with no way to proceed. It reports "fatal error: all goroutines are asleep - deadlock!" However, partial deadlocks (where only some goroutines are stuck but at least one is still running) are NOT detected. Use `pprof` goroutine profiles to diagnose those.

```
┌──────────────────────────────────────────────────────────┐
│       Go Runtime Deadlock Detection                      │
│                                                          │
│  Total Deadlock (detected ✓):                            │
│  ┌──────┐ send ┌────────┐                                │
│  │ main │─────►│ ch (0) │  no receiver → main blocks    │
│  └──────┘      └────────┘  ALL goroutines asleep         │
│  → "fatal error: all goroutines are asleep - deadlock!" │
│                                                          │
│  Partial Deadlock (NOT detected ✗):                      │
│  ┌──────┐ running    ┌──────┐ blocked                    │
│  │ HTTP │            │  G1  │◄──┐                        │
│  │server│            └──────┘   │ waiting on each other  │
│  └──────┘            ┌──────┐   │                        │
│  (still alive)       │  G2  │───┘                        │
│                      └──────┘                            │
│  Use pprof to detect partial deadlocks                   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

// Go runtime detects when ALL goroutines are blocked — a total deadlock.
// It reports: "fatal error: all goroutines are asleep - deadlock!"

// Example 1: Send on unbuffered channel with no receiver
// func main() {
//     ch := make(chan int)
//     ch <- 1  // deadlock! main goroutine blocks, no other goroutines
// }

// Example 2: Receive with no sender
// func main() {
//     ch := make(chan int)
//     <-ch  // deadlock!
// }

// Example 3: Two goroutines waiting on each other
// func main() {
//     ch1 := make(chan int)
//     ch2 := make(chan int)
//     go func() { <-ch1; ch2 <- 1 }()
//     <-ch2  // deadlock: both goroutines waiting
//     ch1 <- 1
// }

// IMPORTANT:
// - Runtime ONLY detects when ALL goroutines are blocked
// - If even one goroutine is running (e.g., HTTP server), no detection
// - Partial deadlocks (subset of goroutines) are NOT detected
// - Use pprof goroutine profile to find partial deadlocks

func main() {
    // This would deadlock:
    // ch := make(chan int)
    // ch <- 1 // fatal error: all goroutines are asleep - deadlock!

    // Fix: use goroutines properly
    ch := make(chan int)
    go func() { ch <- 42 }()
    _ = <-ch
}
```

---

## Interview Questions

1. **What is a goroutine and how is it different from an OS thread?**
   - A goroutine is a lightweight concurrent function execution managed by the Go runtime. It starts with ~2-8 KB stack (vs ~1-8 MB for threads), is scheduled by Go's runtime (not the OS), and thousands can run concurrently.

2. **Explain Go's M:N scheduling model.**
   - M goroutines are multiplexed onto N OS threads via P processors. M = machine (OS thread), G = goroutine, P = processor (logical CPU context). `GOMAXPROCS` controls the number of P's.

3. **What is `GOMAXPROCS`?**
   - It sets the maximum number of OS threads that can execute Go code simultaneously. Default is the number of CPU cores. It does NOT limit the number of goroutines.

4. **How do you wait for goroutines to finish?**
   - Use `sync.WaitGroup`: `wg.Add(n)` before launching, `wg.Done()` inside each goroutine, and `wg.Wait()` to block until all complete. Alternatively, use channels or `errgroup`.

5. **What is a race condition and how do you detect it?**
   - A race condition occurs when multiple goroutines access shared data concurrently and at least one writes. Use `go run -race .` or `go test -race` to detect races — the race detector reports where concurrent accesses happen.

6. **What causes goroutine leaks?**
   - Goroutines that never exit: blocked on a channel with no sender/receiver, waiting for a lock never released, or infinite loops without exit conditions. Prevention: use context cancellation, close channels, set timeouts.

7. **What does `runtime.Gosched()` do?**
   - It yields the processor, allowing other goroutines to run. The current goroutine is not suspended permanently — it will be rescheduled.

8. **Is it safe to share variables between goroutines?**
   - Only with synchronization (mutexes, channels, atomic operations). Without synchronization, concurrent access to shared variables is a data race — undefined behavior.

9. **How many goroutines can you create?**
   - Practically millions (each uses ~2-8 KB of stack). The limit is memory, not a fixed cap. However, having too many can cause scheduling overhead.

10. **What is `runtime.NumGoroutine()` used for?**
    - It returns the number of currently active goroutines. Useful for detecting goroutine leaks in tests — check before and after to ensure no goroutines were leaked.

11. **What is the difference between a data race and a race condition?**
    - A data race is concurrent unsynchronized memory access (at least one write) — it's undefined behavior, detected by `go run -race`. A race condition is a logic bug where correctness depends on timing — it can exist even with proper synchronization.

12. **How does Go detect deadlocks at runtime?**
    - The runtime detects when ALL goroutines are blocked ("fatal error: all goroutines are asleep - deadlock!"). It only catches total deadlocks — partial deadlocks (subset of goroutines stuck) are not detected.

13. **What is `runtime.LockOSThread()` used for?**
    - Pins the current goroutine to its OS thread. Required for some C libraries, GUI frameworks (OpenGL, Cocoa), and code that uses thread-local storage. Paired with `runtime.UnlockOSThread()`.
