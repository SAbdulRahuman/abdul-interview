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
