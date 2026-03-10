# Phase 7: Queue — Queue Operations

## Overview

A **queue** is a First-In-First-Out (FIFO) data structure supporting:
- **Enqueue** — add to back O(1)
- **Dequeue** — remove from front O(1)
- **Front/Peek** — view front element O(1)
- **IsEmpty** — check if empty O(1)

Go has no built-in queue. Common implementations: **slice** (simple but O(n) dequeue), **linked list**, or **ring buffer**.

---

## Example 1: Queue with Slice (Simple)

```go
package main

import "fmt"

type Queue struct {
    data []int
}

func (q *Queue) Enqueue(val int) {
    q.data = append(q.data, val)
}

func (q *Queue) Dequeue() (int, bool) {
    if len(q.data) == 0 {
        return 0, false
    }
    val := q.data[0]
    q.data = q.data[1:] // O(n) — not ideal for large queues
    return val, true
}

func (q *Queue) Front() (int, bool) {
    if len(q.data) == 0 {
        return 0, false
    }
    return q.data[0], true
}

func (q *Queue) IsEmpty() bool {
    return len(q.data) == 0
}

func (q *Queue) Size() int {
    return len(q.data)
}

func main() {
    q := &Queue{}
    q.Enqueue(10)
    q.Enqueue(20)
    q.Enqueue(30)

    fmt.Println("Size:", q.Size()) // 3

    v, _ := q.Front()
    fmt.Println("Front:", v) // 10

    v, _ = q.Dequeue()
    fmt.Println("Dequeue:", v) // 10

    v, _ = q.Dequeue()
    fmt.Println("Dequeue:", v) // 20

    fmt.Println("Empty:", q.IsEmpty()) // false
}
```

**Textual Figure:**

```
Step 1: Enqueue(10)           Step 2: Enqueue(20)
  data: ┌────┐                 data: ┌────┬────┐
        │ 10 │                       │ 10 │ 20 │
        └────┘                       └────┴────┘
         ↑front/rear                  ↑front    ↑rear

Step 3: Enqueue(30)
  data: ┌────┬────┬────┐
        │ 10 │ 20 │ 30 │       Size → 3,  Front → 10
        └────┴────┴────┘
         ↑front         ↑rear

Step 4: Dequeue() → 10         Step 5: Dequeue() → 20
  data: ┌────┬────┐             data: ┌────┐
        │ 20 │ 30 │  (shifted)        │ 30 │  (shifted)
        └────┴────┘                   └────┘
         ↑front ↑rear                  ↑front/rear

  IsEmpty → false  (1 element remains)
```

---

## Example 2: Queue with Linked List (O(1) Operations)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type LinkedQueue struct {
    head *Node
    tail *Node
    size int
}

func (q *LinkedQueue) Enqueue(val int) {
    node := &Node{Val: val}
    if q.tail != nil {
        q.tail.Next = node
    }
    q.tail = node
    if q.head == nil {
        q.head = node
    }
    q.size++
}

func (q *LinkedQueue) Dequeue() (int, bool) {
    if q.head == nil {
        return 0, false
    }
    val := q.head.Val
    q.head = q.head.Next
    if q.head == nil {
        q.tail = nil
    }
    q.size--
    return val, true
}

func (q *LinkedQueue) Front() (int, bool) {
    if q.head == nil {
        return 0, false
    }
    return q.head.Val, true
}

func (q *LinkedQueue) IsEmpty() bool {
    return q.head == nil
}

func main() {
    q := &LinkedQueue{}
    for i := 1; i <= 5; i++ {
        q.Enqueue(i * 10)
    }

    for !q.IsEmpty() {
        v, _ := q.Dequeue()
        fmt.Printf("%d ", v)
    }
    fmt.Println() // 10 20 30 40 50
}
```

**Textual Figure:**

```
Enqueue 10, 20, 30, 40, 50:

  head                                              tail
   │                                                  │
   ▼                                                  ▼
  ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
  │ 10 │───▶│ 20 │───▶│ 30 │───▶│ 40 │───▶│ 50 │──▶ nil
  └────┘    └────┘    └────┘    └────┘    └────┘

Dequeue → 10 (head moves right):
  head                                tail
   │                                    │
   ▼                                    ▼
  ┌────┐    ┌────┐    ┌────┐    ┌────┐
  │ 20 │───▶│ 30 │───▶│ 40 │───▶│ 50 │──▶ nil
  └────┘    └────┘    └────┘    └────┘

Dequeue → 20, 30, 40, 50 (each shifts head right)

  Final: head = nil, tail = nil  →  empty queue
  Output: 10 20 30 40 50  (FIFO order)
```

---

## Example 3: Generic Queue (Go 1.18+)

```go
package main

import "fmt"

type Queue[T any] struct {
    data []T
    head int
}

func (q *Queue[T]) Enqueue(val T) {
    q.data = append(q.data, val)
}

func (q *Queue[T]) Dequeue() (T, bool) {
    var zero T
    if q.head >= len(q.data) {
        return zero, false
    }
    val := q.data[q.head]
    q.head++
    // Reclaim memory periodically
    if q.head > len(q.data)/2 {
        q.data = q.data[q.head:]
        q.head = 0
    }
    return val, true
}

func (q *Queue[T]) IsEmpty() bool {
    return q.head >= len(q.data)
}

func (q *Queue[T]) Size() int {
    return len(q.data) - q.head
}

func main() {
    q := &Queue[string]{}
    q.Enqueue("alpha")
    q.Enqueue("beta")
    q.Enqueue("gamma")

    for !q.IsEmpty() {
        v, _ := q.Dequeue()
        fmt.Println(v)
    }
}
```

**Textual Figure:**

```
Enqueue "alpha", "beta", "gamma":
  data: ┌───────┬──────┬───────┐
        │ alpha │ beta │ gamma │    head=0
        └───────┴──────┴───────┘
          ↑head

Dequeue → "alpha"  (head=1):
  data: ┌───────┬──────┬───────┐
        │(alpha)│ beta │ gamma │    head=1
        └───────┴──────┴───────┘
                  ↑head

Dequeue → "beta"   (head=2, head > len/2 → compact!):
  data: ┌───────┐
        │ gamma │    head=0  (slice compacted)
        └───────┘
          ↑head

Dequeue → "gamma"  (head=1, head >= len → empty):
  data: ┌┐
        ││    head=0, len=0  →  queue empty
        └┘

  Output: alpha, beta, gamma
```

---

## Example 4: Implement Stack Using Queues (LeetCode 225)

```go
package main

import "fmt"

type MyStack struct {
    q []int
}

func (s *MyStack) Push(x int) {
    s.q = append(s.q, x)
    // Rotate so new element is at front
    for i := 0; i < len(s.q)-1; i++ {
        s.q = append(s.q, s.q[0])
        s.q = s.q[1:]
    }
}

func (s *MyStack) Pop() int {
    val := s.q[0]
    s.q = s.q[1:]
    return val
}

func (s *MyStack) Top() int {
    return s.q[0]
}

func (s *MyStack) Empty() bool {
    return len(s.q) == 0
}

func main() {
    s := &MyStack{}
    s.Push(1)
    s.Push(2)
    s.Push(3)

    fmt.Println("Top:", s.Top())  // 3
    fmt.Println("Pop:", s.Pop())  // 3
    fmt.Println("Pop:", s.Pop())  // 2
    fmt.Println("Empty:", s.Empty()) // false
}
```

**Textual Figure:**

```
Push(1):  q = [1]              (no rotation — single element)

Push(2):  q = [1, 2]           append 2
          rotate 1×:  move front→back
          q = [2, 1]           ← 2 is at front now

Push(3):  q = [2, 1, 3]        append 3
          rotate 2×:  [1, 3, 2] → [3, 2, 1]
          q = [3, 2, 1]        ← 3 is at front now

  Queue simulates stack:
  ┌───┬───┬───┐
  │ 3 │ 2 │ 1 │     front = stack top
  └───┴───┴───┘
   ↑front (top)

  Top()  → 3     q = [3, 2, 1]
  Pop()  → 3     q = [2, 1]
  Pop()  → 2     q = [1]
  Empty  → false (1 element remains)
```

---

## Example 5: Number of Recent Calls (LeetCode 933)

```go
package main

import "fmt"

type RecentCounter struct {
    q []int
}

func (rc *RecentCounter) Ping(t int) int {
    rc.q = append(rc.q, t)
    for rc.q[0] < t-3000 {
        rc.q = rc.q[1:]
    }
    return len(rc.q)
}

func main() {
    rc := &RecentCounter{}
    pings := []int{1, 100, 3001, 3002}

    for _, t := range pings {
        fmt.Printf("Ping(%d) → %d recent calls\n", t, rc.Ping(t))
    }
    // Ping(1)    → 1
    // Ping(100)  → 2
    // Ping(3001) → 3
    // Ping(3002) → 3
}
```

**Textual Figure:**

```
Ping(1):     q: ┌───┐                  window: [1-3000, 1]
                │ 1 │                  1 in range ✓
                └───┘                  → 1 recent call

Ping(100):   q: ┌───┬─────┐            window: [100-3000, 100]
                │ 1 │ 100 │            all in range ✓
                └───┴─────┘            → 2 recent calls

Ping(3001):  q: ┌───┬─────┬──────┐     window: [1, 3001]
                │ 1 │ 100 │ 3001 │     1 ≥ 1 ✓
                └───┴─────┴──────┘     → 3 recent calls

Ping(3002):     1 < 3002-3000=2  ✗  remove!
             q: ┌─────┬──────┬──────┐  window: [2, 3002]
                │ 100 │ 3001 │ 3002 │  100 ≥ 2 ✓
                └─────┴──────┴──────┘  → 3 recent calls
```

---

## Example 6: Moving Average (LeetCode 346)

```go
package main

import "fmt"

type MovingAverage struct {
    queue []int
    size  int
    sum   int
}

func NewMovingAverage(size int) *MovingAverage {
    return &MovingAverage{size: size}
}

func (ma *MovingAverage) Next(val int) float64 {
    ma.queue = append(ma.queue, val)
    ma.sum += val

    if len(ma.queue) > ma.size {
        ma.sum -= ma.queue[0]
        ma.queue = ma.queue[1:]
    }

    return float64(ma.sum) / float64(len(ma.queue))
}

func main() {
    ma := NewMovingAverage(3)
    vals := []int{1, 10, 3, 5}

    for _, v := range vals {
        avg := ma.Next(v)
        fmt.Printf("Add %d → avg=%.2f (window=%v)\n", v, avg, ma.queue)
    }
    // avg=1.00, avg=5.50, avg=4.67, avg=6.00
}
```

**Textual Figure:**

```
Moving Average (window size = 3):

Add 1:   queue: ┌───┐      sum=1   avg = 1/1 = 1.00
                │ 1 │
                └───┘

Add 10:  queue: ┌───┬────┐   sum=11  avg = 11/2 = 5.50
                │ 1 │ 10 │
                └───┴────┘

Add 3:   queue: ┌───┬────┬───┐  sum=14  avg = 14/3 = 4.67
                │ 1 │ 10 │ 3 │  (window full)
                └───┴────┴───┘

Add 5:   evict 1, queue: ┌────┬───┬───┐  sum=18  avg = 18/3 = 6.00
                         │ 10 │ 3 │ 5 │
                         └────┴───┴───┘
```

---

## Example 7: Design Hit Counter (LeetCode 362)

```go
package main

import "fmt"

type HitCounter struct {
    hits []int
}

func (hc *HitCounter) Hit(timestamp int) {
    hc.hits = append(hc.hits, timestamp)
}

func (hc *HitCounter) GetHits(timestamp int) int {
    // Remove hits older than 300 seconds
    cutoff := timestamp - 300
    for len(hc.hits) > 0 && hc.hits[0] <= cutoff {
        hc.hits = hc.hits[1:]
    }
    return len(hc.hits)
}

func main() {
    hc := &HitCounter{}

    hc.Hit(1)
    hc.Hit(2)
    hc.Hit(3)
    fmt.Println("Hits at t=4:", hc.GetHits(4))   // 3

    hc.Hit(300)
    fmt.Println("Hits at t=300:", hc.GetHits(300)) // 4

    fmt.Println("Hits at t=301:", hc.GetHits(301)) // 3 (hit at t=1 expired)
}
```

**Textual Figure:**

```
Timeline of hits and queries:

  t=1   t=2   t=3                  t=300  t=301
  │     │     │                    │      │
  Hit   Hit   Hit                  Hit    Query

GetHits(4):   window = (4-300, 4] = all hits visible
  hits: ┌───┬───┬───┐
        │ 1 │ 2 │ 3 │  → 3 hits
        └───┴───┴───┘

Hit(300):
  hits: ┌───┬───┬───┬─────┐
        │ 1 │ 2 │ 3 │ 300 │
        └───┴───┴───┴─────┘

GetHits(300): cutoff = 0  → all 4 in range  → 4 hits

GetHits(301): cutoff = 1  → hit at t=1 expired!
  hits: ┌───┬───┬─────┐
        │ 2 │ 3 │ 300 │  → 3 hits
        └───┴───┴─────┘
```

---

## Example 8: Queue Using Ring Buffer (Amortized O(1))

```go
package main

import (
    "errors"
    "fmt"
)

type RingQueue struct {
    data     []int
    head     int
    tail     int
    size     int
    capacity int
}

func NewRingQueue(cap int) *RingQueue {
    return &RingQueue{
        data:     make([]int, cap),
        capacity: cap,
    }
}

func (q *RingQueue) Enqueue(val int) error {
    if q.size == q.capacity {
        return errors.New("queue full")
    }
    q.data[q.tail] = val
    q.tail = (q.tail + 1) % q.capacity
    q.size++
    return nil
}

func (q *RingQueue) Dequeue() (int, error) {
    if q.size == 0 {
        return 0, errors.New("queue empty")
    }
    val := q.data[q.head]
    q.head = (q.head + 1) % q.capacity
    q.size--
    return val, nil
}

func (q *RingQueue) Front() (int, error) {
    if q.size == 0 {
        return 0, errors.New("queue empty")
    }
    return q.data[q.head], nil
}

func main() {
    q := NewRingQueue(5)

    for i := 1; i <= 5; i++ {
        q.Enqueue(i * 10)
    }

    err := q.Enqueue(60)
    fmt.Println("Full:", err) // queue full

    v, _ := q.Dequeue()
    fmt.Println("Dequeue:", v) // 10

    q.Enqueue(60) // now has space
    v, _ = q.Front()
    fmt.Println("Front:", v) // 20
}
```

**Textual Figure:**

```
Ring Buffer (capacity=5):

Enqueue 10,20,30,40,50:
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 20 │ 30 │ 40 │ 50 │   size=5 (FULL)
  └────┴────┴────┴────┴────┘
   ↑head                  ↑tail(wraps to 0)

Enqueue(60) → ERROR "queue full"  (size == capacity)

Dequeue() → 10:
  ┌────┬────┬────┬────┬────┐
  │    │ 20 │ 30 │ 40 │ 50 │   size=4, head=1
  └────┴────┴────┴────┴────┘
   ↑tail  ↑head

Enqueue(60):  tail wraps around to index 0
  ┌────┬────┬────┬────┬────┐
  │ 60 │ 20 │ 30 │ 40 │ 50 │   size=5, head=1, tail=1
  └────┴────┴────┴────┴────┘
          ↑head/tail

  Front → 20
```

---

## Example 9: Multi-Queue Task Processor

```go
package main

import "fmt"

type Task struct {
    Name     string
    Priority string // "high", "medium", "low"
}

type MultiQueue struct {
    high   []Task
    medium []Task
    low    []Task
}

func (mq *MultiQueue) Enqueue(task Task) {
    switch task.Priority {
    case "high":
        mq.high = append(mq.high, task)
    case "medium":
        mq.medium = append(mq.medium, task)
    default:
        mq.low = append(mq.low, task)
    }
}

func (mq *MultiQueue) Dequeue() (Task, bool) {
    if len(mq.high) > 0 {
        t := mq.high[0]
        mq.high = mq.high[1:]
        return t, true
    }
    if len(mq.medium) > 0 {
        t := mq.medium[0]
        mq.medium = mq.medium[1:]
        return t, true
    }
    if len(mq.low) > 0 {
        t := mq.low[0]
        mq.low = mq.low[1:]
        return t, true
    }
    return Task{}, false
}

func main() {
    mq := &MultiQueue{}
    mq.Enqueue(Task{"Email", "low"})
    mq.Enqueue(Task{"Bug fix", "high"})
    mq.Enqueue(Task{"Meeting", "medium"})
    mq.Enqueue(Task{"Deploy", "high"})
    mq.Enqueue(Task{"Docs", "low"})

    fmt.Println("Processing order:")
    for {
        t, ok := mq.Dequeue()
        if !ok {
            break
        }
        fmt.Printf("  [%s] %s\n", t.Priority, t.Name)
    }
}
```

**Textual Figure:**

```
Three-level priority queue:

  high:   ┌─────────┬────────┐
          │ Bug fix │ Deploy │    ← processed first
          └─────────┴────────┘
  medium: ┌─────────┐
          │ Meeting │              ← processed second
          └─────────┘
  low:    ┌───────┬──────┐
          │ Email │ Docs │        ← processed last
          └───────┴──────┘

Dequeue order:
  1. [high]   Bug fix
  2. [high]   Deploy
  3. [medium] Meeting
  4. [low]    Email
  5. [low]    Docs
```

---

## Example 10: Interleave First and Second Half of Queue

```go
package main

import "fmt"

// Given [1,2,3,4,5,6,7,8,9,10] → [1,6,2,7,3,8,4,9,5,10]
func interleave(q []int) []int {
    n := len(q)
    if n%2 != 0 {
        return q // must be even
    }

    half := n / 2
    firstHalf := make([]int, half)
    copy(firstHalf, q[:half])

    result := make([]int, 0, n)
    for i := 0; i < half; i++ {
        result = append(result, firstHalf[i])
        result = append(result, q[half+i])
    }

    return result
}

func main() {
    q := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    fmt.Println("Before:", q)
    fmt.Println("After: ", interleave(q))

    q2 := []int{11, 12, 13, 14, 15, 16}
    fmt.Println("Before:", q2)
    fmt.Println("After: ", interleave(q2))
}
```

**Textual Figure:**

```
Input:  [1, 2, 3, 4, 5,  6, 7, 8, 9, 10]
         └── 1st half ──┘  └── 2nd half ──┘

Split:
  firstHalf: ┌───┬───┬───┬───┬───┐
             │ 1 │ 2 │ 3 │ 4 │ 5 │
             └───┴───┴───┴───┴───┘
  secondHalf: ┌───┬───┬───┬───┬────┐
              │ 6 │ 7 │ 8 │ 9 │ 10 │
              └───┴───┴───┴───┴────┘

Interleave (take one from each alternately):
  1 → 6 → 2 → 7 → 3 → 8 → 4 → 9 → 5 → 10

Result: ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬────┐
        │ 1 │ 6 │ 2 │ 7 │ 3 │ 8 │ 4 │ 9 │ 5 │ 10 │
        └───┴───┴───┴───┴───┴───┴───┴───┴───┴────┘
```

---

## Queue Implementation Comparison

| Method | Enqueue | Dequeue | Space | Notes |
|--------|---------|---------|-------|-------|
| Slice (simple) | O(1) amortized | O(n) | Dynamic | Head removal shifts all elements |
| Slice (head idx) | O(1) amortized | O(1) | Wastes prefix | Periodic compaction needed |
| Linked list | O(1) | O(1) | More allocs | Best for pure O(1) guarantee |
| Ring buffer | O(1) | O(1) | Fixed size | Best space locality |
| Two stacks | O(1) | O(1) amortized | 2× space | Good for specific patterns |

## Key Takeaways

1. Go uses **slices** as queues, but dequeue is O(n) — use head index or linked list
2. **Ring buffer** is ideal for fixed-capacity queues — mod arithmetic wraps around
3. **Stack ↔ Queue** conversion: two stacks simulate a queue and vice versa
4. **Sliding window** problems often use queues to maintain window state
5. **Multi-level queues** handle priority without a full heap
6. Go's `container/list` can serve as a queue but typed with `interface{}`

> **Next up:** Circular Queue →
