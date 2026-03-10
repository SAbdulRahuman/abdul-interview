# Chapter 24 — Concurrency Patterns

These patterns are reusable solutions for common concurrency problems in Go. They leverage goroutines and channels to build scalable, well-structured concurrent programs.

```
┌──────────────────────────────────────────────────────────┐
│            Common Concurrency Patterns                  │
│                                                          │
│  Worker Pool:                                            │
│  ┌────────┐                                              │
│  │ Jobs   │───► Worker 1 ─┐                              │
│  │ Channel│───► Worker 2 ─┼──► Results Channel           │
│  │        │───► Worker 3 ─┘                              │
│  └────────┘   (N fixed goroutines)                      │
│                                                          │
│  Fan-Out / Fan-In:                                      │
│           ┌──► goroutine 1 ──┐                           │
│  input ───┼──► goroutine 2 ──┼──► merged output          │
│           └──► goroutine 3 ──┘                           │
│  (split work, merge results)                            │
│                                                          │
│  Pipeline:                                               │
│  Stage 1 ──ch──► Stage 2 ──ch──► Stage 3                │
│  (generate)       (process)       (consume)              │
│                                                          │
│  Semaphore:                                              │
│  sem := make(chan struct{}, N) ← limits concurrency to N│
│  sem <- struct{}{}  ← acquire                            │
│  <-sem              ← release                            │
│                                                          │
│  Rate Limiter:                                           │
│  ticker := time.NewTicker(rate)                         │
│  <-ticker.C  ← wait for permission                      │
└──────────────────────────────────────────────────────────┘
```

## Worker Pool

A fixed number of goroutines process jobs from a shared channel.

**Tutorial: Fixed Worker Pool with WaitGroup Coordination**

This example creates 3 worker goroutines that pull jobs from a shared `jobs` channel and push results to a `results` channel. The `sync.WaitGroup` tracks when all workers finish, then closes the results channel so the main goroutine can range over it. Notice how jobs are buffered to allow non-blocking sends, and that `close(jobs)` causes workers' `range` loops to terminate naturally.

```
┌────────────────────────────────────────────────────┐
│   main()                                           │
│   ┌────────────┐     ┌──────────┐                    │
│   │ jobs chan  │────►│ Worker 1 │──┐                 │
│   │ (buffered │────►│ Worker 2 │──┼─► results chan  │
│   │  cap=10)  │────►│ Worker 3 │──┘   (buffered)  │
│   └────────────┘     └──────────┘                    │
│   close(jobs) ►                                     │
│   workers range-exit  wg.Wait()                     │
│                       close(results)                │
│                       main ranges over results      │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(100 * time.Millisecond) // simulate work
        results <- job * 2
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    var wg sync.WaitGroup

    // Start workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs) // no more jobs

    // Wait for workers to finish, then close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for r := range results {
        fmt.Println("Result:", r)
    }
}
```

---

## Fan-In — Merge Multiple Channels

**Tutorial: Merging Multiple Producer Channels into One**

Fan-in combines outputs from multiple independent producer goroutines into a single channel. Each producer runs concurrently and sends to its own channel. The `fanIn` function spawns one goroutine per input channel to forward values to a merged output, using a `WaitGroup` to close the merged channel when all inputs are exhausted. This pattern is essential when you have multiple data sources that a single consumer needs to read from.

```
┌────────────────────────────────────────────────────┐
│  producer(1) ──► ch1 ──┐                             │
│  producer(2) ──► ch2 ──┼──► fanIn() ─► merged chan  │
│  producer(3) ──► ch3 ──┘          │                │
│                                  ▼                │
│                          main ranges over          │
│                          merged channel            │
│                                                    │
│  Inside fanIn():                                   │
│  for each ch:                                      │
│    go func(c) { for v := range c { merged <- v } } │
│  wg.Wait() ─► close(merged)                        │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func producer(id int, count int) <-chan string {
    ch := make(chan string)
    go func() {
        defer close(ch)
        for i := 0; i < count; i++ {
            ch <- fmt.Sprintf("producer-%d: item-%d", id, i)
        }
    }()
    return ch
}

// fanIn merges multiple channels into one
func fanIn(channels ...<-chan string) <-chan string {
    merged := make(chan string)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan string) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}

func main() {
    ch1 := producer(1, 3)
    ch2 := producer(2, 3)
    ch3 := producer(3, 3)

    for msg := range fanIn(ch1, ch2, ch3) {
        fmt.Println(msg)
    }
}
```

---

## Fan-Out — Distribute Tasks

**Tutorial: Distributing Work Across Multiple Workers**

Fan-out is the inverse of fan-in: a single input is distributed across multiple worker goroutines for parallel processing. Here, 10 inputs are sent into a shared `jobs` channel and 4 workers compete to receive and process them. Each worker squares the input value. The `WaitGroup` ensures the `results` channel is closed only after all workers finish. This pattern maximizes throughput for CPU-bound or I/O-bound work.

```
┌────────────────────────────────────────────────────┐
│  inputs [1..10]                                    │
│       │                                            │
│       ▼                                            │
│  ┌────────────┐                                    │
│  │ jobs chan  │  (all inputs sent)                  │
│  └────┬───┬───┘                                    │
│       │   │    competing reads                     │
│   ┌───▼┐ ┌▼───┐ ┌────┐ ┌────┐                    │
│   │ W0 │ │ W1 │ │ W2  │ │ W3  │  4 workers       │
│   └─┬──┘ └─┬─┘ └─┬──┘ └─┬──┘  process(d)=d*d   │
│     └────┼─────┼─────┘                             │
│          ▼                                         │
│   results chan ──► collected by main                │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func process(id int, data int) int {
    return data * data // simulate processing
}

func main() {
    inputs := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    numWorkers := 4

    jobs := make(chan int, len(inputs))
    results := make(chan int, len(inputs))

    var wg sync.WaitGroup

    // Fan-out: start multiple workers
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                results <- process(workerID, job)
            }
        }(w)
    }

    // Send jobs
    for _, input := range inputs {
        jobs <- input
    }
    close(jobs)

    // Close results when all workers done
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect
    for r := range results {
        fmt.Print(r, " ")
    }
    fmt.Println()
}
```

---

## Pipeline Pattern

Each stage reads from an input channel, processes data, and writes to an output channel.

**Tutorial: Multi-Stage Pipeline with Channel Chaining**

This example builds a three-stage pipeline: `generate` produces numbers, `square` transforms them, and `filter` keeps values above a threshold. Each stage runs in its own goroutine and communicates via channels. The pipeline provides natural backpressure — a slow consumer automatically slows the producer. Notice how each stage closes its output channel via `defer close(out)` to signal downstream stages that no more data is coming.

```
┌────────────────────────────────────────────────────┐
│  Stage 1           Stage 2          Stage 3        │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│  │ generate │─ch►│  square  │─ch►│  filter  │    │
│  │ 1..10   │    │  n*n     │    │  n > 20  │    │
│  └──────────┘    └──────────┘    └────┬─────┘    │
│                                      ▼             │
│  Data flow: 5 ─► 25 ─► 25 ✓          main()       │
│              3 ─►  9 ─► (dropped)    fmt.Println  │
│                                                    │
│  Close propagation:                                │
│  close(gen) ► close(sq) ► close(filt) ► range ends  │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Stage 1: generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// Stage 2: square each number
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// Stage 3: filter — keep only values > threshold
func filter(in <-chan int, threshold int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n > threshold {
                out <- n
            }
        }
    }()
    return out
}

func main() {
    // Pipeline: generate → square → filter
    nums := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    squared := square(nums)
    filtered := filter(squared, 20)

    for v := range filtered {
        fmt.Println(v) // 25, 36, 49, 64, 81, 100
    }
}
```

---

## Rate Limiting

**Tutorial: Controlling Request Throughput with Tickers and Burst Buffers**

This example demonstrates two rate limiting strategies. The first uses `time.NewTicker` for steady-rate processing — each request waits for a tick before proceeding. The second creates a bursty rate limiter using a buffered channel pre-filled with tokens, allowing an initial burst of 3 requests before falling back to steady rate. Watch how `<-limiter.C` blocks until the next tick, creating precise timing control.

```
┌────────────────────────────────────────────────────┐
│  Steady Rate (200ms interval):                     │
│                                                    │
│  req1 ── 200ms ── req2 ── 200ms ── req3 ...       │
│  ┌────┐  tick  ┌────┐  tick  ┌────┐              │
│  │ R1 │─────►│ R2 │─────►│ R3 │ ...           │
│  └────┘       └────┘       └────┘              │
│                                                    │
│  Bursty Rate (burst=3, then 200ms):                │
│                                                    │
│  ┌──┐┌──┐┌──┐                                      │
│  │R1││R2││R3│─ instant (pre-filled tokens)          │
│  └──┘└──┘└──┘                                      │
│  ── 200ms ── R4 ── 200ms ── R5                  │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    requests := make(chan int, 10)
    for i := 1; i <= 10; i++ {
        requests <- i
    }
    close(requests)

    // Steady rate: 1 request per 200ms
    limiter := time.NewTicker(200 * time.Millisecond)
    defer limiter.Stop()

    for req := range requests {
        <-limiter.C // wait for tick
        fmt.Println("Processed request", req, "at", time.Now().Format("15:04:05.000"))
    }

    // Bursty rate limiter — allows burst of 3, then steady rate
    fmt.Println("\n--- Bursty Rate Limiter ---")
    burstyLimiter := make(chan time.Time, 3)

    // Pre-fill for burst
    for i := 0; i < 3; i++ {
        burstyLimiter <- time.Now()
    }

    // Then refill at steady rate
    go func() {
        for t := range time.NewTicker(200 * time.Millisecond).C {
            burstyLimiter <- t
        }
    }()

    burstyRequests := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        burstyRequests <- i
    }
    close(burstyRequests)

    for req := range burstyRequests {
        <-burstyLimiter
        fmt.Println("Bursty request", req, "at", time.Now().Format("15:04:05.000"))
    }
}
```

---

## Semaphore — Limit Concurrency

**Tutorial: Buffered Channel as a Counting Semaphore**

This pattern uses a buffered channel of capacity N to limit the number of goroutines executing concurrently. Each goroutine acquires a slot by sending to the channel (`sem <- struct{}{}`) and releases it by receiving (`<-sem`). If the channel is full, the send blocks until a slot opens. This is simpler than `golang.org/x/sync/semaphore` and works well for basic concurrency limiting. The `defer` ensures the slot is released even if the goroutine panics.

```
┌────────────────────────────────────────────────────┐
│  sem := make(chan struct{}, 3)  ← max 3 concurrent  │
│                                                    │
│  Goroutines 1..10:                                 │
│                                                    │
│  ┌───┐ ┌───┐ ┌───┐       ┌───┐ ┌───┐              │
│  │ G1│ │ G2│ │ G3│       │ G4│ │ G5│  (waiting)  │
│  └─┬─┘ └─┬─┘ └─┬─┘       └─┬─┘ └─┬─┘              │
│    │     │     │             │     │ blocked on   │
│  ┌─▼─────▼─────▼─┐         │     │ sem <- {}    │
│  │ sem [#] [#] [#]  │         │     │              │
│  │ (3 slots full)   │         │     │              │
│  └──────────────────┘         │     │              │
│  G1 finishes: <-sem ► G4 enters           ...      │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    const maxConcurrent = 3
    sem := make(chan struct{}, maxConcurrent)

    var wg sync.WaitGroup

    for i := 1; i <= 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            sem <- struct{}{} // acquire semaphore
            defer func() { <-sem }() // release semaphore

            fmt.Printf("Task %d running (concurrent slots used: %d)\n", id, len(sem))
            time.Sleep(200 * time.Millisecond)
        }(i)
    }

    wg.Wait()
    fmt.Println("All tasks completed")
}
```

---

## errgroup — Goroutines with Error Collection

**Tutorial: Managing Goroutine Groups with Error Propagation**

The `errgroup` package from `golang.org/x/sync` provides a structured way to run multiple goroutines and collect the first error. `g.Go(func() error)` launches goroutines, and `g.Wait()` blocks until all finish, returning the first non-nil error. When created with `errgroup.WithContext`, the context is cancelled on the first error, allowing other goroutines to detect the failure and abort early. This is cleaner than manually managing `WaitGroup` + error channels.

```
┌────────────────────────────────────────────────────┐
│  errgroup.WithContext(ctx)                         │
│         │                                          │
│  g.Go ──┼─► fetchURL("example.com")    ✓ ok        │
│  g.Go ──┼─► fetchURL("go.dev")         ✓ ok        │
│  g.Go ──┼─► fetchURL("bad.example")    ✗ error!    │
│  g.Go ──┼─► fetchURL("pkg.go.dev")     ctx cancel │
│         │                                          │
│  g.Wait() ──► returns first error                  │
│               ("failed to fetch bad.example.com")   │
│               ctx cancelled ► other goroutines stop │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
    "errors"

    "golang.org/x/sync/errgroup"
)

func fetchURL(ctx context.Context, url string) error {
    // Simulate fetch
    if url == "https://bad.example.com" {
        return errors.New("failed to fetch " + url)
    }
    fmt.Println("Fetched:", url)
    return nil
}

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    urls := []string{
        "https://example.com",
        "https://go.dev",
        "https://bad.example.com", // this will fail
        "https://pkg.go.dev",
    }

    for _, url := range urls {
        g.Go(func() error {
            return fetchURL(ctx, url)
        })
    }

    // Wait blocks until all goroutines finish.
    // Returns the first non-nil error.
    if err := g.Wait(); err != nil {
        fmt.Println("Error:", err) // Error: failed to fetch https://bad.example.com
    }
}
```

---

## singleflight — Deduplicate Concurrent Calls

**Tutorial: Preventing Thundering Herd with singleflight**

When multiple goroutines request the same expensive resource simultaneously (like a cache miss), `singleflight.Group.Do` ensures only one goroutine executes the function while all others wait and share the result. The `shared` return value indicates whether the result was shared with other callers. This prevents thundering herd problems where N concurrent cache misses trigger N redundant database queries.

```
┌────────────────────────────────────────────────────┐
│  10 goroutines call Do("my-key", fn) simultaneously│
│                                                    │
│  G1 ──┐                                             │
│  G2 ──┼── all same key                              │
│  G3 ──┼──────┐                                       │
│  ... │      ▼                                       │
│  G10 ┘  ┌───────────────────┐                       │
│       │ expensiveLookup()  │  Only ONE executes    │
│       │ (executes once)    │                       │
│       └────────┬──────────┘                       │
│              │  result                              │
│       ┌──────┼───────┐                              │
│       ▼      ▼       ▼                              │
│  G1(shared) G2    G10  ← all get same result      │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"

    "golang.org/x/sync/singleflight"
)

var group singleflight.Group

func expensiveLookup(key string) (string, error) {
    fmt.Println("Actually doing lookup for:", key)
    return "result-for-" + key, nil
}

func main() {
    var wg sync.WaitGroup

    // 10 goroutines all request the same key at once
    // Only ONE actual lookup happens; others share the result
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            // Do deduplicate by key
            val, err, shared := group.Do("my-key", func() (interface{}, error) {
                return expensiveLookup("my-key")
            })

            fmt.Printf("Goroutine %d: val=%s, err=%v, shared=%v\n", id, val, err, shared)
        }(i)
    }

    wg.Wait()
    // "Actually doing lookup for: my-key" prints only ONCE
}
```

---

## Pub/Sub Pattern

**Tutorial: In-Process Publish/Subscribe with Channels**

This publish/subscribe implementation uses a map of topic names to subscriber channel slices. `Subscribe` creates a buffered channel and appends it to the topic. `Publish` sends the message to all channels registered for that topic. `Close` closes all channels for a topic, signaling subscribers to stop. The `RWMutex` ensures concurrent reads (publishes) don't conflict with writes (subscribe/close). Buffered subscriber channels prevent a slow consumer from blocking the publisher.

```
┌────────────────────────────────────────────────────┐
│  PubSub                                            │
│  ┌──────────────────────────────────────────┐    │
│  │ subs["news"] = [ sub1_ch, sub2_ch ]          │    │
│  └──────────────────────────────────────────┘    │
│                                                    │
│  Publish("news", msg)                              │
│       │                                            │
│       ├──► sub1_ch ──► goroutine 1: "Sub1: msg"     │
│       └──► sub2_ch ──► goroutine 2: "Sub2: msg"     │
│                                                    │
│  Close("news")                                     │
│       close(sub1_ch), close(sub2_ch)               │
│       range loops exit naturally                    │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

type PubSub struct {
    mu   sync.RWMutex
    subs map[string][]chan string
}

func NewPubSub() *PubSub {
    return &PubSub{subs: make(map[string][]chan string)}
}

func (ps *PubSub) Subscribe(topic string) <-chan string {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    ch := make(chan string, 10) // buffered to avoid blocking publisher
    ps.subs[topic] = append(ps.subs[topic], ch)
    return ch
}

func (ps *PubSub) Publish(topic, msg string) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()

    for _, ch := range ps.subs[topic] {
        ch <- msg
    }
}

func (ps *PubSub) Close(topic string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    for _, ch := range ps.subs[topic] {
        close(ch)
    }
    delete(ps.subs, topic)
}

func main() {
    ps := NewPubSub()

    sub1 := ps.Subscribe("news")
    sub2 := ps.Subscribe("news")

    var wg sync.WaitGroup
    wg.Add(2)

    go func() {
        defer wg.Done()
        for msg := range sub1 {
            fmt.Println("Sub1:", msg)
        }
    }()

    go func() {
        defer wg.Done()
        for msg := range sub2 {
            fmt.Println("Sub2:", msg)
        }
    }()

    ps.Publish("news", "Breaking: Go is awesome!")
    ps.Publish("news", "Update: New Go release!")
    ps.Close("news")

    wg.Wait()
}
```

---

## Circuit Breaker Pattern

**Tutorial: Protecting Services with a State-Machine Circuit Breaker**

The circuit breaker pattern prevents cascading failures by tracking error counts and transitioning between three states: Closed (normal, requests pass through), Open (too many failures, requests immediately rejected), and Half-Open (testing if the service has recovered). When failures exceed `maxFailures`, the breaker opens. After a timeout, it transitions to half-open to allow a test request. A success resets to closed; a failure re-opens. This avoids hammering a failing service.

```
┌────────────────────────────────────────────────────┐
│     Circuit Breaker State Machine                  │
│                                                    │
│     ┌──────────┐   failures >= max   ┌────────┐    │
│     │  CLOSED  │────────────────►│  OPEN  │    │
│     │ (normal) │                   │ (fail) │    │
│     └────┬─────┘                   └───┬────┘    │
│          ▲                             │          │
│          │  success         timeout elapsed       │
│          │                             │          │
│     ┌────┼───────┐                   │          │
│     │ HALF-OPEN   │◄─────────────────┘          │
│     │  (testing)  │   try one request              │
│     └────────────┘                                │
│     fail ► re-open    success ► closed              │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "errors"
    "fmt"
    "sync"
    "time"
)

type State int

const (
    Closed   State = iota // Normal operation
    Open                  // Failing — reject all requests
    HalfOpen              // Testing — allow limited requests
)

type CircuitBreaker struct {
    mu            sync.Mutex
    state         State
    failures      int
    maxFailures   int
    timeout       time.Duration
    lastFailure   time.Time
}

func NewCircuitBreaker(maxFailures int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:       Closed,
        maxFailures: maxFailures,
        timeout:     timeout,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()

    switch cb.state {
    case Open:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = HalfOpen
            cb.mu.Unlock()
            // Fall through to try the request
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    default:
        cb.mu.Unlock()
    }

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.maxFailures {
            cb.state = Open
        }
        return err
    }

    // Success — reset
    cb.failures = 0
    cb.state = Closed
    return nil
}

func main() {
    cb := NewCircuitBreaker(3, 2*time.Second)

    // Simulate failing service
    failingCall := func() error {
        return errors.New("service unavailable")
    }

    for i := 0; i < 5; i++ {
        err := cb.Execute(failingCall)
        fmt.Printf("Call %d: %v\n", i+1, err)
    }
    // Calls 1-3: service unavailable (failures accumulate)
    // Calls 4-5: circuit breaker is open (fast fail)
}
```

---

## Graceful Shutdown

**Tutorial: Cleanly Stopping an HTTP Server on OS Signals**

This example starts an HTTP server in a goroutine, then waits for `SIGINT` or `SIGTERM` signals. When a signal arrives, it calls `srv.Shutdown(ctx)` with a 5-second timeout, giving in-flight requests time to complete before forcing close. This is critical for production services — abrupt termination can drop active connections and corrupt in-progress work. The `signal.Notify` + channel pattern is the idiomatic way to handle OS signals in Go.

```
┌────────────────────────────────────────────────────┐
│  main()                                            │
│    │                                               │
│    ├─► go srv.ListenAndServe()  (background)       │
│    │         serving requests...                    │
│    │                                               │
│    ├─► signal.Notify(quit, SIGINT, SIGTERM)        │
│    │                                               │
│    ├─► <-quit   (blocks until Ctrl+C / kill)       │
│    │                                               │
│    ├─► ctx, cancel = WithTimeout(5s)               │
│    │                                               │
│    └─► srv.Shutdown(ctx)                           │
│          ├─ stops accepting new connections          │
│          ├─ waits for in-flight requests (≤5s)       │
│          └─ returns (server stopped)                 │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello!")
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // Start server in goroutine
    go func() {
        fmt.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            fmt.Println("Server error:", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    fmt.Println("Shutting down gracefully...")

    // Give active connections 5 seconds to finish
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        fmt.Println("Shutdown error:", err)
    }

    fmt.Println("Server stopped")
}
```

---

## Producer-Consumer

**Tutorial: Classic Producer-Consumer with Buffered Channel**

This example demonstrates the producer-consumer pattern where 2 producers generate random items and 3 consumers process them. The buffered channel acts as a bounded queue, decoupling production speed from consumption speed. The `prodWg` WaitGroup tracks when all producers finish, then closes the channel to signal consumers. The `consWg` ensures main waits for all consumers to drain remaining items before exiting.

```
┌────────────────────────────────────────────────────┐
│  Producer 1 ──┐                                      │
│               ├─► ch (buffered, cap=5)              │
│  Producer 2 ──┘        │                             │
│                       ├──► Consumer 1              │
│  prodWg.Wait()         ├──► Consumer 2              │
│       │                └──► Consumer 3              │
│       ▼                                            │
│  close(ch) ──► consumers' range loops exit          │
│                                                    │
│  consWg.Wait() ──► main exits                      │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

func producer(ch chan<- int, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        item := rand.Intn(100)
        ch <- item
        fmt.Printf("Producer %d: sent %d\n", id, item)
        time.Sleep(time.Duration(rand.Intn(50)) * time.Millisecond)
    }
}

func consumer(ch <-chan int, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    for item := range ch {
        fmt.Printf("Consumer %d: received %d\n", id, item)
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
    }
}

func main() {
    ch := make(chan int, 5) // buffered channel as buffer

    var prodWg, consWg sync.WaitGroup

    // Start producers
    for i := 1; i <= 2; i++ {
        prodWg.Add(1)
        go producer(ch, i, &prodWg)
    }

    // Start consumers
    for i := 1; i <= 3; i++ {
        consWg.Add(1)
        go consumer(ch, i, &consWg)
    }

    // Close channel when all producers done
    go func() {
        prodWg.Wait()
        close(ch)
    }()

    consWg.Wait()
    fmt.Println("Done")
}
```

---

## Timeout & Cancellation

**Tutorial: Context-Based Timeout for Long Operations**

This example uses `context.WithTimeout` to set a 2-second deadline on a 5-second operation. The `longOperation` function uses `select` to race between completing work (`time.After`) and context cancellation (`ctx.Done()`). Whichever fires first wins. Since the timeout (2s) is shorter than the operation (5s), the context is cancelled and `ctx.Err()` returns `context.DeadlineExceeded`. Always call `defer cancel()` to release context resources.

```
┌────────────────────────────────────────────────────┐
│  WithTimeout(ctx, 2s)                              │
│       │                                            │
│       ▼                                            │
│  longOperation(ctx):                               │
│  select {                                          │
│     case <-time.After(5s):  ← completes at 5s      │
│     case <-ctx.Done():      ← fires at 2s  ✓ WINS  │
│  }                                                 │
│       │                                            │
│       ▼                                            │
│  "context deadline exceeded"                       │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func longOperation(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second):
        fmt.Println("Operation completed")
        return nil
    case <-ctx.Done():
        fmt.Println("Operation cancelled:", ctx.Err())
        return ctx.Err()
    }
}

func main() {
    // Timeout-based cancellation
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    err := longOperation(ctx)
    fmt.Println("Result:", err) // context deadline exceeded
}
```

---

## Or-Done Channel Pattern

**Tutorial: Safely Reading from Channels with Cancellation Support**

The or-done pattern wraps a channel read so it also responds to context cancellation. Without it, a goroutine reading from a channel could block indefinitely if the sender stalls or the pipeline shuts down. The `orDone` function uses nested `select` statements: the outer one reads from the input channel or context, and the inner one forwards the value or responds to context cancellation. This prevents goroutine leaks in pipeline architectures.

```
┌────────────────────────────────────────────────────┐
│  Without orDone:               With orDone:         │
│                                                    │
│  producer ──► ch ──► consumer  producer ─► ch        │
│  (stalls)     (blocks!)             │              │
│               goroutine leaks!   orDone(ctx, ch)   │
│                                     │              │
│                                  select {          │
│                                    <-ctx.Done() ✓  │
│                                    <-ch          │  │
│                                  }              │  │
│                                     ▼              │
│                                  consumer (safe) │  │
│                                  cancel() ► exit    │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
)

// orDone wraps a channel read so it also responds to context cancellation.
// Prevents goroutines from blocking when a pipeline shuts down.
func orDone(ctx context.Context, ch <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-ch:
                if !ok {
                    return
                }
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    // Simulate a slow producer
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < 100; i++ {
            ch <- i
        }
    }()

    // Consumer reads with cancellation support
    for val := range orDone(ctx, ch) {
        fmt.Println(val)
        if val == 5 {
            cancel() // stop after 5 values
            break
        }
    }
    // Without orDone, the producer goroutine could leak
}
```

---

## Heartbeat Pattern

**Tutorial: Detecting Stalled Goroutines with Heartbeats**

The heartbeat pattern has a worker goroutine periodically send signals on a heartbeat channel to prove it's alive. The caller monitors this channel and uses a timeout to detect if the worker has stalled. The heartbeat send is non-blocking (`select` with `default`) so a missed heartbeat doesn't block the worker. This is useful in long-running or distributed systems where you need health monitoring of background goroutines.

```
┌────────────────────────────────────────────────────┐
│  heartbeatWorker(ctx, 50ms)                        │
│       │                                            │
│       ├─► heartbeat chan  ──► monitor (main)        │
│       │    (periodic {})       select {             │
│       │                          <-heartbeat: ok    │
│       └─► results chan   ──►      <-results: data   │
│            (work output)          <-timeout: stuck! │
│                                 }                   │
│                                                    │
│  Timeline:                                         │
│  ──┬───┬───┬───┬───┬───┬────────────────────────  │
│    ♥   R   ♥   R   ♥   R  ...                       │
│    beat result beat result                         │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// heartbeatWorker sends periodic heartbeats to signal it's alive.
// Callers can detect stalled goroutines by monitoring heartbeats.
func heartbeatWorker(ctx context.Context, interval time.Duration) (<-chan struct{}, <-chan int) {
    heartbeat := make(chan struct{}, 1)
    results := make(chan int)

    go func() {
        defer close(results)
        defer close(heartbeat)

        tick := time.NewTicker(interval)
        defer tick.Stop()

        for i := 0; i < 5; i++ {
            // Do work
            time.Sleep(80 * time.Millisecond)

            select {
            case <-ctx.Done():
                return
            case <-tick.C:
                // Send heartbeat (non-blocking)
                select {
                case heartbeat <- struct{}{}:
                default: // don't block if nobody is listening
                }
            default:
            }

            select {
            case results <- i:
            case <-ctx.Done():
                return
            }
        }
    }()

    return heartbeat, results
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    heartbeat, results := heartbeatWorker(ctx, 50*time.Millisecond)

    // Monitor heartbeats — detect if worker is stuck
    timeout := time.After(500 * time.Millisecond)
    for {
        select {
        case _, ok := <-heartbeat:
            if !ok {
                return
            }
            fmt.Println("heartbeat received")
        case r, ok := <-results:
            if !ok {
                fmt.Println("work done")
                return
            }
            fmt.Println("result:", r)
        case <-timeout:
            fmt.Println("worker appears stuck!")
            return
        }
    }
}
```

---

## Bridge Channel — Channel of Channels

**Tutorial: Flattening a Channel of Channels into a Single Stream**

The bridge pattern takes a `<-chan <-chan int` (a channel that produces channels) and flattens it into a single `<-chan int`. It consumes each inner channel sequentially, forwarding all values to the output. This is useful when pipeline stages produce batches as separate channels — the bridge lets the consumer read a single unified stream. The `done` channel enables clean shutdown by breaking out of the forwarding loop.

```
┌────────────────────────────────────────────────────┐
│  chanStream (<-chan <-chan int)                      │
│    │                                               │
│    ├─► ch-A [0, 1, 2]                              │
│    ├─► ch-B [3, 4, 5]                              │
│    └─► ch-C [6, 7, 8]                              │
│                                                    │
│  bridge(done, chanStream)                          │
│    │                                               │
│    ▼  out (<-chan int)                              │
│    0, 1, 2, 3, 4, 5, 6, 7, 8  (flattened)         │
│                                                    │
│  Reads ch-A to completion, then ch-B, then ch-C   │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// bridge flattens a channel of channels into a single channel.
// Useful when multiple stages produce channels of results.
func bridge(done <-chan struct{}, chanStream <-chan <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            var stream <-chan int
            select {
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return
                }
                stream = maybeStream
            case <-done:
                return
            }
            for val := range stream {
                select {
                case out <- val:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}

func main() {
    done := make(chan struct{})
    defer close(done)

    // Create a channel that produces channels
    chanStream := make(chan (<-chan int))
    go func() {
        defer close(chanStream)
        for i := 0; i < 3; i++ {
            ch := make(chan int, 3)
            for j := i * 3; j < (i+1)*3; j++ {
                ch <- j
            }
            close(ch)
            chanStream <- ch
        }
    }()

    // Bridge flattens it into a single stream
    for val := range bridge(done, chanStream) {
        fmt.Print(val, " ") // 0 1 2 3 4 5 6 7 8
    }
    fmt.Println()
}
```

---

## Tee Channel — Split One Channel Into Two

**Tutorial: Duplicating a Channel Stream to Two Independent Consumers**

The tee pattern splits one input channel into two output channels, where each output receives every value from the input. It uses a clever technique: after reading a value, it sends to both outputs using a `select` with local copies. After sending to one, it nils that copy to prevent double-sending, ensuring both outputs get the value before proceeding to the next input. This is useful when two consumers need the same data stream.

```
┌────────────────────────────────────────────────────┐
│                                                    │
│  input ch: [1, 2, 3, 4, 5]                         │
│       │                                            │
│       ▼                                            │
│  tee(done, ch)                                     │
│       ├──► out1: [1, 2, 3, 4, 5]  ─► consumer A    │
│       │                                            │
│       └──► out2: [1, 2, 3, 4, 5]  ─► consumer B    │
│                                                    │
│  Both outputs receive ALL values from input.        │
│  o1, o2 := out1, out2                              │
│  select { o1<-val ► o1=nil; o2<-val ► o2=nil }    │
│  (send to both, nil after each send)               │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// tee splits one input channel into two output channels.
// Each output receives every value from the input.
func tee(done <-chan struct{}, in <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    go func() {
        defer close(out1)
        defer close(out2)
        for val := range in {
            // Use local copies to allow independent sends
            o1, o2 := out1, out2
            for i := 0; i < 2; i++ {
                select {
                case o1 <- val:
                    o1 = nil // disable after sending
                case o2 <- val:
                    o2 = nil
                case <-done:
                    return
                }
            }
        }
    }()
    return out1, out2
}

func main() {
    done := make(chan struct{})
    defer close(done)

    ch := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch)

    out1, out2 := tee(done, ch)

    // Both outputs get every value
    go func() {
        for v := range out1 {
            fmt.Println("out1:", v)
        }
    }()
    for v := range out2 {
        fmt.Println("out2:", v)
    }
}
```

---

## Interview Questions

1. **What is the fan-out/fan-in pattern?**
   - Fan-out: launch multiple goroutines to process work in parallel. Fan-in: merge results from multiple channels into one. Used for parallelizing CPU-bound or I/O-bound work.

2. **What is the pipeline pattern in Go?**
   - A series of stages connected by channels. Each stage is a goroutine that receives from inbound, processes, and sends to outbound. Enables streaming data processing with backpressure.

3. **How do you implement a worker pool in Go?**
   - Create N goroutines reading from a shared jobs channel and writing to a results channel. The producer sends jobs, workers process concurrently. Use `sync.WaitGroup` to know when all workers finish.

4. **What is the "done channel" pattern?**
   - A `chan struct{}` used for cancellation signaling. Close the channel to broadcast cancellation to all goroutines selecting on it. Superseded by `context.Context` in modern Go.

5. **How do you prevent goroutine leaks?**
   - Always ensure goroutines can exit: use context cancellation, done channels, or close input channels. Monitor with `runtime.NumGoroutine()`. Common leak: goroutine blocked on channel no one reads/writes.

6. **What is the semaphore pattern?**
   - Use a buffered channel of capacity N as a semaphore: `sem := make(chan struct{}, N)`. Acquire: `sem <- struct{}{}`. Release: `<-sem`. Limits concurrent access to a resource.

7. **What is `errgroup` and how does it work?**
   - `golang.org/x/sync/errgroup` manages a group of goroutines. `g.Go(func() error)` launches work. `g.Wait()` blocks until all complete, returning the first error. Supports context cancellation.

8. **What is the "or-done" channel pattern?**
   - Wraps a channel read to also respond to cancellation: select on both the data channel and done/context channel. Prevents blocking when the pipeline is shutting down.

9. **How do you implement rate limiting in Go?**
   - Use `time.Ticker` for fixed-rate limiting or `golang.org/x/time/rate.Limiter` for token bucket. `limiter.Wait(ctx)` blocks until allowed. `limiter.Allow()` for non-blocking check.

10. **What is the pub/sub pattern in Go?**
    - Publishers send events to a broker, subscribers receive via channels. Each subscriber has its own channel. The broker fans out messages. Use mutex or channels to manage subscriber registration.

11. **What is the or-done channel pattern?**
    - Wraps a channel read to also respond to context cancellation: select on both the data channel and done/ctx channel. Prevents goroutines from blocking forever when a pipeline shuts down.

12. **What is the heartbeat pattern?**
    - A goroutine periodically sends signals on a heartbeat channel to prove it's alive. The caller monitors heartbeats and can detect stalled/stuck goroutines via timeout.

13. **What is the bridge channel pattern?**
    - Flattens a channel of channels (`<-chan <-chan T`) into a single channel (`<-chan T`). Useful when pipeline stages produce channels of results that need to be consumed sequentially.

14. **What is the tee channel pattern?**
    - Splits one input channel into two output channels. Every value from the input is sent to both outputs. Useful when multiple consumers need the same data stream.

15. **What is `singleflight` used for?**
    - `golang.org/x/sync/singleflight` deduplicates concurrent calls with the same key. Only one goroutine executes the function; others wait and share the result. Prevents thundering herd problems (e.g., cache misses).
