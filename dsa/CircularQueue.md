# Phase 7: Queue — Circular Queue

## Overview

A **circular queue** (ring buffer) uses a fixed-size array with `head` and `tail` pointers that wrap around using modulo arithmetic. No memory waste, no shifting.

| Operation | Time | Space |
|-----------|------|-------|
| Enqueue | O(1) | O(n) fixed |
| Dequeue | O(1) | |
| Peek | O(1) | |
| IsFull | O(1) | |

---

## Example 1: Design Circular Queue (LeetCode 622)

```go
package main

import "fmt"

type MyCircularQueue struct {
    data     []int
    head     int
    tail     int
    size     int
    capacity int
}

func Constructor(k int) MyCircularQueue {
    return MyCircularQueue{
        data:     make([]int, k),
        head:     0,
        tail:     0,
        size:     0,
        capacity: k,
    }
}

func (q *MyCircularQueue) EnQueue(value int) bool {
    if q.IsFull() {
        return false
    }
    q.data[q.tail] = value
    q.tail = (q.tail + 1) % q.capacity
    q.size++
    return true
}

func (q *MyCircularQueue) DeQueue() bool {
    if q.IsEmpty() {
        return false
    }
    q.head = (q.head + 1) % q.capacity
    q.size--
    return true
}

func (q *MyCircularQueue) Front() int {
    if q.IsEmpty() {
        return -1
    }
    return q.data[q.head]
}

func (q *MyCircularQueue) Rear() int {
    if q.IsEmpty() {
        return -1
    }
    return q.data[(q.tail-1+q.capacity)%q.capacity]
}

func (q *MyCircularQueue) IsEmpty() bool {
    return q.size == 0
}

func (q *MyCircularQueue) IsFull() bool {
    return q.size == q.capacity
}

func main() {
    q := Constructor(3)
    fmt.Println(q.EnQueue(1)) // true
    fmt.Println(q.EnQueue(2)) // true
    fmt.Println(q.EnQueue(3)) // true
    fmt.Println(q.EnQueue(4)) // false (full)
    fmt.Println("Rear:", q.Rear())   // 3
    fmt.Println(q.IsFull())          // true
    fmt.Println(q.DeQueue())         // true
    fmt.Println(q.EnQueue(4))        // true
    fmt.Println("Rear:", q.Rear())   // 4
}
```

---

## Example 2: Circular Queue with Wasted Slot (No Size Counter)

```go
package main

import "fmt"

// Classic approach: waste one slot to distinguish full from empty
type CircularQueue struct {
    data []int
    head int
    tail int
    cap  int // actual capacity = cap - 1
}

func NewCircularQueue(k int) *CircularQueue {
    return &CircularQueue{
        data: make([]int, k+1), // one extra slot
        cap:  k + 1,
    }
}

func (q *CircularQueue) Enqueue(val int) bool {
    if q.IsFull() {
        return false
    }
    q.data[q.tail] = val
    q.tail = (q.tail + 1) % q.cap
    return true
}

func (q *CircularQueue) Dequeue() (int, bool) {
    if q.IsEmpty() {
        return 0, false
    }
    val := q.data[q.head]
    q.head = (q.head + 1) % q.cap
    return val, true
}

func (q *CircularQueue) IsEmpty() bool {
    return q.head == q.tail
}

func (q *CircularQueue) IsFull() bool {
    return (q.tail+1)%q.cap == q.head
}

func (q *CircularQueue) Size() int {
    return (q.tail - q.head + q.cap) % q.cap
}

func main() {
    q := NewCircularQueue(3)
    q.Enqueue(10)
    q.Enqueue(20)
    q.Enqueue(30)
    fmt.Println("Full:", q.IsFull()) // true

    v, _ := q.Dequeue()
    fmt.Println("Dequeued:", v) // 10
    fmt.Println("Size:", q.Size()) // 2

    q.Enqueue(40) // wraps around
    fmt.Println("Size:", q.Size()) // 3
}
```

---

## Example 3: Dynamic Circular Queue (Auto-Resize)

```go
package main

import "fmt"

type DynamicCircularQueue struct {
    data []int
    head int
    size int
    cap  int
}

func NewDCQ(cap int) *DynamicCircularQueue {
    return &DynamicCircularQueue{
        data: make([]int, cap),
        cap:  cap,
    }
}

func (q *DynamicCircularQueue) Enqueue(val int) {
    if q.size == q.cap {
        q.resize(q.cap * 2)
    }
    idx := (q.head + q.size) % q.cap
    q.data[idx] = val
    q.size++
}

func (q *DynamicCircularQueue) Dequeue() (int, bool) {
    if q.size == 0 {
        return 0, false
    }
    val := q.data[q.head]
    q.head = (q.head + 1) % q.cap
    q.size--

    // Shrink if quarter full
    if q.size > 0 && q.size == q.cap/4 {
        q.resize(q.cap / 2)
    }
    return val, true
}

func (q *DynamicCircularQueue) resize(newCap int) {
    newData := make([]int, newCap)
    for i := 0; i < q.size; i++ {
        newData[i] = q.data[(q.head+i)%q.cap]
    }
    q.data = newData
    q.head = 0
    q.cap = newCap
}

func main() {
    q := NewDCQ(2)
    for i := 1; i <= 10; i++ {
        q.Enqueue(i)
        fmt.Printf("Enqueue %d → cap=%d, size=%d\n", i, q.cap, q.size)
    }

    fmt.Println("\nDequeuing:")
    for q.size > 0 {
        v, _ := q.Dequeue()
        fmt.Printf("Dequeue %d → cap=%d, size=%d\n", v, q.cap, q.size)
    }
}
```

---

## Example 4: Circular Buffer for Producer-Consumer

```go
package main

import (
    "fmt"
    "sync"
)

type CircularBuffer struct {
    data []int
    head int
    tail int
    size int
    cap  int
    mu   sync.Mutex
    notFull  *sync.Cond
    notEmpty *sync.Cond
}

func NewBuffer(cap int) *CircularBuffer {
    b := &CircularBuffer{
        data: make([]int, cap),
        cap:  cap,
    }
    b.notFull = sync.NewCond(&b.mu)
    b.notEmpty = sync.NewCond(&b.mu)
    return b
}

func (b *CircularBuffer) Produce(val int) {
    b.mu.Lock()
    defer b.mu.Unlock()

    for b.size == b.cap {
        b.notFull.Wait()
    }

    b.data[b.tail] = val
    b.tail = (b.tail + 1) % b.cap
    b.size++
    b.notEmpty.Signal()
}

func (b *CircularBuffer) Consume() int {
    b.mu.Lock()
    defer b.mu.Unlock()

    for b.size == 0 {
        b.notEmpty.Wait()
    }

    val := b.data[b.head]
    b.head = (b.head + 1) % b.cap
    b.size--
    b.notFull.Signal()
    return val
}

func main() {
    buf := NewBuffer(3)
    var wg sync.WaitGroup

    // Producer
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 1; i <= 6; i++ {
            buf.Produce(i)
            fmt.Printf("Produced: %d\n", i)
        }
    }()

    // Consumer
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 6; i++ {
            val := buf.Consume()
            fmt.Printf("Consumed: %d\n", val)
        }
    }()

    wg.Wait()
}
```

---

## Example 5: Go's container/ring Package

```go
package main

import (
    "container/ring"
    "fmt"
)

func main() {
    // Create ring of size 5
    r := ring.New(5)

    // Fill with values
    for i := 0; i < r.Len(); i++ {
        r.Value = i + 1
        r = r.Next()
    }

    // Print all elements
    fmt.Print("Ring: ")
    r.Do(func(val interface{}) {
        fmt.Printf("%d ", val.(int))
    })
    fmt.Println()

    // Move forward
    r = r.Move(2)
    fmt.Println("After Move(2), current:", r.Value)

    // Remove 2 elements after current
    removed := r.Unlink(2)
    fmt.Print("Removed: ")
    removed.Do(func(val interface{}) {
        fmt.Printf("%d ", val.(int))
    })
    fmt.Println()

    fmt.Print("Remaining: ")
    r.Do(func(val interface{}) {
        fmt.Printf("%d ", val.(int))
    })
    fmt.Println()
}
```

---

## Example 6: Josephus Problem Using Circular Queue

```go
package main

import "fmt"

func josephus(n, k int) int {
    // Simulate with circular queue
    queue := make([]int, n)
    for i := range queue {
        queue[i] = i + 1
    }

    idx := 0
    fmt.Printf("Elimination order: ")

    for len(queue) > 1 {
        idx = (idx + k - 1) % len(queue)
        fmt.Printf("%d ", queue[idx])
        queue = append(queue[:idx], queue[idx+1:]...)
        if idx >= len(queue) {
            idx = 0
        }
    }

    fmt.Println()
    return queue[0]
}

func main() {
    for _, test := range []struct{ n, k int }{
        {7, 3},
        {5, 2},
        {10, 3},
    } {
        survivor := josephus(test.n, test.k)
        fmt.Printf("n=%d, k=%d → survivor=%d\n\n", test.n, test.k, survivor)
    }
}
```

---

## Example 7: Circular Queue with Timestamps (Rate Limiter)

```go
package main

import (
    "fmt"
    "time"
)

type RateLimiter struct {
    timestamps []time.Time
    head       int
    size       int
    limit      int
    window     time.Duration
}

func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
    return &RateLimiter{
        timestamps: make([]time.Time, limit),
        limit:      limit,
        window:     window,
    }
}

func (rl *RateLimiter) Allow(now time.Time) bool {
    // Remove expired timestamps
    for rl.size > 0 && now.Sub(rl.timestamps[rl.head]) > rl.window {
        rl.head = (rl.head + 1) % rl.limit
        rl.size--
    }

    if rl.size >= rl.limit {
        return false
    }

    // Add new timestamp
    idx := (rl.head + rl.size) % rl.limit
    rl.timestamps[idx] = now
    rl.size++
    return true
}

func main() {
    rl := NewRateLimiter(3, 10*time.Second)
    base := time.Now()

    requests := []int{0, 1, 2, 3, 5, 11, 12}
    for _, sec := range requests {
        t := base.Add(time.Duration(sec) * time.Second)
        allowed := rl.Allow(t)
        fmt.Printf("t=%ds: allowed=%v (window count=%d)\n", sec, allowed, rl.size)
    }
}
```

---

## Example 8: Circular Queue Visualization

```go
package main

import (
    "fmt"
    "strings"
)

type VisualQueue struct {
    data []string
    head int
    tail int
    size int
    cap  int
}

func NewVisualQueue(cap int) *VisualQueue {
    data := make([]string, cap)
    for i := range data {
        data[i] = "_"
    }
    return &VisualQueue{data: data, cap: cap}
}

func (q *VisualQueue) Enqueue(val string) bool {
    if q.size == q.cap {
        return false
    }
    q.data[q.tail] = val
    q.tail = (q.tail + 1) % q.cap
    q.size++
    return true
}

func (q *VisualQueue) Dequeue() (string, bool) {
    if q.size == 0 {
        return "", false
    }
    val := q.data[q.head]
    q.data[q.head] = "_"
    q.head = (q.head + 1) % q.cap
    q.size--
    return val, true
}

func (q *VisualQueue) String() string {
    parts := make([]string, q.cap)
    for i := 0; i < q.cap; i++ {
        marker := " "
        if i == q.head && q.size > 0 {
            marker = "H"
        }
        if i == q.tail {
            marker += "T"
        }
        parts[i] = fmt.Sprintf("[%s%s]", q.data[i], marker)
    }
    return strings.Join(parts, " ")
}

func main() {
    q := NewVisualQueue(5)
    fmt.Println("Empty:", q)

    ops := []string{"A", "B", "C"}
    for _, v := range ops {
        q.Enqueue(v)
        fmt.Printf("Enqueue(%s): %s\n", v, q)
    }

    q.Dequeue()
    fmt.Printf("Dequeue:    %s\n", q)

    q.Dequeue()
    fmt.Printf("Dequeue:    %s\n", q)

    q.Enqueue("D")
    fmt.Printf("Enqueue(D): %s\n", q)

    q.Enqueue("E")
    fmt.Printf("Enqueue(E): %s\n", q)

    q.Enqueue("F") // wraps!
    fmt.Printf("Enqueue(F): %s\n", q)
}
```

---

## Example 9: Hot Potato Game (Queue Rotation)

```go
package main

import "fmt"

func hotPotato(names []string, passes int) string {
    queue := make([]string, len(names))
    copy(queue, names)

    for len(queue) > 1 {
        // Pass potato (rotate queue)
        for i := 0; i < passes; i++ {
            // Move front to back
            queue = append(queue, queue[0])
            queue = queue[1:]
        }
        // Eliminate person at front
        eliminated := queue[0]
        queue = queue[1:]
        fmt.Printf("Eliminated: %s\n", eliminated)
    }

    return queue[0]
}

func main() {
    names := []string{"Alice", "Bob", "Charlie", "Diana", "Eve"}
    winner := hotPotato(names, 3)
    fmt.Printf("\nWinner: %s\n", winner)
}
```

---

## Example 10: Circular Log Buffer

```go
package main

import (
    "fmt"
    "time"
)

type LogEntry struct {
    Timestamp time.Time
    Message   string
}

type LogBuffer struct {
    entries []LogEntry
    head    int
    count   int
    cap     int
}

func NewLogBuffer(cap int) *LogBuffer {
    return &LogBuffer{
        entries: make([]LogEntry, cap),
        cap:     cap,
    }
}

func (lb *LogBuffer) Write(msg string) {
    idx := (lb.head + lb.count) % lb.cap
    if lb.count == lb.cap {
        // Overwrite oldest
        lb.head = (lb.head + 1) % lb.cap
    } else {
        lb.count++
    }
    lb.entries[idx] = LogEntry{
        Timestamp: time.Now(),
        Message:   msg,
    }
}

func (lb *LogBuffer) GetAll() []LogEntry {
    result := make([]LogEntry, lb.count)
    for i := 0; i < lb.count; i++ {
        result[i] = lb.entries[(lb.head+i)%lb.cap]
    }
    return result
}

func main() {
    lb := NewLogBuffer(3) // only keeps last 3 logs

    messages := []string{"start", "process", "checkpoint", "validate", "complete"}
    for _, msg := range messages {
        lb.Write(msg)
        entries := lb.GetAll()
        fmt.Printf("After '%s': [", msg)
        for j, e := range entries {
            if j > 0 {
                fmt.Print(", ")
            }
            fmt.Print(e.Message)
        }
        fmt.Println("]")
    }
    // After overwriting: [checkpoint, validate, complete]
}
```

---

## Circular Queue Variants

| Variant | Full Condition | Empty Condition | Extra |
|---------|---------------|----------------|-------|
| Size counter | size == cap | size == 0 | Simple |
| Wasted slot | (tail+1)%cap == head | head == tail | No counter |
| Boolean flag | head == tail && full | head == tail && !full | Extra bool |

## Key Takeaways

1. **Mod arithmetic** `(index + 1) % capacity` makes the array circular
2. **Head** tracks front, **tail** tracks next write position
3. **Full vs empty** ambiguity: use a size counter, waste a slot, or add a flag
4. **Ring buffers** are ideal for fixed-size caches, logs, and producer-consumer
5. **Dynamic resizing**: copy elements in logical order when growing
6. Go's `container/ring` provides a doubly-linked circular list

> **Next up:** Deque →
