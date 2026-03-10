# Chapter 35 — Race Conditions & Data Races in Go

Race conditions are the most dangerous class of concurrency bugs. Go provides built-in tooling (the race detector) and language-level constructs to prevent them. Understanding races is critical for interviews and production Go.

## Data Race vs Race Condition

```
┌────────────────────────────────────────────────────────────┐
│  Data Race vs Race Condition                               │
│                                                            │
│  Data Race:                                                │
│  Two goroutines access the same variable concurrently      │
│  AND at least one is a write.                              │
│  → Undefined behavior. Go race detector catches these.     │
│  → Always a bug. Must be fixed.                            │
│                                                            │
│  Race Condition:                                           │
│  Program correctness depends on timing/ordering of events. │
│  → May or may not involve a data race.                     │
│  → Logical bug — harder to detect automatically.           │
│                                                            │
│  Example distinction:                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Data Race:    counter++ from two goroutines          │  │
│  │               (unsynchronized read-modify-write)     │  │
│  │                                                      │  │
│  │ Race Cond:    if !fileExists() { createFile() }      │  │
│  │  (no data     Another goroutine creates it between   │  │
│  │   race, but   the check and creation — TOCTOU bug)   │  │
│  │   wrong                                              │  │
│  │   behavior)                                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  All data races are race conditions.                       │
│  Not all race conditions are data races.                   │
└────────────────────────────────────────────────────────────┘
```

---

## The Go Race Detector

**Tutorial: Built-in Detection with `-race`**

Go has a built-in race detector powered by ThreadSanitizer. Enable it with the `-race` flag on `go run`, `go test`, or `go build`. It instruments memory accesses at compile time and detects data races at runtime.

```
┌────────────────────────────────────────────────────────────┐
│  Using the Race Detector                                   │
│                                                            │
│  go run -race main.go       ← run with race detection     │
│  go test -race ./...        ← test with race detection    │
│  go build -race -o myapp    ← build instrumented binary   │
│                                                            │
│  Performance impact:                                       │
│  • 2-20x slower execution                                  │
│  • 5-10x more memory                                       │
│  • Use in CI/testing, NOT production                       │
│                                                            │
│  Output on detection:                                      │
│  ==================                                        │
│  WARNING: DATA RACE                                        │
│  Write at 0x00c0000b4010 by goroutine 7:                   │
│    main.main.func1()                                       │
│        /app/main.go:12 +0x38                               │
│                                                            │
│  Previous read at 0x00c0000b4010 by goroutine 6:           │
│    main.main.func2()                                       │
│        /app/main.go:18 +0x2c                               │
│  ==================                                        │
│                                                            │
│  ⚠️ Race detector only finds races that actually execute.  │
│  It cannot find races in code paths not exercised.         │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// Run with: go run -race main.go
	counter := 0
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++ // DATA RACE: unsynchronized read-modify-write
		}()
	}

	wg.Wait()
	fmt.Println("Counter:", counter) // Will NOT be 1000 — undefined behavior
}
```

**Output with `-race`:**
```
==================
WARNING: DATA RACE
Write at 0x00c0000b4010 by goroutine 8:
  main.main.func1()
      /app/main.go:16 +0x38

Previous write at 0x00c0000b4010 by goroutine 7:
  main.main.func1()
      /app/main.go:16 +0x38
==================
```

---

## Classic Data Race Patterns

### Pattern 1: Shared Counter

```
┌────────────────────────────────────────────────────────────┐
│  The Shared Counter Race                                   │
│                                                            │
│  counter = 0                                               │
│                                                            │
│  Goroutine A              Goroutine B                      │
│  ──────────              ──────────                        │
│  READ counter (0)                                          │
│                          READ counter (0)                   │
│  ADD 1 → 1                                                 │
│                          ADD 1 → 1                          │
│  WRITE counter = 1                                         │
│                          WRITE counter = 1                  │
│                                                            │
│  Expected: 2    Actual: 1   ← Lost update!                │
│                                                            │
│  The ++ operator is NOT atomic:                            │
│  counter++ = read + increment + write (3 steps)            │
└────────────────────────────────────────────────────────────┘
```

**Three fixes for the shared counter:**

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	const N = 10000

	// ══════════════════════════════════
	// Fix 1: sync.Mutex
	// ══════════════════════════════════
	var mu sync.Mutex
	counterMu := 0
	var wg1 sync.WaitGroup
	for i := 0; i < N; i++ {
		wg1.Add(1)
		go func() {
			defer wg1.Done()
			mu.Lock()
			counterMu++
			mu.Unlock()
		}()
	}
	wg1.Wait()
	fmt.Println("Mutex counter:", counterMu) // 10000

	// ══════════════════════════════════
	// Fix 2: sync/atomic
	// ══════════════════════════════════
	var counterAtomic int64
	var wg2 sync.WaitGroup
	for i := 0; i < N; i++ {
		wg2.Add(1)
		go func() {
			defer wg2.Done()
			atomic.AddInt64(&counterAtomic, 1)
		}()
	}
	wg2.Wait()
	fmt.Println("Atomic counter:", counterAtomic) // 10000

	// ══════════════════════════════════
	// Fix 3: Channel (accumulator pattern)
	// ══════════════════════════════════
	counterCh := 0
	ch := make(chan int, N)
	var wg3 sync.WaitGroup
	for i := 0; i < N; i++ {
		wg3.Add(1)
		go func() {
			defer wg3.Done()
			ch <- 1 // send increment
		}()
	}

	// Collect in a single goroutine — no race
	go func() {
		wg3.Wait()
		close(ch)
	}()

	for v := range ch {
		counterCh += v
	}
	fmt.Println("Channel counter:", counterCh) // 10000
}
```

### Pattern 2: Concurrent Map Access

```
┌────────────────────────────────────────────────────────────┐
│  Map Race — FATAL at Runtime                               │
│                                                            │
│  Go maps are NOT goroutine-safe.                           │
│  Concurrent read+write or write+write → panic (Go 1.6+)   │
│                                                            │
│  fatal error: concurrent map writes                        │
│  fatal error: concurrent map read and map write            │
│                                                            │
│  Fixes:                                                    │
│  1. sync.Mutex / sync.RWMutex around map operations        │
│  2. sync.Map (optimized for specific patterns)             │
│  3. Channel-based access (single owner goroutine)          │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
)

// ❌ RACE — will panic
func unsafeMapAccess() {
	m := make(map[string]int)
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			key := fmt.Sprintf("key-%d", n%10)
			m[key] = n // RACE: concurrent map writes → panic
		}(i)
	}
	wg.Wait()
}

// ✅ Fix 1: RWMutex-protected map
type SafeMap struct {
	mu sync.RWMutex
	m  map[string]int
}

func NewSafeMap() *SafeMap {
	return &SafeMap{m: make(map[string]int)}
}

func (s *SafeMap) Set(key string, value int) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.m[key] = value
}

func (s *SafeMap) Get(key string) (int, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	v, ok := s.m[key]
	return v, ok
}

// ✅ Fix 2: sync.Map (best when keys are stable or append-only)
func syncMapExample() {
	var m sync.Map
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			key := fmt.Sprintf("key-%d", n%10)
			m.Store(key, n)
		}(i)
	}
	wg.Wait()

	m.Range(func(key, value any) bool {
		fmt.Printf("%s = %v\n", key, value)
		return true
	})
}

func main() {
	sm := NewSafeMap()
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			sm.Set(fmt.Sprintf("key-%d", n%10), n)
		}(i)
	}
	wg.Wait()

	v, _ := sm.Get("key-5")
	fmt.Println("SafeMap key-5:", v)

	syncMapExample()
}
```

### Pattern 3: Loop Variable Capture (Pre Go 1.22)

```
┌────────────────────────────────────────────────────────────┐
│  Loop Variable Capture Race (Go < 1.22)                    │
│                                                            │
│  for i := 0; i < 5; i++ {                                 │
│      go func() {                                           │
│          fmt.Println(i)  ← captures &i, not value          │
│      }()                                                   │
│  }                                                         │
│  // May print: 5 5 5 5 5 (all see final value)            │
│                                                            │
│  Fix (pre Go 1.22): pass as parameter                      │
│  for i := 0; i < 5; i++ {                                 │
│      go func(n int) {                                      │
│          fmt.Println(n)  ← n is a copy                     │
│      }(i)                                                  │
│  }                                                         │
│                                                            │
│  Go 1.22+: loop variables are per-iteration by default.    │
│  The old bug is fixed at the language level.               │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	// Go 1.22+: this is safe — each iteration gets its own `i`
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Printf("i=%d ", i)
		}()
	}
	wg.Wait()
	fmt.Println()

	// Pre Go 1.22 fix: shadow the variable
	for i := 0; i < 5; i++ {
		i := i // shadow — creates new variable per iteration
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Printf("i=%d ", i)
		}()
	}
	wg.Wait()
	fmt.Println()
}
```

### Pattern 4: Check-Then-Act (TOCTOU)

```
┌────────────────────────────────────────────────────────────┐
│  TOCTOU: Time-of-Check to Time-of-Use                     │
│                                                            │
│  Goroutine A               Goroutine B                     │
│  ──────────               ──────────                       │
│  if val, ok := m[k]; !ok {                                │
│      // "k not in map"                                     │
│                           m[k] = "other value"             │
│      m[k] = "my value"   // OVERWRITES B's value          │
│  }                                                         │
│                                                            │
│  The check (ok) and act (m[k]=v) must be atomic.          │
│  Fix: hold lock across both check AND act.                 │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
)

type Registry struct {
	mu    sync.Mutex
	items map[string]string
}

func NewRegistry() *Registry {
	return &Registry{items: make(map[string]string)}
}

// ❌ WRONG — check and act are not atomic
func (r *Registry) RegisterUnsafe(key, value string) bool {
	r.mu.Lock()
	_, exists := r.items[key]
	r.mu.Unlock() // releases lock between check and act!

	if !exists {
		r.mu.Lock()
		r.items[key] = value // another goroutine may have registered between unlock/lock
		r.mu.Unlock()
		return true
	}
	return false
}

// ✅ RIGHT — check and act under same lock
func (r *Registry) Register(key, value string) bool {
	r.mu.Lock()
	defer r.mu.Unlock()

	if _, exists := r.items[key]; exists {
		return false // already registered
	}
	r.items[key] = value
	return true
}

func main() {
	reg := NewRegistry()
	var wg sync.WaitGroup

	// Race to register the same key
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			value := fmt.Sprintf("goroutine-%d", n)
			if reg.Register("service", value) {
				fmt.Printf("%s won the registration\n", value)
			}
		}(i)
	}
	wg.Wait()

	fmt.Println("Final value:", reg.items["service"])
}
```

### Pattern 5: Slice Append Race

```
┌────────────────────────────────────────────────────────────┐
│  Slice Append Race                                         │
│                                                            │
│  Slices have (ptr, len, cap). Concurrent append can:       │
│  • Corrupt slice header (data race on len)                 │
│  • Cause lost elements                                     │
│  • Panic from out-of-bounds access                         │
│                                                            │
│  Goroutine A               Goroutine B                     │
│  ──────────               ──────────                       │
│  read len=3                                                │
│                           read len=3                        │
│  write s[3] = "a"                                          │
│  set len=4                                                 │
│                           write s[3] = "b"  ← overwrites! │
│                           set len=4         ← lost "a"     │
│                                                            │
│  Fix: mutex, or collect via channel                        │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// ❌ RACE: concurrent append to shared slice
	// results := []int{}
	// for i := 0; i < 100; i++ {
	//     go func(n int) {
	//         results = append(results, n) // RACE
	//     }(i)
	// }

	// ✅ Fix 1: Mutex
	var mu sync.Mutex
	var results []int
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			mu.Lock()
			results = append(results, n)
			mu.Unlock()
		}(i)
	}
	wg.Wait()
	fmt.Println("Mutex results count:", len(results)) // 100

	// ✅ Fix 2: Pre-allocate with index (no append needed)
	results2 := make([]int, 100)
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			results2[n] = n * 2 // each goroutine writes to its own index — safe!
		}(i)
	}
	wg.Wait()
	fmt.Println("Index results[50]:", results2[50]) // 100

	// ✅ Fix 3: Channel collector
	ch := make(chan int, 100)
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			ch <- n
		}(i)
	}
	go func() {
		wg.Wait()
		close(ch)
	}()

	var results3 []int
	for v := range ch {
		results3 = append(results3, v)
	}
	fmt.Println("Channel results count:", len(results3)) // 100
}
```

---

## sync.Mutex vs sync.RWMutex

```
┌────────────────────────────────────────────────────────────┐
│  Mutex Types                                               │
│                                                            │
│  sync.Mutex:                                               │
│  • Exclusive lock — only one goroutine at a time           │
│  • Use when reads AND writes need protection               │
│  • Simpler, lower overhead per operation                   │
│                                                            │
│  sync.RWMutex:                                             │
│  • Multiple concurrent readers OR one exclusive writer     │
│  • RLock() / RUnlock() for read access                     │
│  • Lock() / Unlock() for write access                      │
│  • Use when reads greatly outnumber writes                 │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Read-heavy workload (100 readers, rare writes):     │  │
│  │                                                      │  │
│  │  Mutex:   readers serialize → slow                   │  │
│  │  RWMutex: readers run concurrently → fast            │  │
│  │                                                      │  │
│  │  Write-heavy workload:                               │  │
│  │  RWMutex has slightly more overhead than Mutex       │  │
│  │  → prefer plain Mutex                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ⚠️ Common mistake: holding a lock while doing I/O        │
│  → Blocks all other goroutines waiting for the lock       │
│  → Keep critical sections short!                           │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Cache with RWMutex — many readers, few writers
type Cache struct {
	mu   sync.RWMutex
	data map[string]string
}

func NewCache() *Cache {
	return &Cache{data: make(map[string]string)}
}

func (c *Cache) Get(key string) (string, bool) {
	c.mu.RLock() // multiple readers can hold RLock simultaneously
	defer c.mu.RUnlock()
	v, ok := c.data[key]
	return v, ok
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock() // exclusive — blocks all readers and writers
	defer c.mu.Unlock()
	c.data[key] = value
}

func main() {
	cache := NewCache()
	cache.Set("greeting", "hello")

	var wg sync.WaitGroup

	// 100 concurrent readers — all run in parallel with RWMutex
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			v, _ := cache.Get("greeting")
			_ = v
		}()
	}

	// Occasional writer
	wg.Add(1)
	go func() {
		defer wg.Done()
		time.Sleep(1 * time.Millisecond)
		cache.Set("greeting", "world")
	}()

	wg.Wait()
	v, _ := cache.Get("greeting")
	fmt.Println("Final:", v)
}
```

---

## sync/atomic Package

```
┌────────────────────────────────────────────────────────────┐
│  sync/atomic — Lock-Free Atomic Operations                 │
│                                                            │
│  Atomic operations execute as a single, indivisible unit.  │
│  No mutex needed. Hardware-level guaranteed.               │
│                                                            │
│  Key operations:                                           │
│  atomic.AddInt64(&val, delta)    ← atomic increment        │
│  atomic.LoadInt64(&val)          ← atomic read             │
│  atomic.StoreInt64(&val, new)    ← atomic write            │
│  atomic.CompareAndSwapInt64(&val, old, new) ← CAS          │
│  atomic.SwapInt64(&val, new)     ← swap, return old        │
│                                                            │
│  Go 1.19+: atomic.Int64, atomic.Bool (typed wrappers)     │
│  var counter atomic.Int64                                  │
│  counter.Add(1)                                            │
│  counter.Load()                                            │
│  counter.Store(0)                                          │
│                                                            │
│  When to use:                                              │
│  ✅ Simple counters, flags, pointers                        │
│  ✅ Performance-critical hot paths                          │
│  ❌ Complex multi-field updates (use mutex)                 │
│  ❌ Operations that need to read-modify-write atomically   │
│     beyond what CAS provides                               │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	// Go 1.19+ typed atomics
	var counter atomic.Int64
	var active atomic.Bool
	var wg sync.WaitGroup

	active.Store(true)

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if active.Load() {
				counter.Add(1)
			}
		}()
	}

	wg.Wait()
	fmt.Println("Counter:", counter.Load()) // 1000

	// CAS (Compare-And-Swap) — only update if current value matches expected
	var state atomic.Int32
	state.Store(0) // IDLE

	// Try to transition from IDLE(0) to RUNNING(1)
	if state.CompareAndSwap(0, 1) {
		fmt.Println("Started! State:", state.Load()) // 1
	}

	// Another attempt fails — already RUNNING
	if !state.CompareAndSwap(0, 1) {
		fmt.Println("Already running, state:", state.Load())
	}
}
```

---

## sync.Once — Exactly-Once Initialization

```go
package main

import (
	"fmt"
	"sync"
)

type DBConn struct {
	DSN string
}

var (
	dbOnce sync.Once
	dbConn *DBConn
)

func GetDB() *DBConn {
	dbOnce.Do(func() {
		fmt.Println("Initializing DB connection (runs exactly once)")
		dbConn = &DBConn{DSN: "postgres://localhost/mydb"}
	})
	return dbConn
}

func main() {
	var wg sync.WaitGroup

	// 100 goroutines call GetDB — init runs exactly once
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			db := GetDB()
			_ = db
		}()
	}
	wg.Wait()

	fmt.Println("DSN:", GetDB().DSN)
	// Output:
	// Initializing DB connection (runs exactly once)
	// DSN: postgres://localhost/mydb
}
```

---

## Real-World Race Pattern: Goroutine Leak with Unbuffered Channel

```
┌────────────────────────────────────────────────────────────┐
│  Goroutine Leak via Unbuffered Channel                     │
│                                                            │
│  func fetchFirst(urls []string) string {                   │
│      ch := make(chan string)  // unbuffered!                │
│      for _, url := range urls {                            │
│          go func(u string) {                               │
│              ch <- fetch(u)  // blocks if no receiver      │
│          }(url)                                            │
│      }                                                     │
│      return <-ch  // takes first result, returns           │
│  }                                                         │
│  // Problem: N-1 goroutines are stuck sending to ch        │
│  // forever — goroutine leak!                              │
│                                                            │
│  Fix: Use buffered channel with capacity = len(urls)       │
│  ch := make(chan string, len(urls))                        │
│  // All goroutines can send without blocking               │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"time"
)

// ❌ LEAKS goroutines
func fetchFirstLeaky(tasks []string) string {
	ch := make(chan string) // unbuffered
	for _, t := range tasks {
		go func(task string) {
			time.Sleep(10 * time.Millisecond) // simulate work
			ch <- task + "-done"               // blocks forever for losers
		}(t)
	}
	return <-ch // only reads one, N-1 goroutines leak
}

// ✅ SAFE — buffered channel, all goroutines can complete
func fetchFirstSafe(tasks []string) string {
	ch := make(chan string, len(tasks)) // buffered
	for _, t := range tasks {
		go func(task string) {
			time.Sleep(10 * time.Millisecond)
			ch <- task + "-done" // never blocks — buffer has room
		}(t)
	}
	return <-ch // reads first, others drain naturally via GC
}

func main() {
	tasks := []string{"task-1", "task-2", "task-3"}
	result := fetchFirstSafe(tasks)
	fmt.Println("First result:", result)
}
```

---

## Channel Ownership Pattern

```
┌────────────────────────────────────────────────────────────┐
│  Channel Ownership Rule                                    │
│                                                            │
│  The goroutine that CREATES a channel should:              │
│  1. Be the only writer (or coordinate writers)             │
│  2. Close the channel when done                            │
│  3. Never close a channel from the reader side             │
│                                                            │
│  ❌ Multiple goroutines closing same channel → panic       │
│  ❌ Writing to a closed channel → panic                    │
│  ✅ Reading from a closed channel → returns zero value     │
│                                                            │
│  Pattern: Producer creates, writes, and closes.            │
│           Consumer only reads.                             │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Producer owns the channel — creates, writes, closes
func produce(n int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch) // producer closes
		for i := 0; i < n; i++ {
			ch <- i
		}
	}()
	return ch
}

// Multiple producers with WaitGroup to coordinate close
func fanIn(channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup

	for _, ch := range channels {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(ch)
	}

	go func() {
		wg.Wait()
		close(out) // close after all producers done
	}()

	return out
}
```

Wait, let me fix this — I need the sync import:

```go
package main

import (
	"fmt"
	"sync"
)

func produce(n int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		for i := 0; i < n; i++ {
			ch <- i
		}
	}()
	return ch
}

func fanIn(channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup

	for _, ch := range channels {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(ch)
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func main() {
	ch1 := produce(3) // 0, 1, 2
	ch2 := produce(3) // 0, 1, 2

	for v := range fanIn(ch1, ch2) {
		fmt.Print(v, " ")
	}
	fmt.Println()
}
```

---

## Race-Free Design Patterns Summary

```
┌────────────────────────────────────────────────────────────────┐
│  How to Avoid Races — Design Patterns                          │
│                                                                │
│  1. Don't share state                                          │
│     • Each goroutine works on its own copy                     │
│     • Pass data via channels, not shared variables             │
│                                                                │
│  2. Confine state to one goroutine                             │
│     • Only one goroutine reads/writes the variable             │
│     • Others communicate via channels                          │
│     • "Don't communicate by sharing memory;                    │
│       share memory by communicating"                           │
│                                                                │
│  3. Use sync primitives                                        │
│     • sync.Mutex / sync.RWMutex — general purpose              │
│     • sync/atomic — simple counters, flags                     │
│     • sync.Once — one-time initialization                      │
│     • sync.WaitGroup — wait for goroutine completion           │
│     • sync.Map — concurrent map (specific use cases)           │
│                                                                │
│  4. Immutability                                               │
│     • Return new values instead of modifying shared ones       │
│     • Struct copies are safe (if no pointer fields)            │
│                                                                │
│  Order of preference:                                          │
│  channels > confinement > atomic > RWMutex > Mutex             │
│  (simplest/safest first)                                       │
└────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **What is a data race in Go?**
   - Two or more goroutines access the same variable concurrently, and at least one access is a write, without synchronization. Causes undefined behavior. Detected by `go run -race`.

2. **How does the Go race detector work?**
   - Instrumented at compile time via `-race` flag. Uses ThreadSanitizer to track every memory access and detect unsynchronized concurrent accesses at runtime. Only detects races that actually execute — can't find races in untested code paths. 2-20x slowdown.

3. **What's the difference between data race and race condition?**
   - Data race: unsynchronized concurrent access to shared memory (at least one write). Race condition: program correctness depends on goroutine scheduling order. All data races are race conditions, but not all race conditions are data races (e.g., TOCTOU).

4. **Why is `map` not safe for concurrent use?**
   - Maps are not designed for concurrent access. Since Go 1.6, concurrent read+write or write+write triggers a runtime panic (`fatal error: concurrent map writes`). Fix with `sync.Mutex`, `sync.RWMutex`, or `sync.Map`.

5. **When should you use `sync.RWMutex` vs `sync.Mutex`?**
   - `RWMutex` when reads greatly outnumber writes — lets multiple readers proceed concurrently. If writes are frequent, the overhead of `RWMutex` makes plain `Mutex` better. Profile before deciding.

6. **When should you use `sync/atomic` vs `sync.Mutex`?**
   - Atomic for single variables (counters, flags, pointers). Mutex for protecting multi-step operations or multiple variables that must change together. Atomic is faster but limited in scope.

7. **What is a goroutine leak? How do you prevent it?**
   - A goroutine that blocks forever (e.g., sending on unbuffered channel with no receiver). Prevented by: buffered channels, context cancellation, `select` with `done` channel. Monitor with `runtime.NumGoroutine()`.

8. **What does `sync.Once` guarantee?**
   - The function passed to `Do()` executes exactly once, even if called from multiple goroutines concurrently. All calls block until the first call completes. Commonly used for lazy initialization.

---

## Interview Problems & Solutions

### Problem 1 — Thread-Safe Rate Limiter

**Problem:** Build a goroutine-safe rate limiter that allows at most N requests per second using a token bucket algorithm. It must be safe for concurrent use.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

type RateLimiter struct {
	tokens     atomic.Int64
	maxTokens  int64
	refillRate int64 // tokens per second
	stopCh     chan struct{}
}

func NewRateLimiter(maxTokens, refillRate int64) *RateLimiter {
	rl := &RateLimiter{
		maxTokens:  maxTokens,
		refillRate: refillRate,
		stopCh:     make(chan struct{}),
	}
	rl.tokens.Store(maxTokens)

	// Background refill goroutine
	go rl.refill()
	return rl
}

func (rl *RateLimiter) refill() {
	ticker := time.NewTicker(time.Second / time.Duration(rl.refillRate))
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			for {
				current := rl.tokens.Load()
				if current >= rl.maxTokens {
					break // already full
				}
				// CAS: only increment if still below max
				if rl.tokens.CompareAndSwap(current, current+1) {
					break
				}
				// CAS failed — another goroutine changed it, retry
			}
		case <-rl.stopCh:
			return
		}
	}
}

func (rl *RateLimiter) Allow() bool {
	for {
		current := rl.tokens.Load()
		if current <= 0 {
			return false // no tokens available
		}
		// CAS: decrement if still has tokens
		if rl.tokens.CompareAndSwap(current, current-1) {
			return true
		}
		// CAS failed — retry (another goroutine consumed a token)
	}
}

func (rl *RateLimiter) Stop() {
	close(rl.stopCh)
}

func main() {
	limiter := NewRateLimiter(5, 5) // 5 max tokens, refill 5/sec
	defer limiter.Stop()

	var wg sync.WaitGroup
	var allowed, denied atomic.Int64

	// Simulate 20 concurrent requests
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			if limiter.Allow() {
				allowed.Add(1)
				fmt.Printf("Request %d: ALLOWED\n", id)
			} else {
				denied.Add(1)
				fmt.Printf("Request %d: DENIED\n", id)
			}
		}(i)
	}

	wg.Wait()
	fmt.Printf("\nAllowed: %d, Denied: %d\n", allowed.Load(), denied.Load())
}
```

**Key Points:**
- `atomic.Int64` for lock-free token tracking
- `CompareAndSwap` loop handles concurrent token consumption without mutex
- Background goroutine refills tokens; stopped via `stopCh` channel to prevent goroutine leak
- No mutex needed — purely atomic operations

---

### Problem 2 — Concurrent-Safe LRU Cache

**Problem:** Implement a thread-safe LRU cache that supports concurrent `Get` and `Put` operations. Use an `RWMutex` for read-heavy workloads.

```go
package main

import (
	"container/list"
	"fmt"
	"sync"
)

type entry struct {
	key   string
	value any
}

type LRUCache struct {
	mu       sync.RWMutex
	capacity int
	items    map[string]*list.Element
	order    *list.List // front = most recent
}

func NewLRUCache(capacity int) *LRUCache {
	return &LRUCache{
		capacity: capacity,
		items:    make(map[string]*list.Element),
		order:    list.New(),
	}
}

func (c *LRUCache) Get(key string) (any, bool) {
	c.mu.Lock() // need write lock because we modify order
	defer c.mu.Unlock()

	if elem, ok := c.items[key]; ok {
		c.order.MoveToFront(elem) // mark as recently used
		return elem.Value.(*entry).value, true
	}
	return nil, false
}

func (c *LRUCache) Put(key string, value any) {
	c.mu.Lock()
	defer c.mu.Unlock()

	// Update existing
	if elem, ok := c.items[key]; ok {
		c.order.MoveToFront(elem)
		elem.Value.(*entry).value = value
		return
	}

	// Evict if at capacity
	if c.order.Len() >= c.capacity {
		oldest := c.order.Back()
		if oldest != nil {
			c.order.Remove(oldest)
			delete(c.items, oldest.Value.(*entry).key)
		}
	}

	// Insert new
	e := &entry{key: key, value: value}
	elem := c.order.PushFront(e)
	c.items[key] = elem
}

func (c *LRUCache) Len() int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.order.Len()
}

func main() {
	cache := NewLRUCache(3)

	var wg sync.WaitGroup

	// Concurrent writes
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			key := fmt.Sprintf("key-%d", n)
			cache.Put(key, n*10)
		}(i)
	}
	wg.Wait()

	fmt.Println("Cache size:", cache.Len()) // 3 (capacity limit)

	// Concurrent reads
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			key := fmt.Sprintf("key-%d", n)
			if v, ok := cache.Get(key); ok {
				fmt.Printf("  %s = %v\n", key, v)
			}
		}(i)
	}
	wg.Wait()
}
```

**Key Points:**
- `Get` uses `Lock()` not `RLock()` because it mutates the LRU order (`MoveToFront`)
- `Len()` uses `RLock()` since it's a pure read
- Eviction happens under the write lock — check-then-act is atomic
- `container/list` provides O(1) removal and move-to-front
