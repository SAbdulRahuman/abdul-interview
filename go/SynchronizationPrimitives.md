# Chapter 13 — Synchronization Primitives

The `sync` package provides traditional synchronization primitives for when channels aren't the right fit — such as protecting shared state, caching, and object pooling.

```
┌──────────────────────────────────────────────────────────┐
│         When to Use Mutex vs Channel                    │
│                                                          │
│  Use Mutex when:              Use Channel when:          │
│  ┌──────────────────────┐     ┌──────────────────────┐   │
│  │ Protecting shared    │     │ Passing data between │   │
│  │ state (counters,     │     │ goroutines           │   │
│  │ maps, caches)        │     │                      │   │
│  │                      │     │ Signaling events     │   │
│  │ Simple lock/unlock   │     │ (done, cancel)       │   │
│  │ around data access   │     │                      │   │
│  │                      │     │ Coordinating pipeline│   │
│  │ Performance-critical │     │ stages               │   │
│  │ hot paths            │     │                      │   │
│  └──────────────────────┘     └──────────────────────┘   │
│                                                          │
│  sync Primitives:                                        │
│  ├── Mutex      — exclusive lock (1 goroutine at a time)│
│  ├── RWMutex    — multiple readers OR 1 writer          │
│  ├── Once       — run exactly once (lazy init)          │
│  ├── WaitGroup  — wait for N goroutines to finish       │
│  ├── Cond       — signal/broadcast to waiting goroutines│
│  ├── Map        — concurrent-safe map                   │
│  ├── Pool       — object reuse pool (reduce GC)        │
│  └── atomic     — lock-free operations on primitives    │
└──────────────────────────────────────────────────────────┘
```

## sync.Mutex

**Tutorial: Protecting Shared State with sync.Mutex**

Mutex provides mutually exclusive access to shared state. `Lock()` acquires exclusive access and `Unlock()` releases it — only one goroutine can hold the lock at a time, others block. This example wraps a counter in a struct with a Mutex, using `defer mu.Unlock()` to guarantee release even if a panic occurs.

```
┌──────────────────────────────────────────────────────────┐
│          sync.Mutex — Exclusive Lock                     │
│                                                          │
│  G1: Lock() ──► │ count++ │ ──► Unlock()                │
│                 └─────────┘                              │
│  G2: Lock() ─── BLOCKS ──────────────►│ count++ │─►Unlock│
│                                        └─────────┘       │
│  G3: Lock() ─── BLOCKS ──────────────────────────►...    │
│                                                          │
│  Only ONE goroutine inside critical section at a time    │
│  defer mu.Unlock() ensures unlock even on panic          │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func main() {
    counter := &SafeCounter{}
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }

    wg.Wait()
    fmt.Println("Final count:", counter.Value()) // Always 1000
}
```

---

## sync.RWMutex

**Tutorial: Multiple Readers or Single Writer with RWMutex**

RWMutex allows multiple concurrent readers OR a single exclusive writer. `RLock()` acquires a shared read lock — multiple goroutines can hold it simultaneously. `Lock()` acquires an exclusive write lock that blocks all readers and writers. Use RWMutex when reads vastly outnumber writes, such as caches.

```
┌──────────────────────────────────────────────────────────┐
│          sync.RWMutex — Readers/Writer Lock              │
│                                                          │
│  Multiple readers (concurrent):                          │
│  R1: RLock() ──► read data ──► RUnlock()                │
│  R2: RLock() ──► read data ──► RUnlock()  ← simultaneous│
│  R3: RLock() ──► read data ──► RUnlock()                │
│                                                          │
│  Writer (exclusive):                                     │
│  W1: Lock() ──► write data ──► Unlock()                 │
│  R4: RLock() ─── BLOCKS ──────────────► read ──► done  │
│  R5: RLock() ─── BLOCKS ──────────────► read ──► done  │
│                                                          │
│  Use when reads >> writes (e.g., caches)                 │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// RWMutex allows multiple concurrent READERS or a single WRITER
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewCache() *Cache {
    return &Cache{data: make(map[string]string)}
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()         // multiple goroutines can RLock simultaneously
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()          // exclusive lock — blocks all readers and writers
    defer c.mu.Unlock()
    c.data[key] = value
}

func main() {
    cache := NewCache()
    cache.Set("greeting", "hello")

    var wg sync.WaitGroup

    // Multiple concurrent readers
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            val, _ := cache.Get("greeting")
            fmt.Printf("Reader %d: %s\n", id, val)
            time.Sleep(10 * time.Millisecond)
        }(i)
    }

    wg.Wait()
}
```

---

## sync.Once

**Tutorial: One-Time Initialization with sync.Once**

Guarantees a function executes exactly once, regardless of how many goroutines call it concurrently. This is the idiomatic Go singleton pattern — all goroutines that call `dbOnce.Do(init)` will block until the first invocation completes, then receive the same result. Subsequent calls are no-ops.

```
┌──────────────────────────────────────────────────────────┐
│         sync.Once — Execute Exactly Once                 │
│                                                          │
│  G1 ──► dbOnce.Do(init) ──► init() RUNS ──► dbInstance  │
│  G2 ──► dbOnce.Do(init) ──► skipped (already ran)       │
│  G3 ──► dbOnce.Do(init) ──► skipped (already ran)       │
│  ...                                                     │
│  G10 ─► dbOnce.Do(init) ──► skipped                     │
│                                                          │
│  All goroutines receive the SAME *Database pointer       │
│  Init function runs exactly once, even under contention  │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

// Singleton pattern using sync.Once
type Database struct {
    connectionString string
}

var (
    dbInstance *Database
    dbOnce    sync.Once
)

func GetDatabase() *Database {
    dbOnce.Do(func() {
        // This runs EXACTLY once, even with concurrent calls
        fmt.Println("Initializing database connection...")
        dbInstance = &Database{
            connectionString: "postgres://localhost:5432/mydb",
        }
    })
    return dbInstance
}

func main() {
    var wg sync.WaitGroup

    // 10 goroutines all try to get the database
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            db := GetDatabase()
            fmt.Printf("Goroutine %d got DB: %p\n", id, db)
        }(i)
    }

    wg.Wait()
    // "Initializing database connection..." printed EXACTLY once
    // All goroutines get the SAME pointer
}
```

---

## Mutex Deadlock — Lock Ordering

**Tutorial: Avoiding Deadlock with Consistent Lock Ordering**

Deadlock occurs when two goroutines acquire the same mutexes in opposite order: G1 holds mu1 and waits for mu2, while G2 holds mu2 and waits for mu1 — neither can proceed. The fix is simple: always acquire multiple locks in a consistent global order. This example demonstrates the problem and lists prevention strategies.

```
┌──────────────────────────────────────────────────────────┐
│       Deadlock: Inconsistent Lock Ordering               │
│                                                          │
│  G1:  Lock(mu1) ──► Sleep ──► Lock(mu2)   BLOCKS ──┐    │
│                                                     │    │
│  G2:  Lock(mu2) ──► Sleep ──► Lock(mu1)   BLOCKS ──┘    │
│                                                          │
│       G1 holds mu1, waits for mu2                        │
│       G2 holds mu2, waits for mu1                        │
│       → DEADLOCK! Neither can proceed.                   │
│                                                          │
│  Fix: Always lock in same order (mu1 → mu2)             │
│  G1:  Lock(mu1) → Lock(mu2) → work → Unlock both       │
│  G2:  Lock(mu1) → Lock(mu2) → work → Unlock both       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Deadlock: two goroutines acquire the same locks in different order
func main() {
    var mu1, mu2 sync.Mutex

    // Goroutine 1: locks mu1 then mu2
    go func() {
        mu1.Lock()
        defer mu1.Unlock()
        time.Sleep(10 * time.Millisecond) // increase chance of deadlock
        mu2.Lock()
        defer mu2.Unlock()
        fmt.Println("G1: locked both")
    }()

    // Goroutine 2: locks mu2 then mu1 — DEADLOCK risk!
    go func() {
        mu2.Lock()
        defer mu2.Unlock()
        time.Sleep(10 * time.Millisecond)
        mu1.Lock()
        defer mu1.Unlock()
        fmt.Println("G2: locked both")
    }()

    time.Sleep(1 * time.Second)
    // May deadlock! Fix: ALWAYS acquire locks in the same order

    // Prevention strategies:
    // 1. Always acquire multiple mutexes in a consistent global order
    // 2. Use a single mutex instead of multiple
    // 3. Use channels instead of mutexes
    // 4. Use context with timeout to detect stalls
    // 5. Keep critical sections small — lock, do work, unlock
}
```

---

## Mutex TryLock (Go 1.18+)

**Tutorial: Non-Blocking Lock Attempts with TryLock**

`TryLock()` attempts to acquire a lock without blocking. It returns `true` if the lock was acquired, `false` if it's already held. This is useful for opportunistic locking and deadlock avoidance, but rarely the right tool — prefer regular `Lock()` with proper design in most cases.

```
┌──────────────────────────────────────────────────────────┐
│         TryLock — Non-Blocking Lock Attempt              │
│                                                          │
│  mu.Lock()     ← mu is now held                          │
│                                                          │
│  mu.TryLock()  → false  (lock busy, don't block)        │
│                  "doing something else"                   │
│                                                          │
│  mu.Unlock()   ← mu is now free                          │
│                                                          │
│  mu.TryLock()  → true   (lock acquired!)                │
│  mu.Unlock()                                             │
│                                                          │
│  Also: rw.TryRLock() for read locks                      │
│  NOTE: Prefer Lock() in most cases                       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var mu sync.Mutex

    mu.Lock()

    // TryLock: non-blocking attempt to acquire the lock
    // Returns true if lock acquired, false if already locked
    if mu.TryLock() {
        fmt.Println("Acquired lock")
        mu.Unlock()
    } else {
        fmt.Println("Lock is busy, doing something else") // this path
    }

    mu.Unlock()

    // Now it's free
    if mu.TryLock() {
        fmt.Println("Acquired lock on second try") // this path
        mu.Unlock()
    }

    // RWMutex also has TryLock() and TryRLock()
    var rw sync.RWMutex
    if rw.TryRLock() {
        fmt.Println("Acquired read lock")
        rw.RUnlock()
    }

    // NOTE: TryLock is rarely the right tool.
    // It exists for edge cases (deadlock avoidance, opportunistic locking).
    // Prefer regular Lock() with proper design.
}
```

---

## sync.Map

**Tutorial: Concurrent-Safe Map with sync.Map**

The `sync.Map` type provides a concurrent-safe map without external locking. It is optimized for cases where keys are mostly stable (written once, read many times) or goroutines access disjoint key sets. For frequent writes or when you need typed keys/values, prefer a regular `map` protected by `sync.RWMutex`.

```
┌──────────────────────────────────────────────────────────┐
│          sync.Map — Concurrent Safe Map                  │
│                                                          │
│  Store("key1","v1")   Load("key1") → "v1", true         │
│  Store("key2", 42)    Load("key9") → nil, false         │
│                                                          │
│  LoadOrStore("key1","new")  → "v1", true  (existed)      │
│  LoadOrStore("key4","new")  → "new", false (stored)     │
│                                                          │
│  Delete("key3")                                          │
│  Range(func(k,v) bool {...})  ← iterate all entries     │
│                                                          │
│  Best for: stable keys, read-heavy, disjoint access     │
│  Otherwise: map + sync.RWMutex is usually better         │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // sync.Map is a concurrent-safe map
    // Best when: keys are mostly stable, or goroutines access disjoint key sets
    var m sync.Map

    // Store
    m.Store("key1", "value1")
    m.Store("key2", 42)
    m.Store("key3", true)

    // Load
    val, ok := m.Load("key1")
    fmt.Println(val, ok) // value1 true

    // LoadOrStore — load existing or store new
    actual, loaded := m.LoadOrStore("key1", "new_value")
    fmt.Println(actual, loaded) // value1 true (already existed)

    actual, loaded = m.LoadOrStore("key4", "new_value")
    fmt.Println(actual, loaded) // new_value false (was stored)

    // Delete
    m.Delete("key3")

    // Range — iterate all entries
    m.Range(func(key, value any) bool {
        fmt.Printf("%s: %v\n", key, value)
        return true // return false to stop iteration
    })

    // When to use sync.Map vs Mutex + map:
    // sync.Map: keys are stable, lots of reads, few writes
    // Mutex + map: frequent writes, need more control, typed keys/values
}
```

---

## sync.Pool

**Tutorial: Reducing GC Pressure with sync.Pool**

An object pool that caches temporary objects for reuse, reducing garbage collection pressure. Call `Get()` to retrieve (or create) an object, and `Put()` to return it for reuse. Always `Reset()` objects before returning them. Pool objects may be evicted at any GC cycle — don't rely on persistence.

```
┌──────────────────────────────────────────────────────────┐
│          sync.Pool — Object Reuse                        │
│                                                          │
│  pool.Get()                pool.Put(buf)                 │
│  ┌──────────┐              ┌──────────┐                  │
│  │ Pool     │──► *Buffer   │ Pool     │◄── *Buffer       │
│  │ (empty?) │   (new!)     │          │   (reused!)      │
│  │ call New │              │ stores   │                  │
│  └──────────┘              └──────────┘                  │
│                                                          │
│  Get → reuse or create     Put → return for reuse        │
│  buf.Reset() before Put!   Objects may be GC'd anytime   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

func main() {
    // sync.Pool reuses objects, reducing GC pressure
    bufferPool := &sync.Pool{
        New: func() any {
            fmt.Println("Creating new buffer")
            return new(bytes.Buffer)
        },
    }

    // Get a buffer from pool (or create new one)
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.WriteString("Hello, Pool!")
    fmt.Println(buf.String()) // Hello, Pool!

    // Reset and return to pool for reuse
    buf.Reset()
    bufferPool.Put(buf)

    // Get again — reuses the returned buffer (no "Creating new buffer")
    buf2 := bufferPool.Get().(*bytes.Buffer)
    buf2.WriteString("Reused!")
    fmt.Println(buf2.String()) // Reused!

    // NOTE: Pool objects may be garbage collected at any time
    // Don't rely on them persisting — they're for optimization only
}
```

---

## sync.OnceFunc, sync.OnceValue, sync.OnceValues (Go 1.21+)

**Tutorial: Cleaner Once Patterns with OnceFunc/OnceValue**

Go 1.21 introduced convenience wrappers: `OnceFunc` wraps a function to run once with subsequent calls being no-ops, `OnceValue` caches a single return value, and `OnceValues` caches a `(T, error)` pair. These are cleaner alternatives to using `sync.Once` directly. If the function panics, every subsequent call panics with the same value.

```
┌──────────────────────────────────────────────────────────┐
│     sync.Once* Convenience Wrappers (Go 1.21+)          │
│                                                          │
│  OnceFunc(f):       f() → runs once, subsequent = no-op │
│  OnceValue(f):      f() → T cached, returned every call │
│  OnceValues(f):     f() → (T, error) cached             │
│                                                          │
│  init := OnceFunc(setup)                                 │
│  init()  → runs setup                                    │
│  init()  → no-op                                         │
│                                                          │
│  getConfig := OnceValue(loadConfig)                      │
│  cfg := getConfig()  → computes once                     │
│  cfg := getConfig()  → returns cached                    │
│                                                          │
│  If f panics → every subsequent call panics same value   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // sync.OnceFunc — wraps a func() to only run once
    // Cleaner alternative to sync.Once for simple cases
    init := sync.OnceFunc(func() {
        fmt.Println("Initialized!") // prints once
    })
    init() // runs
    init() // no-op
    init() // no-op

    // sync.OnceValue — wraps a func() T to compute a value once
    getConfig := sync.OnceValue(func() map[string]string {
        fmt.Println("Loading config...") // runs once
        return map[string]string{"env": "production"}
    })
    cfg1 := getConfig() // computes
    cfg2 := getConfig() // returns cached
    fmt.Println(cfg1["env"], cfg2["env"]) // production production

    // sync.OnceValues — wraps a func() (T, error)
    loadDB := sync.OnceValues(func() (string, error) {
        fmt.Println("Connecting to DB...") // runs once
        return "db-connection", nil
    })
    conn1, err1 := loadDB()
    conn2, err2 := loadDB() // returns cached
    fmt.Println(conn1, err1, conn2, err2)

    // If the function panics, every subsequent call panics with same value
}
```

---

## sync.Cond

**Tutorial: Coordinating Goroutines with Condition Variables**

A condition variable lets goroutines wait for a condition to become true. `Wait()` atomically releases the lock and suspends the goroutine; when woken, it re-acquires the lock. `Broadcast()` wakes ALL waiting goroutines, while `Signal()` wakes just one. Always check the condition in a `for` loop (not `if`) to handle spurious wakeups.

```
┌──────────────────────────────────────────────────────────┐
│        sync.Cond — Condition Variable                    │
│                                                          │
│  Workers:                      Signaler:                 │
│  ┌─────────────────┐          ┌──────────────────┐       │
│  │ mu.Lock()       │          │ mu.Lock()         │      │
│  │ for !ready {    │          │ ready = true      │      │
│  │   cond.Wait() ──│── waits  │ mu.Unlock()       │      │
│  │ }               │          │ cond.Broadcast() ─│─► wake│
│  │ "proceeding!"   │◄─────── │                   │      │
│  │ mu.Unlock()     │          └──────────────────┘       │
│  └─────────────────┘                                     │
│                                                          │
│  Wait() = Unlock + Sleep + re-Lock (atomic)              │
│  Broadcast() = wake ALL;  Signal() = wake ONE            │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    // sync.Cond — condition variable for coordinating goroutines
    var mu sync.Mutex
    cond := sync.NewCond(&mu)

    ready := false

    // Waiting goroutines
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            mu.Lock()
            for !ready {
                cond.Wait() // atomically unlocks mu, waits, re-locks mu
            }
            fmt.Printf("Worker %d: proceeding!\n", id)
            mu.Unlock()
        }(i)
    }

    // Signal all waiters
    time.Sleep(100 * time.Millisecond)
    mu.Lock()
    ready = true
    mu.Unlock()
    cond.Broadcast() // wake ALL waiting goroutines
    // cond.Signal()  // wakes ONE waiting goroutine

    wg.Wait()
}
```

---

## atomic Package

**Tutorial: Lock-Free Operations with sync/atomic**

Atomic operations provide the fastest synchronization for simple operations like counters and flags — they're lock-free and don't require a Mutex. `AddInt64` atomically increments, `CompareAndSwap` enables lock-free algorithms, and `atomic.Value` lets you store/load any type atomically for patterns like hot-swapping configuration.

```
┌──────────────────────────────────────────────────────────┐
│          atomic Package — Lock-Free Operations           │
│                                                          │
│  AddInt64(&counter, 1)     atomic increment              │
│  LoadInt64(&counter)       atomic read                   │
│  StoreInt64(&counter, 0)   atomic write                  │
│                                                          │
│  CompareAndSwapInt64(&v, old, new):                      │
│  ┌────────────────────────────────┐                      │
│  │ if v == old → v = new (true)  │  single atomic op    │
│  │ if v != old → no-op  (false)  │                      │
│  └────────────────────────────────┘                      │
│                                                          │
│  atomic.Value → Store/Load any type atomically           │
│  Faster than Mutex for simple counter/flag operations    │
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
    // Atomic operations — lock-free, fastest synchronization
    var counter int64

    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1) // atomic increment
        }()
    }
    wg.Wait()
    fmt.Println("Counter:", atomic.LoadInt64(&counter)) // Always 1000

    // CompareAndSwap — set value only if it matches expected
    var status int64 = 0
    swapped := atomic.CompareAndSwapInt64(&status, 0, 1) // if status==0, set to 1
    fmt.Println("Swapped:", swapped, "Status:", status)   // true 1

    swapped = atomic.CompareAndSwapInt64(&status, 0, 2)   // status is 1, not 0
    fmt.Println("Swapped:", swapped, "Status:", status)   // false 1

    // atomic.Value — store/load any type atomically
    var config atomic.Value
    config.Store(map[string]string{"env": "production"})

    // Can be read from any goroutine without locks
    cfg := config.Load().(map[string]string)
    fmt.Println("Env:", cfg["env"]) // production
}
```

---

## Type-Safe Atomic Types (Go 1.19+)

**Tutorial: Modern Atomic Types with Methods**

Go 1.19 introduced type-safe atomic wrappers: `atomic.Bool`, `atomic.Int64`, `atomic.Pointer[T]`, etc. They provide methods like `.Add()`, `.Load()`, `.Store()`, and `.CompareAndSwap()` directly on the type, eliminating the need for passing pointers. The compiler enforces correct usage, making code cleaner and less error-prone.

```
┌──────────────────────────────────────────────────────────┐
│    Type-Safe Atomic Types (Go 1.19+)                     │
│                                                          │
│  Old style:                  New style:                  │
│  atomic.AddInt64(&c, 1)      var c atomic.Int64          │
│  atomic.LoadInt64(&c)        c.Add(1)                    │
│                              c.Load()                    │
│                                                          │
│  Types: atomic.Bool, Int32, Int64, Uint32, Uint64        │
│         atomic.Pointer[T]                                │
│                                                          │
│  ┌────────────────────┐  ┌────────────────────┐          │
│  │ var p atomic.       │  │ p.Store(&Config{}) │          │
│  │   Pointer[Config]  │  │ cfg := p.Load()    │          │
│  └────────────────────┘  └────────────────────┘          │
│  Compiler-enforced type safety + cleaner API             │
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
    // Go 1.19+ introduced type-safe atomic wrappers
    // Cleaner API vs the older function-based style

    // atomic.Bool
    var isRunning atomic.Bool
    isRunning.Store(true)
    fmt.Println("Running:", isRunning.Load())       // true
    fmt.Println("Was:", isRunning.Swap(false))       // true (returns old value)
    fmt.Println("Now:", isRunning.Load())             // false

    // atomic.Int64 (also Int32, Uint32, Uint64, Uintptr)
    var counter atomic.Int64
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Add(1) // cleaner than atomic.AddInt64(&val, 1)
        }()
    }
    wg.Wait()
    fmt.Println("Counter:", counter.Load()) // 1000

    // CompareAndSwap on type-safe types
    counter.Store(10)
    swapped := counter.CompareAndSwap(10, 20)
    fmt.Println("CAS:", swapped, counter.Load()) // true 20

    // atomic.Pointer[T] — type-safe pointer atomics
    type Config struct {
        Debug bool
        Addr  string
    }
    var cfgPtr atomic.Pointer[Config]
    cfgPtr.Store(&Config{Debug: true, Addr: ":8080"})

    cfg := cfgPtr.Load()
    fmt.Println("Config:", cfg.Addr) // :8080

    // Advantages over old function-based API:
    // - No need for &variable (methods on the type)
    // - Compiler enforces correct usage
    // - Cleaner, more readable code
}
```

---

## Mutex vs Channel — When to Use Which

**Tutorial: Choosing Between Mutex and Channel**

This is the key decision guide for Go concurrency. Use mutexes when protecting shared state (counters, maps, structs) — they're simpler and faster for that purpose. Use channels when passing data between goroutines, coordinating lifecycle, or building pipelines. The Go proverb says "share memory by communicating," but don't force channels where a mutex is simpler.

```
┌──────────────────────────────────────────────────────────┐
│    Mutex vs Channel Decision Guide                       │
│                                                          │
│  ┌─── Use Mutex ───────────┐ ┌─── Use Channel ────────┐ │
│  │ • Protect shared state  │ │ • Pass data between G's│ │
│  │ • Counters, maps, cache │ │ • Signal done/cancel   │ │
│  │ • Simple lock/unlock    │ │ • Pipeline stages      │ │
│  │ • Performance critical  │ │ • Fan-in / Fan-out     │ │
│  │ • Read-heavy (RWMutex)  │ │ • Timeouts (select)    │ │
│  └───────────────────────┘ └───────────────────────┘ │
│                                                          │
│  "Share memory by communicating"                         │
│   but don't force channels where mutex is simpler!       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // USE MUTEX when:
    // - Protecting shared state (counters, maps, structs)
    // - Simple lock/unlock semantics
    // - Performance critical (mutexes are faster)

    // USE CHANNELS when:
    // - Communicating between goroutines (passing data)
    // - Coordinating goroutine lifecycle (signaling done)
    // - Building pipelines or fan-in/fan-out patterns
    // - Implementing timeouts and cancellation

    // Go proverb: "Don't communicate by sharing memory;
    //              share memory by communicating."

    // But this doesn't mean "always use channels"!
    // Choose the right tool for the job.

    fmt.Println("Mutex: protect shared state")
    fmt.Println("Channel: communicate between goroutines")
}
```

---

## Interview Questions

1. **What is the difference between `sync.Mutex` and `sync.RWMutex`?**
   - `Mutex` allows only one goroutine at a time (exclusive lock). `RWMutex` allows multiple concurrent readers (`RLock`) OR one exclusive writer (`Lock`). Use `RWMutex` when reads vastly outnumber writes.

2. **What is `sync.Once` and what is it used for?**
   - `sync.Once` ensures a function is executed exactly once, regardless of how many goroutines call it. Common use: singleton initialization, one-time setup.

3. **What is `sync.WaitGroup`?**
   - A counter-based synchronization primitive: `Add(n)` increments, `Done()` decrements, `Wait()` blocks until counter reaches zero. Used to wait for a group of goroutines to complete.

4. **When should you use `sync.Map` vs a regular map with mutex?**
   - `sync.Map` is optimized for two patterns: (1) keys are written once and read many times, (2) disjoint sets of keys per goroutine. For other patterns, a regular `map` with `sync.RWMutex` is usually better.

5. **What is `sync.Pool` used for?**
   - It caches temporary objects for reuse, reducing GC pressure. Objects may be evicted at any GC cycle. Common use: reusing buffers, HTTP request objects, etc.

6. **What is `sync.Cond`?**
   - A condition variable for goroutine coordination. `Wait()` releases the lock and suspends. `Signal()` wakes one waiting goroutine. `Broadcast()` wakes all. Used for producer-consumer patterns.

7. **What are atomic operations and when should you use them?**
   - `sync/atomic` provides lock-free operations on simple types (`AddInt64`, `LoadInt64`, `StoreInt64`, `CompareAndSwap`). Use for simple counters/flags — they're faster than mutexes but limited to primitive operations.

8. **What happens if you copy a `sync.Mutex`?**
   - The copy retains the lock state of the original — if the original was locked, the copy is also locked. This is a bug. `go vet` detects this. Never copy a Mutex; pass by pointer.

9. **Can you lock a mutex in one goroutine and unlock in another?**
   - Technically yes, but it's strongly discouraged. The convention is to lock and unlock in the same goroutine, often with `defer mu.Unlock()`.

10. **When should you use channels vs mutexes?**
    - Mutexes: protecting shared state, controlling access to a resource. Channels: passing data between goroutines, signaling completion/cancellation, building pipelines. Rule of thumb: if you're protecting data, use mutex; if you're communicating, use channels.

11. **What is `Mutex.TryLock()` (Go 1.18+)?**
    - A non-blocking attempt to acquire a lock. Returns `true` if acquired, `false` if already held. Rarely the right choice — it exists for deadlock avoidance and opportunistic locking. RWMutex also has `TryRLock()`.

12. **How do mutex deadlocks happen and how do you prevent them?**
    - Deadlock occurs when goroutines acquire multiple locks in different orders. Prevention: always lock in a consistent global order, minimize lock scope, prefer a single lock, or use channels instead.

13. **What are `sync.OnceFunc` and `sync.OnceValue` (Go 1.21+)?**
    - `OnceFunc(f)` returns a function that calls `f` once. `OnceValue(f)` calls `f` once and caches the result. `OnceValues(f)` does the same for `(T, error)`. Cleaner alternatives to `sync.Once`.

14. **What are the type-safe atomic types in Go 1.19+?**
    - `atomic.Bool`, `atomic.Int32`, `atomic.Int64`, `atomic.Uint32`, `atomic.Uint64`, `atomic.Pointer[T]`. They provide methods like `.Add()`, `.Load()`, `.Store()`, `.CompareAndSwap()` — cleaner and safer than the older function-based API.
