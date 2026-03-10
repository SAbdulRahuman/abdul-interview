# Chapter 12 — Channels

Channels are Go's primary mechanism for communication between goroutines — following the philosophy "Don't communicate by sharing memory; share memory by communicating." Channels are typed, thread-safe conduits that can be unbuffered (synchronous) or buffered (asynchronous).

```
┌──────────────────────────────────────────────────────────┐
│           Channel Types and Behavior                    │
│                                                          │
│  Unbuffered: ch := make(chan T)                           │
│  ┌──────────┐     ┌──────────┐                           │
│  │ Sender   │────►│ Receiver │  Both must be ready       │
│  │ blocks   │     │ blocks   │  Synchronous handshake    │
│  └──────────┘     └──────────┘                           │
│                                                          │
│  Buffered: ch := make(chan T, 3)                          │
│  ┌──────────┐  ┌───┬───┬───┐  ┌──────────┐              │
│  │ Sender   │──►│ 1 │ 2 │ 3 │──►│ Receiver │             │
│  │ blocks   │  └───┴───┴───┘  │ blocks   │              │
│  │ when full│  buffer (FIFO)  │ when empty│              │
│  └──────────┘                  └──────────┘              │
│                                                          │
│  Directional:                                            │
│  chan<- T    Send-only   (producer side)                  │
│  <-chan T    Receive-only (consumer side)                 │
│  chan T      Bidirectional (converted implicitly)         │
│                                                          │
│  Closing: close(ch)                                      │
│  • Subsequent sends → PANIC                              │
│  • Subsequent receives → zero value + false              │
│  • range ch → iterates until closed                      │
│  • Only sender should close                              │
└──────────────────────────────────────────────────────────┘
```

## Unbuffered Channels

**Tutorial: Unbuffered Channel Synchronous Handshake**

Unbuffered channels act as a synchronous rendezvous point: the sender blocks until a receiver is ready, and the receiver blocks until a sender delivers a value. Both goroutines must arrive at the channel operation at the same time for the transfer to happen. This provides a strong synchronization guarantee.

```
┌──────────────────────────────────────────────────────────┐
│         Unbuffered Channel Handshake                     │
│                                                          │
│  Goroutine                            main               │
│  ┌──────────────────┐                ┌─────────────┐    │
│  │ ch <- "hello"    │                │ Sleep(100ms)  │   │
│  │  (BLOCKS here    │◄── handshake ──►│              │   │
│  │   until recv)    │                │ msg := <-ch   │   │
│  │                  │                │  (BLOCKS here │   │
│  │ "sent!" printed  │                │   until send) │   │
│  └──────────────────┘                └─────────────┘    │
│                                                          │
│  Timeline:                                               │
│  G:  ───send(blocks)──────────────────handshake──►print      │
│  M:  ───sleep──────────────recv(unblocks G)──►print      │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Unbuffered channel — SYNCHRONOUS
    // Send blocks until someone receives, and vice versa
    ch := make(chan string)

    go func() {
        fmt.Println("Goroutine: about to send...")
        ch <- "hello" // blocks until main receives
        fmt.Println("Goroutine: sent!")
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main: about to receive...")
    msg := <-ch // blocks until goroutine sends
    fmt.Println("Main: received:", msg)

    // Output order:
    // Goroutine: about to send...
    // Main: about to receive...
    // Goroutine: sent!
    // Main: received: hello
}
```

---

## Buffered Channels

**Tutorial: Buffered Channels — Asynchronous Up to Capacity**

Buffered channels hold up to N values without blocking the sender. Sends only block when the buffer is full; receives only block when the buffer is empty. This makes them behave like a FIFO queue between goroutines. Use `len(ch)` to check current fill and `cap(ch)` for the capacity.

```
┌──────────────────────────────────────────────────────────┐
│        Buffered Channel (capacity 3)                     │
│                                                          │
│  ch <- 1      ch <- 2      ch <- 3     ch <- 4          │
│                                          BLOCKS!         │
│  ┌───┬───┬───┐                                           │
│  │ 1 │ 2 │ 3 │  ← buffer full (len=3, cap=3)            │
│  └───┴───┴───┘                                           │
│                                                          │
│  <-ch → 1     <-ch → 2     <-ch → 3                     │
│  ┌───┬───┬───┐                                           │
│  │   │   │   │  ← buffer empty (len=0, cap=3)           │
│  └───┴───┴───┘                                           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // Buffered channel — ASYNCHRONOUS up to capacity
    ch := make(chan int, 3) // buffer holds 3 values

    // Can send without a receiver (up to capacity)
    ch <- 1
    ch <- 2
    ch <- 3
    // ch <- 4  // This would BLOCK — buffer is full

    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3

    fmt.Println("Length:", len(ch)) // 0 (empty now)
    fmt.Println("Capacity:", cap(ch)) // 3
}
```

---

## Directional Channels

**Tutorial: Enforcing Channel Direction at Compile Time**

Restricting a channel to send-only (`chan<- T`) or receive-only (`<-chan T`) enforces correct usage at compile time. A bidirectional `chan T` is implicitly convertible to either direction when passed to a function. This pattern ensures producers can only send and consumers can only receive, preventing accidental misuse.

```
┌──────────────────────────────────────────────────────────┐
│       Directional Channel Type Conversion                │
│                                                          │
│   ch := make(chan int, 5)    bidirectional                │
│          │                                               │
│    ┌─────┴──────┐                                        │
│    ▼             ▼                                       │
│  chan<- int    <-chan int                                  │
│  (send-only)  (receive-only)                             │
│    │             │                                       │
│    ▼             ▼                                       │
│  producer(ch)  consumer(ch)   implicit conversion        │
│  can only      can only                                  │
│  send+close    receive+range                             │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Send-only channel parameter
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}

// Receive-only channel parameter
func consumer(ch <-chan int) {
    for val := range ch {
        fmt.Println("Received:", val)
    }
}

func main() {
    ch := make(chan int, 5)

    // Bidirectional channel is implicitly convertible to directional
    go producer(ch) // ch converts to chan<- int
    consumer(ch)    // ch converts to <-chan int
}
```

---

## Closing Channels

**Tutorial: Channel Close Semantics**

`close(ch)` signals that no more values will be sent on the channel. Receives after close drain any buffered values first, then return the zero value with `ok=false`. Watch for two panics: sending to a closed channel, and closing an already-closed channel. Only the sender should close a channel.

```
┌──────────────────────────────────────────────────────────┐
│        Closed Channel Behavior                           │
│                                                          │
│  ch <- 1, 2, 3 → close(ch)                              │
│                                                          │
│  <-ch  → 1, true   ◄── buffered values returned first   │
│  <-ch  → 2, true                                        │
│  <-ch  → 3, true                                        │
│  <-ch  → 0, false  ◄── zero value, ok=false (closed)    │
│  <-ch  → 0, false  ◄── keeps returning zero, false      │
│                                                          │
│  ch <- 4   → PANIC: send on closed channel              │
│  close(ch) → PANIC: close of closed channel             │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 5)

    // Send some values and close
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch) // signals: no more values will be sent

    // Receiving from closed channel returns zero value + false
    val, ok := <-ch
    fmt.Println(val, ok) // 1 true

    val, ok = <-ch
    fmt.Println(val, ok) // 2 true

    val, ok = <-ch
    fmt.Println(val, ok) // 3 true

    val, ok = <-ch
    fmt.Println(val, ok) // 0 false — channel closed and empty

    // Sending to closed channel PANICS!
    // ch <- 4 // panic: send on closed channel

    // Closing already closed channel PANICS!
    // close(ch) // panic: close of closed channel
}
```

---

## Range Over Channel

**Tutorial: Iterating Channels with range**

`for val := range ch` reads values from a channel until it is closed. The sender MUST close the channel when done, or the range loop blocks forever waiting for more values (causing a deadlock). This pattern is ideal for producers that generate a known sequence then signal completion.

```
┌──────────────────────────────────────────────────────────┐
│        range Over Channel                                │
│                                                          │
│  fibonacci goroutine:          main goroutine:           │
│  ┌─────────────────┐          ┌──────────────────┐       │
│  │ ch <- 0         │────────► │ for val := range ch│     │
│  │ ch <- 1         │────────► │   print(val)      │      │
│  │ ch <- 1         │────────► │   ...             │      │
│  │ ...             │          │                   │      │
│  │ ch <- 34        │────────► │   print(34)       │      │
│  │ close(ch) ──────│────X───► │ loop exits ✓      │      │
│  └─────────────────┘          └──────────────────┘       │
│                                                          │
│  Without close(ch) → range blocks forever (deadlock!)   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func fibonacci(n int, ch chan<- int) {
    a, b := 0, 1
    for i := 0; i < n; i++ {
        ch <- a
        a, b = b, a+b
    }
    close(ch) // MUST close, or range will block forever
}

func main() {
    ch := make(chan int)

    go fibonacci(10, ch)

    // range reads until channel is closed
    for val := range ch {
        fmt.Print(val, " ")
    }
    fmt.Println()
    // 0 1 1 2 3 5 8 13 21 34
}
```

---

## Nil Channel Behavior

**Tutorial: Nil Channels and Disabling select Cases**

Operations on a nil channel block forever — both sends and receives. This seems useless but is a powerful tool in `select` statements: set a channel variable to `nil` to disable that case on future iterations. This pattern lets you drain multiple channels exactly once without duplicating logic.

```
┌──────────────────────────────────────────────────────────┐
│        Nil Channel Behavior in select                    │
│                                                          │
│  var ch chan int   (nil)                                  │
│  ch <- 1           → blocks forever                      │
│  <-ch              → blocks forever                      │
│                                                          │
│  Useful pattern — disable select cases:                  │
│  Iteration 1:  ch1="one"  ch2="two"                      │
│    select picks ch1 → print → ch1 = nil (disabled)      │
│                                                          │
│  Iteration 2:  ch1=nil    ch2="two"                      │
│    select skips nil ch1 → picks ch2 → print → ch2 = nil │
│                                                          │
│  Both channels consumed exactly once ✓                   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    var ch chan int // nil channel

    // Sending to nil channel blocks FOREVER
    // ch <- 1 // blocks forever (deadlock if in main goroutine)

    // Receiving from nil channel blocks FOREVER
    // <-ch // blocks forever

    // Useful in select: disable a case by setting channel to nil
    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)

    ch1 <- "one"
    ch2 <- "two"

    for i := 0; i < 2; i++ {
        select {
        case v, ok := <-ch1:
            if ok {
                fmt.Println("ch1:", v)
                ch1 = nil // disable this case for future iterations
            }
        case v, ok := <-ch2:
            if ok {
                fmt.Println("ch2:", v)
                ch2 = nil
            }
        }
    }

    _ = time.Now()
}
```

---

## select Statement — Advanced

**Tutorial: Multiplexing Channel Operations with select**

The `select` statement multiplexes channel operations: it blocks until one case is ready, then executes that case. Key behaviors include first-ready-wins for racing channels, timeouts with `time.After`, non-blocking operation via `default`, and random selection when multiple cases are ready simultaneously.

```
┌──────────────────────────────────────────────────────────┐
│          select Statement Behaviors                      │
│                                                          │
│  1. Multi-way:  select on ch1 and ch2                    │
│     → first ready wins (ch1 at 100ms beats ch2 at 500ms)│
│                                                          │
│  2. Timeout:    select { case <-ch; case <-time.After }  │
│     → if ch not ready in 200ms, timeout case fires      │
│                                                          │
│  3. Non-blocking: select { case <-ch; default: }         │
│     → if ch not ready, default runs immediately          │
│                                                          │
│  4. Random:     ch4 and ch5 both ready                   │
│     ┌────┐  ┌────┐                                       │
│     │ ch4│  │ ch5│  ← both have data                     │
│     └──┬─┘  └──┬─┘                                       │
│        └───┬───┘                                         │
│          select → picks one at RANDOM (fair)             │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "fast"
    }()

    go func() {
        time.Sleep(500 * time.Millisecond)
        ch2 <- "slow"
    }()

    // 1. Multi-way channel operations
    select {
    case msg := <-ch1:
        fmt.Println("Received from ch1:", msg) // This wins (faster)
    case msg := <-ch2:
        fmt.Println("Received from ch2:", msg)
    }

    // 2. Timeout
    select {
    case msg := <-ch2:
        fmt.Println("Got:", msg)
    case <-time.After(200 * time.Millisecond):
        fmt.Println("Timeout!") // Timeout!
    }

    // 3. Non-blocking with default
    ch3 := make(chan int)
    select {
    case v := <-ch3:
        fmt.Println("Got:", v)
    default:
        fmt.Println("No data ready") // No data ready
    }

    // 4. When multiple cases ready — random selection
    ch4 := make(chan int, 1)
    ch5 := make(chan int, 1)
    ch4 <- 1
    ch5 <- 2

    select {
    case v := <-ch4:
        fmt.Println("From ch4:", v)
    case v := <-ch5:
        fmt.Println("From ch5:", v)
    }
    // Random! Sometimes ch4, sometimes ch5
}
```

---

## Channel Patterns

### Done Channel / Cancellation

**Tutorial: Broadcasting Cancellation with a Done Channel**

The "done channel" pattern uses `chan struct{}` for cancellation signaling. Closing the channel broadcasts to ALL listening goroutines simultaneously — every `select` case watching `<-done` unblocks. Using `struct{}` as the element type costs zero memory since it carries no data, only a signal.

```
┌──────────────────────────────────────────────────────────┐
│        Done Channel Cancellation Pattern                 │
│                                                          │
│  main:                     worker:                       │
│  ┌────────────────┐       ┌─────────────────────┐        │
│  │ done := make() │       │ select {            │        │
│  │ go worker(done)│──────►│ case <-done: return │        │
│  │ Sleep(350ms)   │       │ default: do work... │        │
│  │ close(done) ───│──X──► │ }                   │        │
│  └────────────────┘       └─────────────────────┘        │
│                                                          │
│  close(done) → all receivers see it immediately          │
│  struct{} → zero memory cost                             │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func worker(done <-chan struct{}) {
    for {
        select {
        case <-done:
            fmt.Println("Worker: received done signal, stopping")
            return
        default:
            fmt.Println("Worker: doing work...")
            time.Sleep(100 * time.Millisecond)
        }
    }
}

func main() {
    done := make(chan struct{})

    go worker(done)
    time.Sleep(350 * time.Millisecond)

    close(done) // signal all workers to stop
    time.Sleep(50 * time.Millisecond)
    fmt.Println("Main: done")
}
```

### Fan-In (merge multiple channels into one)

**Tutorial: Fan-In — Merging Multiple Channels**

The fan-in pattern merges multiple input channels into a single output channel. A goroutine is spawned per input channel to forward values to the merged channel. A `WaitGroup` tracks when all inputs are exhausted, then closes the merged channel so consumers know to stop.

```
┌──────────────────────────────────────────────────────────┐
│          Fan-In: Merge Multiple Channels                 │
│                                                          │
│  ch1 ──► [1, 2, 3]  ──┐                                 │
│                        ├──► merged ──► [1,10,2,20,3,30]  │
│  ch2 ──► [10,20,30] ──┘       (order may vary)          │
│                                                          │
│  Implementation:                                         │
│  ┌──────────┐    ┌────────────┐    ┌───────────┐         │
│  │ G reads  │───►│            │    │           │         │
│  │ from ch1 │    │  merged ch │───►│  consumer │         │
│  │ G reads  │───►│            │    │           │         │
│  │ from ch2 │    └────────────┘    └───────────┘         │
│  └──────────┘    wg.Wait() → close(merged)              │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

func fanIn(channels ...<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                merged <- val
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
    ch1 := make(chan int, 3)
    ch2 := make(chan int, 3)

    ch1 <- 1; ch1 <- 2; ch1 <- 3; close(ch1)
    ch2 <- 10; ch2 <- 20; ch2 <- 30; close(ch2)

    for val := range fanIn(ch1, ch2) {
        fmt.Print(val, " ")
    }
    fmt.Println() // 1 2 3 10 20 30 (order may vary)
}
```

### Pipeline

**Tutorial: Channel Pipeline — Chaining Processing Stages**

A pipeline chains processing stages connected by channels. Each stage is a goroutine that reads from an input channel, transforms data, and writes to an output channel. Each stage closes its output channel when its input is exhausted, propagating completion down the chain.

```
┌──────────────────────────────────────────────────────────┐
│          Channel Pipeline Pattern                        │
│                                                          │
│  generate(1,2,3,4,5)     square()         filter(>10)   │
│  ┌───────────────┐    ┌───────────┐    ┌─────────────┐   │
│  │ 1→2→3→4→5     │───►│ 1→4→9→   │───►│ 16→25       │   │
│  │               │    │ 16→25    │    │             │   │
│  └───────────────┘    └───────────┘    └──────┬──────┘   │
│  out = chan int        out = chan int       result ch     │
│  close(out) ✓          close(out) ✓       close(out) ✓  │
│                                                          │
│  Each stage: goroutine + input ch + output ch            │
│  Each stage closes its output when input is exhausted    │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func filter(in <-chan int, predicate func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if predicate(n) {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

func main() {
    // Pipeline: generate → square → filter (keep > 10)
    nums := generate(1, 2, 3, 4, 5)
    squared := square(nums)
    result := filter(squared, func(n int) bool { return n > 10 })

    for val := range result {
        fmt.Print(val, " ") // 16 25
    }
    fmt.Println()
}
```

### Semaphore (buffered channel as counting semaphore)

**Tutorial: Limiting Concurrency with a Channel Semaphore**

A buffered channel can act as a counting semaphore to cap the number of concurrent goroutines. Sending to the channel "acquires" a slot (blocking if full), and receiving "releases" it. This is simpler than using a weighted semaphore library and perfectly idiomatic in Go.

```
┌──────────────────────────────────────────────────────────┐
│       Buffered Channel as Counting Semaphore             │
│                                                          │
│  semaphore := make(chan struct{}, 3)   capacity = 3      │
│                                                          │
│  Worker 0: acquire ─► [■ . .]  running                   │
│  Worker 1: acquire ─► [■ ■ .]  running                   │
│  Worker 2: acquire ─► [■ ■ ■]  running                   │
│  Worker 3: acquire ─► BLOCKS   (buffer full!)            │
│            ...                                           │
│  Worker 0: release ─► [. ■ ■]  Worker 3 unblocks ──►run │
│                                                          │
│  Max 3 goroutines execute concurrently                   │
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
    // Buffered channel limits concurrency to 3
    semaphore := make(chan struct{}, 3)
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            semaphore <- struct{}{} // acquire (blocks if 3 already running)
            defer func() { <-semaphore }() // release

            fmt.Printf("Worker %d running\n", id)
            time.Sleep(100 * time.Millisecond)
        }(i)
    }

    wg.Wait()
    fmt.Println("All done!")
}
```

---

## Interview Questions

1. **What is the difference between buffered and unbuffered channels?**
   - Unbuffered (`make(chan T)`): send blocks until a receiver is ready (synchronous). Buffered (`make(chan T, n)`): send blocks only when buffer is full, receive blocks when empty.

2. **What happens when you send to or receive from a closed channel?**
   - Sending to a closed channel panics. Receiving from a closed channel returns the zero value immediately (with `ok=false` in comma-ok form).

3. **What happens with a nil channel?**
   - Both send and receive on a nil channel block forever. This is useful in `select` to disable a case — set the channel to nil to stop selecting on it.

4. **How does `select` work when multiple cases are ready?**
   - Go picks one case at random. This ensures fairness and prevents starvation. If no case is ready and there's a `default`, the default executes (non-blocking).

5. **What is the done channel pattern?**
   - A `chan struct{}` used for signaling cancellation or completion. Close the channel to broadcast to all receivers simultaneously. `struct{}` uses zero memory.

6. **What is fan-in and fan-out?**
   - Fan-out: distribute work from one channel to multiple goroutines. Fan-in: merge results from multiple channels into one channel. Together they form common concurrency pipelines.

7. **How do you prevent goroutine leaks with channels?**
   - Always ensure channels are closed or goroutines have an exit path. Use `context.Context` for cancellation, `select` with a done channel, or timeouts with `time.After`.

8. **What are directional channels?**
   - `chan<- T` is send-only, `<-chan T` is receive-only. They enforce channel usage at compile time. A bidirectional channel can be implicitly converted to a directional one.

9. **When should you use channels vs mutexes?**
   - Channels: for communication between goroutines, signaling, pipelines. Mutexes: for protecting shared state (concurrent read/write to a variable). Go proverb: "Don't communicate by sharing memory; share memory by communicating."

10. **What is the pipeline pattern?**
    - A chain of stages where each stage is a goroutine reading from an input channel and writing to an output channel. Data flows through the pipeline: `generate → transform → filter → output`.
