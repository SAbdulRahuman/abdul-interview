# Chapter 31 — Go Scheduler Deep Dive (GMP Model)

Go's runtime scheduler is what makes goroutines so efficient. It implements an **M:N scheduling model** — multiplexing N goroutines onto M OS threads. Understanding the GMP model is essential for senior-level Go interviews.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Go Scheduler — GMP Model                      │
│                                                                  │
│  G = Goroutine      M = Machine (OS Thread)     P = Processor   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Global Run Queue                         │ │
│  │  [G5] [G6] [G7] [G8] ...                                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          │                                       │
│                     (steal / fill)                                │
│                          │                                       │
│  ┌──────────────────┐   ┌──────────────────┐                    │
│  │    P0             │   │    P1             │                    │
│  │  Local Queue:     │   │  Local Queue:     │     GOMAXPROCS=2  │
│  │  [G1] [G2]        │   │  [G3] [G4]        │                    │
│  │       │           │   │       │           │                    │
│  │       ▼           │   │       ▼           │                    │
│  │  ┌─────────┐     │   │  ┌─────────┐     │                    │
│  │  │  M0     │     │   │  │  M1     │     │                    │
│  │  │(thread) │     │   │  │(thread) │     │                    │
│  │  │ runs G1 │     │   │  │ runs G3 │     │                    │
│  │  └─────────┘     │   │  └─────────┘     │                    │
│  └──────────────────┘   └──────────────────┘                    │
│                                                                  │
│  M2 (idle, parked — no P attached)                               │
│  M3 (blocked in syscall — P detached, handed to M4)              │
└──────────────────────────────────────────────────────────────────┘
```

## The Three Entities: G, M, P

### G — Goroutine

**Tutorial: Understanding Goroutine Internals**

A goroutine (G) is Go's unit of concurrent execution. Each G has its own stack (starts at ~2-8 KB, grows dynamically up to 1 GB default), a program counter, and scheduling state. Goroutines are extremely cheap compared to OS threads — you can run millions of them.

```
┌────────────────────────────────┐
│  Goroutine (G)                 │
│  ┌──────────────────────────┐  │
│  │ Stack      ~2-8 KB init  │  │
│  │            grows as needed│  │
│  │ PC         program counter│  │
│  │ Status     runnable/      │  │
│  │            running/       │  │
│  │            waiting/       │  │
│  │            dead           │  │
│  │ Fn         function to run│  │
│  │ Context    local storage  │  │
│  └──────────────────────────┘  │
│                                │
│  States:                       │
│  Grunnable → Grunning → Gwaiting │
│       ↑                   │    │
│       └───────────────────┘    │
│                                │
│  OS Thread: ~1-8 MB stack       │
│  Goroutine: ~2-8 KB stack       │
│  Ratio: 100-1000x cheaper       │
└────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	// Each go statement creates a new G
	for i := 0; i < 10; i++ {
		go func(id int) {
			fmt.Printf("G%d running on thread\n", id)
		}(i)
	}

	// Check how many goroutines exist
	fmt.Println("Active goroutines:", runtime.NumGoroutine())

	// Check goroutine stack size
	var buf [64]byte
	n := runtime.Stack(buf[:], false)
	fmt.Printf("Stack trace:\n%s\n", buf[:n])
}
```

### M — Machine (OS Thread)

**Tutorial: OS Thread Management in the Runtime**

An M represents an OS thread managed by the Go runtime. The runtime creates Ms as needed to run Gs. An M must be attached to a P to execute goroutines. When an M blocks on a system call, its P is detached and given to another M so other goroutines can continue executing. Idle Ms are parked (sleeping) until needed.

```
┌────────────────────────────────────────────┐
│  M (Machine / OS Thread)                   │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  curg    → currently running G       │  │
│  │  p       → attached P (or nil)       │  │
│  │  g0      → scheduler goroutine       │  │
│  │  spinning → looking for work?         │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  M lifecycle:                              │
│  Created → Attached to P → Running G       │
│    │                          │             │
│    │                     Syscall blocks     │
│    │                          │             │
│    │                     P detached         │
│    │                     P given to new M   │
│    │                          │             │
│    └── Parked (idle) ◄───── Syscall returns │
│                              no P available │
│                                            │
│  Max M count: 10,000 (default)             │
│  runtime/debug.SetMaxThreads(n)            │
└────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"sync"
)

func main() {
	// GOMAXPROCS controls how many Ps (and thus max concurrent Ms)
	fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0)) // 0 = query, don't change

	// When a goroutine makes a blocking syscall, the M blocks
	// but the P is handed off so other goroutines aren't stalled
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		// This blocks the M in a syscall
		// The runtime hands P to another M
		f, _ := os.CreateTemp("", "test")
		f.Write([]byte("blocking syscall"))
		f.Close()
		os.Remove(f.Name())
		fmt.Println("Syscall goroutine done")
	}()

	go func() {
		defer wg.Done()
		// This goroutine runs on a different M (or same M after handoff)
		total := 0
		for i := 0; i < 1000; i++ {
			total += i
		}
		fmt.Println("Compute goroutine done, total:", total)
	}()

	wg.Wait()
}
```

### P — Processor (Logical Processor)

**Tutorial: The P as a Scheduling Context**

A P (Processor) is a logical scheduling context. The number of Ps equals `GOMAXPROCS` (default = number of CPU cores). Each P has a **local run queue** (LRQ) that holds runnable goroutines. A P must be attached to an M to execute goroutines. The P stores per-P resources like mcache (memory allocation cache), making allocation fast and lock-free.

```
┌────────────────────────────────────────────┐
│  P (Processor / Logical CPU)               │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  Local Run Queue (LRQ)               │  │
│  │  capacity: 256 Gs                    │  │
│  │  [G1] → [G2] → [G3] → ...           │  │
│  │                                      │  │
│  │  mcache    → per-P memory cache      │  │
│  │  runnext   → next G to run (fast)    │  │
│  │  status    → idle / running / syscall │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  Total Ps = GOMAXPROCS (default: NumCPU)   │
│  Max LRQ size = 256 goroutines             │
│  Overflow → global run queue               │
└────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	// Set number of Ps (logical processors)
	prev := runtime.GOMAXPROCS(2) // set to 2
	fmt.Printf("Previous GOMAXPROCS: %d, now: %d\n", prev, runtime.GOMAXPROCS(0))

	// With GOMAXPROCS=2, at most 2 goroutines execute truly in parallel
	// Other goroutines wait in local run queues or global run queue

	// Reset to use all CPUs
	runtime.GOMAXPROCS(runtime.NumCPU())
	fmt.Printf("CPUs: %d, GOMAXPROCS: %d\n", runtime.NumCPU(), runtime.GOMAXPROCS(0))
}
```

---

## Scheduling Cycle

**Tutorial: How A Goroutine Gets Scheduled**

When a goroutine becomes runnable, the scheduler places it in a P's local run queue. The M attached to that P picks up the next G and executes it. The scheduling cycle is: find a runnable G → execute it → when it yields/blocks → find the next G.

```
┌──────────────────────────────────────────────────────────────┐
│  Scheduling Cycle (what M does in a loop)                    │
│                                                              │
│  ┌─────────┐                                                 │
│  │ findRunnable() ◄──────────────────────────┐               │
│  └────┬────┘                                 │               │
│       │ found G                              │               │
│       ▼                                      │               │
│  ┌─────────┐                                 │               │
│  │ execute(G) │  ← set G status = running    │               │
│  └────┬────┘                                 │               │
│       │ G yields / blocks / finishes         │               │
│       ▼                                      │               │
│  ┌─────────┐                                 │               │
│  │ schedule() │ ← put G back / discard       │               │
│  └────┬────┘                                 │               │
│       │                                      │               │
│       └──────────────────────────────────────┘               │
│                                                              │
│  findRunnable() priority:                                    │
│  1. P.runnext (1 slot, highest priority)                     │
│  2. P's local run queue                                      │
│  3. Global run queue (checked every 61 schedules)            │
│  4. Network poller (poll for I/O completion)                 │
│  5. Work stealing from other P's local queue (steal half)    │
└──────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	// Demonstrate scheduling with GOMAXPROCS=1
	// With 1 P, goroutines run cooperatively (no true parallelism)
	runtime.GOMAXPROCS(1)

	var wg sync.WaitGroup
	wg.Add(3)

	for i := 0; i < 3; i++ {
		go func(id int) {
			defer wg.Done()
			for j := 0; j < 3; j++ {
				fmt.Printf("G%d: iteration %d\n", id, j)
				// runtime.Gosched() voluntarily yields the processor
				// Without this, one goroutine might run to completion
				// before others get a chance (pre-Go 1.14)
				runtime.Gosched()
			}
		}(i)
	}

	wg.Wait()
}
```

---

## Work Stealing

**Tutorial: How Idle Processors Find Work**

When a P's local run queue is empty, the M doesn't just sit idle — it **steals work** from other Ps. The scheduler randomly picks another P and steals half of its local run queue. This keeps all processors busy and distributes work evenly without a central lock on the global queue.

```
┌────────────────────────────────────────────────────────┐
│  Work Stealing Algorithm                               │
│                                                        │
│  P0 (empty LRQ)         P1 (has [G5][G6][G7][G8])     │
│  ┌──────────────┐       ┌──────────────────────────┐   │
│  │  LRQ: empty  │       │  LRQ: [G5][G6][G7][G8]  │   │
│  │              │       │                          │   │
│  │  M0: idle    │       │  M1: running G4          │   │
│  └──────┬───────┘       └──────────────────────────┘   │
│         │                                              │
│         │  1. Check own LRQ → empty                    │
│         │  2. Check global queue → empty               │
│         │  3. Check netpoll → nothing                  │
│         │  4. Pick random victim P → P1                │
│         │  5. Steal half of P1's LRQ                   │
│         │                                              │
│         ▼                                              │
│  P0 (after steal)        P1 (after steal)              │
│  ┌──────────────┐       ┌──────────────────────────┐   │
│  │  LRQ: [G7][G8]│       │  LRQ: [G5][G6]          │   │
│  │  M0: runs G7 │       │  M1: running G4          │   │
│  └──────────────┘       └──────────────────────────┘   │
│                                                        │
│  Fairness: global queue checked every 61 schedules     │
│  to prevent starvation of globally queued goroutines   │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
)

func main() {
	// With GOMAXPROCS=4, we have 4 Ps
	// Work stealing ensures balanced distribution
	runtime.GOMAXPROCS(4)

	var (
		wg      sync.WaitGroup
		counter int64
	)

	// Create 1000 goroutines — far more than 4 Ps
	// The scheduler distributes them across P local queues
	// and uses work stealing to keep all Ps busy
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			// Simulate work
			for j := 0; j < 1000; j++ {
				atomic.AddInt64(&counter, 1)
			}
		}()
	}

	wg.Wait()
	fmt.Println("Total operations:", counter)
	fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
}
```

---

## Preemption

**Tutorial: Cooperative vs Asynchronous Preemption**

Before Go 1.14, goroutines were only preempted at **safe points** — function calls, channel ops, etc. A tight loop with no function calls could hog a P indefinitely. Go 1.14 introduced **asynchronous preemption**: the runtime sends a signal (SIGURG on Unix) to the thread running a long goroutine, forcing it to yield. This ensures no goroutine can monopolize a processor.

```
┌────────────────────────────────────────────────────────────┐
│  Preemption Evolution                                      │
│                                                            │
│  Pre Go 1.14: Cooperative Preemption                       │
│  ─────────────────────────────────────                     │
│  Goroutines preempted ONLY at safe points:                 │
│  • Function calls (stack check → might yield)              │
│  • Channel operations                                      │
│  • System calls                                            │
│  • runtime.Gosched()                                       │
│                                                            │
│  Problem: for { x++ }  ← never yields, starves others!    │
│                                                            │
│  Go 1.14+: Asynchronous Preemption                         │
│  ─────────────────────────────────                         │
│  Runtime sends SIGURG signal to OS thread                  │
│  Thread handles signal → saves goroutine state → yields    │
│                                                            │
│  for { x++ }  ← now gets preempted after ~10ms            │
│                                                            │
│  Mechanism:                                                │
│  sysmon goroutine ──► detects G running > 10ms             │
│       │                                                    │
│       ▼                                                    │
│  send SIGURG to M ──► signal handler saves G state         │
│       │                                                    │
│       ▼                                                    │
│  G moved to run queue ──► M picks next G                   │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	runtime.GOMAXPROCS(1) // single P — highlights preemption

	// In Go 1.14+, even tight loops get preempted
	go func() {
		// Tight compute loop — no function calls
		// Pre-1.14: this would starve the main goroutine
		// Post-1.14: async preemption kicks in after ~10ms
		x := 0
		for i := 0; i < 1e9; i++ {
			x += i
		}
		fmt.Println("Loop done, x =", x)
	}()

	// This goroutine gets a chance to run thanks to async preemption
	go func() {
		time.Sleep(50 * time.Millisecond)
		fmt.Println("Second goroutine ran!")
	}()

	time.Sleep(2 * time.Second)
}
```

---

## Sysmon — The System Monitor

**Tutorial: Background Monitoring Thread**

The sysmon thread is a special M that runs WITHOUT a P. It wakes up periodically to perform background tasks: retaking Ps from goroutines stuck in syscalls, triggering GC, preempting long-running goroutines, and flushing network polls. It starts checking every 20μs and backs off to 10ms when idle.

```
┌────────────────────────────────────────────────────────┐
│  sysmon — System Monitor Thread                        │
│  (runs without a P, periodic background tasks)         │
│                                                        │
│  Every ~10ms (adaptive interval):                      │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 1. Retake Ps from syscall-blocked Ms             │  │
│  │    - M blocked in syscall > 10ms → detach P      │  │
│  │    - P given to idle M or new M                  │  │
│  │                                                  │  │
│  │ 2. Preempt long-running goroutines               │  │
│  │    - G running > 10ms → set preempt flag         │  │
│  │    - (Go 1.14+: send SIGURG for async preempt)   │  │
│  │                                                  │  │
│  │ 3. Network polling                               │  │
│  │    - Poll for completed network I/O              │  │
│  │    - Wake goroutines waiting on net connections   │  │
│  │                                                  │  │
│  │ 4. GC triggers                                   │  │
│  │    - Force GC if none for 2 minutes              │  │
│  │    - (forcegchelper)                              │  │
│  │                                                  │  │
│  │ 5. Timer management                              │  │
│  │    - Check for expired timers                    │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  Interval: starts at 20μs, backs off to 10ms          │
│  Thread: runs as a regular M, no P needed              │
└────────────────────────────────────────────────────────┘
```

---

## Syscall Handling & Handoff

**Tutorial: How The Scheduler Handles Blocking System Calls**

When a goroutine makes a blocking system call (file I/O, network on some platforms, CGo), the OS thread (M) blocks. The runtime detects this and **hands off** the P to another M so the remaining goroutines in that P's queue can continue executing. When the syscall returns, the M tries to reacquire a P; if none is available, the G goes to the global queue and the M parks.

```
┌────────────────────────────────────────────────────────────┐
│  Syscall Handoff                                           │
│                                                            │
│  BEFORE syscall:                                           │
│  ┌──────┐    ┌──────┐                                     │
│  │  P0  │────│  M0  │─── running G1                       │
│  │[G2,G3]│    └──────┘                                     │
│  └──────┘                                                  │
│                                                            │
│  G1 enters blocking syscall (e.g., file read):             │
│                                                            │
│  DURING syscall:                                           │
│  ┌──────┐    ┌──────┐    ┌──────┐                         │
│  │  P0  │────│  M1  │    │  M0  │─── G1 (blocked in       │
│  │[G2,G3]│    │runs G2│    │      │    syscall, no P)       │
│  └──────┘    └──────┘    └──────┘                         │
│  P0 detached from M0, given to M1 (or new M created)      │
│                                                            │
│  AFTER syscall returns:                                    │
│  M0 wakes up with G1, tries to acquire a P:               │
│  • If idle P available → attach P, run G1                  │
│  • If no idle P → put G1 in global queue, M0 parks         │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(1) // 1 P — makes handoff visible

	var wg sync.WaitGroup
	wg.Add(2)

	// Goroutine 1: blocking syscall
	go func() {
		defer wg.Done()
		// os.ReadFile triggers a blocking syscall
		// The runtime hands off P0 to another M
		// so goroutine 2 can run during this syscall
		data, err := os.ReadFile("/etc/hostname")
		if err != nil {
			fmt.Println("Read error:", err)
			return
		}
		fmt.Printf("Read %d bytes from file\n", len(data))
	}()

	// Goroutine 2: pure computation
	go func() {
		defer wg.Done()
		// This runs on a different M using the handed-off P
		sum := 0
		for i := 0; i < 1000; i++ {
			sum += i
		}
		fmt.Println("Computed sum:", sum)
	}()

	wg.Wait()
}
```

---

## Network Poller Integration

**Tutorial: Non-Blocking I/O with the Netpoller**

Network I/O in Go doesn't block OS threads. Instead, the runtime uses the OS's asynchronous I/O mechanism (epoll on Linux, kqueue on macOS) via the **netpoller**. When a goroutine does network I/O, it's parked (not consuming an M) until the I/O completes. The netpoller wakes goroutines when their file descriptors are ready.

```
┌────────────────────────────────────────────────────────────┐
│  Network Poller (epoll/kqueue integration)                 │
│                                                            │
│  Goroutine does net.Read():                                │
│  1. Attempt read → would block (EAGAIN)                    │
│  2. G is parked (status = waiting)                         │
│  3. fd registered with epoll/kqueue                        │
│  4. M is free to run other goroutines                      │
│  5. When data arrives → epoll notifies runtime             │
│  6. G made runnable again → put back on run queue          │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  G1: net.Read() → park G1 → register fd with epoll  │  │
│  │       │                                              │  │
│  │  M0 runs G2, G3 ... (no thread blocked!)             │  │
│  │       │                                              │  │
│  │  epoll: fd ready! → wake G1 → put in run queue       │  │
│  │       │                                              │  │
│  │  M0 (or M1) picks up G1, completes Read()           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Result: 10,000 concurrent connections with ~10 threads    │
│  (NOT 10,000 threads like thread-per-connection models)    │
└────────────────────────────────────────────────────────────┘
```

---

## LockOSThread

**Tutorial: Pinning a Goroutine to an OS Thread**

`runtime.LockOSThread()` pins the current goroutine to its current OS thread. No other goroutine can run on that thread until `UnlockOSThread()` is called. This is needed when interacting with thread-local state (e.g., OpenGL, certain C libraries, or OS APIs that require thread affinity).

```
┌────────────────────────────────────────────────────────┐
│  runtime.LockOSThread()                                │
│                                                        │
│  Normal:  G can run on any M                           │
│  Locked:  G is pinned to specific M                    │
│           No other G runs on this M                    │
│                                                        │
│  Use cases:                                            │
│  • GUI libraries (OpenGL, Cocoa) — need main thread    │
│  • C libraries with thread-local storage                │
│  • OS-level thread affinity (setns, unshare on Linux)  │
│  • init() in main package locks main goroutine to M0   │
│                                                        │
│  ⚠️  The locked M cannot be reused by other Gs         │
│  ⚠️  Goroutine MUST call UnlockOSThread or exit        │
│  ⚠️  If locked G exits without unlock, M is destroyed  │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	done := make(chan bool)

	go func() {
		// Pin this goroutine to its OS thread
		runtime.LockOSThread()
		defer runtime.UnlockOSThread()

		// All code here runs on the SAME OS thread
		// Useful for C libraries with thread-local state
		fmt.Println("Locked goroutine running")

		// Any goroutine created here runs on OTHER threads
		// This thread is exclusively for this goroutine

		done <- true
	}()

	<-done
}
```

---

## GODEBUG Scheduler Tracing

**Tutorial: Observing the Scheduler in Action**

Set `GODEBUG=schedtrace=1000` to print scheduler state every 1000ms. Add `scheddetail=1` for per-P and per-M details. This is invaluable for debugging scheduling issues in production.

```
┌────────────────────────────────────────────────────────────┐
│  GODEBUG=schedtrace=1000 ./myapp                           │
│                                                            │
│  Output format:                                            │
│  SCHED 1004ms: gomaxprocs=4 idleprocs=2 threads=6          │
│    spinningthreads=1 idlethreads=2                         │
│    runqueue=0 [5 0 2 0]                                    │
│                                                            │
│  Breakdown:                                                │
│  ┌──────────────────┬──────────────────────────────────┐   │
│  │ gomaxprocs=4     │ Number of Ps                     │   │
│  │ idleprocs=2      │ Ps with no work                  │   │
│  │ threads=6        │ Total OS threads (Ms)            │   │
│  │ spinningthreads=1│ Ms looking for work              │   │
│  │ idlethreads=2    │ Parked Ms                        │   │
│  │ runqueue=0       │ Global run queue length           │   │
│  │ [5 0 2 0]        │ Local run queue per P             │   │
│  └──────────────────┴──────────────────────────────────┘   │
│                                                            │
│  GODEBUG=schedtrace=1000,scheddetail=1 → per-G details    │
└────────────────────────────────────────────────────────────┘
```

```go
// Run with: GODEBUG=schedtrace=1000 go run main.go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(4)

	var wg sync.WaitGroup
	// Create enough goroutines to see scheduling in action
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			sum := 0
			for j := 0; j < 1_000_000; j++ {
				sum += j
			}
			_ = sum
		}(i)
	}

	wg.Wait()
	fmt.Println("All goroutines completed")
}
```

---

## Interview Questions

1. **What are G, M, and P in Go's scheduler?**
   - G = goroutine (lightweight execution unit, ~2-8KB stack). M = machine/OS thread. P = processor/scheduling context (number = GOMAXPROCS). M must acquire a P to run a G. The GMP model enables M:N scheduling — N goroutines multiplexed onto M threads via P processors.

2. **What happens when a goroutine makes a blocking syscall?**
   - The M blocks in the kernel. The runtime detaches the P from the blocked M and assigns it to an idle M (or creates a new one). When the syscall returns, the M tries to reacquire a P. If none is available, the G is put in the global run queue and M parks.

3. **What is work stealing?**
   - When a P's local run queue is empty, its M steals half the goroutines from another random P's local queue. This keeps all Ps busy without a central lock. The global queue is also checked every 61 scheduling cycles to prevent starvation.

4. **How does asynchronous preemption work (Go 1.14+)?**
   - The sysmon thread detects goroutines running > 10ms and sends a SIGURG signal to the thread. The signal handler saves the goroutine state and returns it to the run queue. This prevents tight loops from starving other goroutines.

5. **What is GOMAXPROCS and how does it affect scheduling?**
   - GOMAXPROCS sets the number of Ps (default = NumCPU). It controls maximum parallelism — how many goroutines execute simultaneously. It doesn't limit total goroutines or threads. More Ms can exist (for blocked syscalls), but only GOMAXPROCS Gs run at once.

6. **What is the sysmon thread?**
   - A background M that runs without a P. Periodically checks for: goroutines stuck in syscalls (retakes their P), long-running goroutines (triggers preemption), GC deadlines, network I/O readiness, and expired timers. Interval adapts from 20μs to 10ms.

7. **When would you use `runtime.LockOSThread()`?**
   - When calling C libraries with thread-local storage, GPU/OpenGL APIs requiring main thread, or Linux namespacing (setns/unshare). Pins the goroutine to its M — no other G runs on that M. Must call `UnlockOSThread()` or the M is destroyed when G exits.

8. **Why doesn't network I/O block OS threads in Go?**
   - The runtime uses the netpoller (epoll/kqueue), which does non-blocking I/O. When a goroutine reads from a socket and it would block, the goroutine is parked (doesn't consume an M), and the fd is registered with epoll. When data arrives, the goroutine is woken up. This allows thousands of concurrent connections on few threads.

---

## Interview Problems & Solutions

### Problem 1 — Demonstrate Scheduler Behavior with GOMAXPROCS

**Problem:** Write a program that shows different behavior when `GOMAXPROCS` is 1 vs the number of CPUs. Create goroutines that increment a shared counter a million times each. Show how parallelism affects execution and data races.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	for _, procs := range []int{1, runtime.NumCPU()} {
		runtime.GOMAXPROCS(procs)

		var counter int64
		var wg sync.WaitGroup
		numGoroutines := 10
		opsPerGoroutine := 1_000_000

		start := time.Now()

		for i := 0; i < numGoroutines; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				for j := 0; j < opsPerGoroutine; j++ {
					atomic.AddInt64(&counter, 1)
				}
			}()
		}

		wg.Wait()
		elapsed := time.Since(start)

		fmt.Printf("GOMAXPROCS=%d  counter=%d  time=%v\n",
			procs, counter, elapsed)
	}
}
```

**Key Points:**
- With `GOMAXPROCS=1`, goroutines run concurrently but NOT in parallel (single P)
- With `GOMAXPROCS=N`, goroutines can run truly in parallel on N cores
- Atomic operations ensure correctness regardless of GOMAXPROCS
- More Ps doesn't always mean faster — contention on shared atomic can slow things down

---

### Problem 2 — Visualize Work Stealing with Unbalanced Load

**Problem:** Create an unbalanced workload — one goroutine gets 10x more work than others. Show that Go's scheduler distributes work efficiently despite the imbalance. Track which OS thread each goroutine runs on.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	runtime.GOMAXPROCS(4)

	type result struct {
		id       int
		work     int
		duration time.Duration
	}

	results := make(chan result, 10)
	var wg sync.WaitGroup

	// Create unbalanced workloads
	workloads := []int{
		10_000_000, // heavy
		1_000_000,
		1_000_000,
		1_000_000,
		10_000_000, // heavy
		1_000_000,
		1_000_000,
		1_000_000,
	}

	for i, work := range workloads {
		wg.Add(1)
		go func(id, iterations int) {
			defer wg.Done()
			start := time.Now()
			sum := 0
			for j := 0; j < iterations; j++ {
				sum += j % 7
			}
			results <- result{id: id, work: iterations, duration: time.Since(start)}
		}(i, work)
	}

	// Collect results
	go func() {
		wg.Wait()
		close(results)
	}()

	for r := range results {
		label := "light"
		if r.work > 5_000_000 {
			label = "HEAVY"
		}
		fmt.Printf("G%-2d [%s]  work=%10d  took=%v\n",
			r.id, label, r.work, r.duration)
	}

	fmt.Printf("\nWith %d Ps, the scheduler balances heavy and light work\n",
		runtime.GOMAXPROCS(0))
	fmt.Println("via work stealing — idle Ps steal from busy Ps")
}
```

**Key Points:**
- Scheduler balances load across Ps — heavy goroutines don't block light ones
- Work stealing redistributes goroutines from busy P queues to idle ones
- All 4 Ps stay utilized despite uneven initial distribution
- `GOMAXPROCS(0)` queries current setting without changing it
