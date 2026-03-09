# Phase 7: Queue — Priority Queue

## Overview

A **priority queue** serves the element with the highest (or lowest) priority first, regardless of insertion order. It's typically implemented with a **heap**.

| Operation | Time |
|-----------|------|
| Insert | O(log n) |
| ExtractMin/Max | O(log n) |
| Peek | O(1) |

Go provides `container/heap` — you implement the `heap.Interface` and the package handles the rest.

---

## Example 1: Min-Heap Priority Queue (container/heap)

```go
package main

import (
    "container/heap"
    "fmt"
)

type MinHeap []int

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }

func (h *MinHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    val := old[n-1]
    *h = old[:n-1]
    return val
}

func main() {
    h := &MinHeap{}
    heap.Init(h)

    nums := []int{5, 3, 8, 1, 9, 2}
    for _, n := range nums {
        heap.Push(h, n)
    }

    fmt.Print("Min-heap order: ")
    for h.Len() > 0 {
        fmt.Printf("%d ", heap.Pop(h))
    }
    fmt.Println() // 1 2 3 5 8 9
}
```

---

## Example 2: Max-Heap Priority Queue

```go
package main

import (
    "container/heap"
    "fmt"
)

type MaxHeap []int

func (h MaxHeap) Len() int            { return len(h) }
func (h MaxHeap) Less(i, j int) bool  { return h[i] > h[j] } // reversed!
func (h MaxHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() interface{} {
    old := *h
    n := len(old)
    val := old[n-1]
    *h = old[:n-1]
    return val
}

func main() {
    h := &MaxHeap{}
    heap.Init(h)

    for _, n := range []int{5, 3, 8, 1, 9, 2} {
        heap.Push(h, n)
    }

    fmt.Print("Max-heap order: ")
    for h.Len() > 0 {
        fmt.Printf("%d ", heap.Pop(h))
    }
    fmt.Println() // 9 8 5 3 2 1
}
```

---

## Example 3: Priority Queue with Custom Struct

```go
package main

import (
    "container/heap"
    "fmt"
)

type Task struct {
    Name     string
    Priority int
}

type TaskQueue []*Task

func (q TaskQueue) Len() int            { return len(q) }
func (q TaskQueue) Less(i, j int) bool  { return q[i].Priority > q[j].Priority } // higher priority first
func (q TaskQueue) Swap(i, j int)       { q[i], q[j] = q[j], q[i] }
func (q *TaskQueue) Push(x interface{}) { *q = append(*q, x.(*Task)) }
func (q *TaskQueue) Pop() interface{} {
    old := *q
    n := len(old)
    task := old[n-1]
    *q = old[:n-1]
    return task
}

func main() {
    pq := &TaskQueue{}
    heap.Init(pq)

    tasks := []*Task{
        {"Email", 1},
        {"Bug fix", 5},
        {"Meeting", 3},
        {"Deploy", 4},
        {"Docs", 2},
    }

    for _, t := range tasks {
        heap.Push(pq, t)
    }

    fmt.Println("Processing by priority:")
    for pq.Len() > 0 {
        task := heap.Pop(pq).(*Task)
        fmt.Printf("  [P%d] %s\n", task.Priority, task.Name)
    }
}
```

---

## Example 4: Kth Largest Element (LeetCode 215)

```go
package main

import (
    "container/heap"
    "fmt"
)

type MinHeap []int

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    val := old[n-1]
    *h = old[:n-1]
    return val
}

func findKthLargest(nums []int, k int) int {
    h := &MinHeap{}
    heap.Init(h)

    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    return (*h)[0]
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{3, 2, 1, 5, 6, 4}, 2},
        {[]int{3, 2, 3, 1, 2, 4, 5, 5, 6}, 4},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %d\n", t.nums, t.k, findKthLargest(t.nums, t.k))
    }
    // 5, 4
}
```

---

## Example 5: Top K Frequent Elements (LeetCode 347)

```go
package main

import (
    "container/heap"
    "fmt"
)

type Item struct {
    Val  int
    Freq int
}

type MinFreqHeap []*Item

func (h MinFreqHeap) Len() int            { return len(h) }
func (h MinFreqHeap) Less(i, j int) bool  { return h[i].Freq < h[j].Freq }
func (h MinFreqHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinFreqHeap) Push(x interface{}) { *h = append(*h, x.(*Item)) }
func (h *MinFreqHeap) Pop() interface{} {
    old := *h
    n := len(old)
    item := old[n-1]
    *h = old[:n-1]
    return item
}

func topKFrequent(nums []int, k int) []int {
    freq := map[int]int{}
    for _, n := range nums {
        freq[n]++
    }

    h := &MinFreqHeap{}
    heap.Init(h)

    for val, f := range freq {
        heap.Push(h, &Item{Val: val, Freq: f})
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(*Item).Val
    }
    return result
}

func main() {
    fmt.Println(topKFrequent([]int{1, 1, 1, 2, 2, 3}, 2))     // [1, 2]
    fmt.Println(topKFrequent([]int{1}, 1))                       // [1]
    fmt.Println(topKFrequent([]int{4, 1, -1, 2, -1, 2, 3}, 2)) // [-1, 2]
}
```

---

## Example 6: Merge K Sorted Arrays

```go
package main

import (
    "container/heap"
    "fmt"
)

type Element struct {
    Val    int
    ArrIdx int
    ElemIdx int
}

type MinHeap []*Element

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].Val < h[j].Val }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*Element)) }
func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    item := old[n-1]
    *h = old[:n-1]
    return item
}

func mergeKSorted(arrs [][]int) []int {
    h := &MinHeap{}
    heap.Init(h)

    for i, arr := range arrs {
        if len(arr) > 0 {
            heap.Push(h, &Element{Val: arr[0], ArrIdx: i, ElemIdx: 0})
        }
    }

    result := []int{}
    for h.Len() > 0 {
        min := heap.Pop(h).(*Element)
        result = append(result, min.Val)

        if min.ElemIdx+1 < len(arrs[min.ArrIdx]) {
            heap.Push(h, &Element{
                Val:     arrs[min.ArrIdx][min.ElemIdx+1],
                ArrIdx:  min.ArrIdx,
                ElemIdx: min.ElemIdx + 1,
            })
        }
    }

    return result
}

func main() {
    arrs := [][]int{
        {1, 4, 7},
        {2, 5, 8},
        {3, 6, 9},
    }
    fmt.Println(mergeKSorted(arrs)) // [1 2 3 4 5 6 7 8 9]
}
```

---

## Example 7: Task Scheduler (LeetCode 621)

```go
package main

import (
    "container/heap"
    "fmt"
)

type MaxHeap []int

func (h MaxHeap) Len() int            { return len(h) }
func (h MaxHeap) Less(i, j int) bool  { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() interface{} {
    old := *h
    n := len(old)
    val := old[n-1]
    *h = old[:n-1]
    return val
}

func leastInterval(tasks []byte, n int) int {
    freq := [26]int{}
    for _, t := range tasks {
        freq[t-'A']++
    }

    h := &MaxHeap{}
    heap.Init(h)
    for _, f := range freq {
        if f > 0 {
            heap.Push(h, f)
        }
    }

    time := 0
    for h.Len() > 0 {
        cycle := n + 1
        temp := []int{}

        for cycle > 0 && h.Len() > 0 {
            f := heap.Pop(h).(int)
            if f > 1 {
                temp = append(temp, f-1)
            }
            time++
            cycle--
        }

        for _, f := range temp {
            heap.Push(h, f)
        }

        if h.Len() > 0 {
            time += cycle // idle time
        }
    }

    return time
}

func main() {
    tests := []struct {
        tasks []byte
        n     int
    }{
        {[]byte{'A', 'A', 'A', 'B', 'B', 'B'}, 2},
        {[]byte{'A', 'A', 'A', 'B', 'B', 'B'}, 0},
        {[]byte{'A', 'A', 'A', 'A', 'A', 'A', 'B', 'C', 'D', 'E', 'F', 'G'}, 2},
    }

    for _, t := range tests {
        fmt.Printf("tasks=%s, n=%d → %d\n", string(t.tasks), t.n, leastInterval(t.tasks, t.n))
    }
    // 8, 6, 16
}
```

---

## Example 8: Median Finder (LeetCode 295) — Two Heaps

```go
package main

import (
    "container/heap"
    "fmt"
)

type MaxHeap []int
func (h MaxHeap) Len() int            { return len(h) }
func (h MaxHeap) Less(i, j int) bool  { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

type MinHeap []int
func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

type MedianFinder struct {
    lo *MaxHeap // max-heap for lower half
    hi *MinHeap // min-heap for upper half
}

func NewMedianFinder() *MedianFinder {
    lo := &MaxHeap{}
    hi := &MinHeap{}
    heap.Init(lo)
    heap.Init(hi)
    return &MedianFinder{lo: lo, hi: hi}
}

func (mf *MedianFinder) AddNum(num int) {
    heap.Push(mf.lo, num)
    // Balance: move max of lo to hi
    heap.Push(mf.hi, heap.Pop(mf.lo))
    // Ensure lo.Len() >= hi.Len()
    if mf.lo.Len() < mf.hi.Len() {
        heap.Push(mf.lo, heap.Pop(mf.hi))
    }
}

func (mf *MedianFinder) FindMedian() float64 {
    if mf.lo.Len() > mf.hi.Len() {
        return float64((*mf.lo)[0])
    }
    return float64((*mf.lo)[0]+(*mf.hi)[0]) / 2.0
}

func main() {
    mf := NewMedianFinder()
    nums := []int{1, 2, 3, 4, 5}

    for _, n := range nums {
        mf.AddNum(n)
        fmt.Printf("Add %d → median=%.1f\n", n, mf.FindMedian())
    }
    // 1→1.0, 2→1.5, 3→2.0, 4→2.5, 5→3.0
}
```

---

## Example 9: Reorganize String (LeetCode 767)

```go
package main

import (
    "container/heap"
    "fmt"
)

type CharFreq struct {
    Char byte
    Freq int
}

type CharHeap []*CharFreq
func (h CharHeap) Len() int            { return len(h) }
func (h CharHeap) Less(i, j int) bool  { return h[i].Freq > h[j].Freq }
func (h CharHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *CharHeap) Push(x interface{}) { *h = append(*h, x.(*CharFreq)) }
func (h *CharHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

func reorganizeString(s string) string {
    freq := [26]int{}
    for _, ch := range s {
        freq[ch-'a']++
    }

    h := &CharHeap{}
    heap.Init(h)
    for i, f := range freq {
        if f > 0 {
            heap.Push(h, &CharFreq{Char: byte('a' + i), Freq: f})
        }
    }

    result := []byte{}
    for h.Len() >= 2 {
        first := heap.Pop(h).(*CharFreq)
        second := heap.Pop(h).(*CharFreq)

        result = append(result, first.Char, second.Char)
        first.Freq--
        second.Freq--

        if first.Freq > 0 {
            heap.Push(h, first)
        }
        if second.Freq > 0 {
            heap.Push(h, second)
        }
    }

    if h.Len() > 0 {
        last := heap.Pop(h).(*CharFreq)
        if last.Freq > 1 {
            return "" // impossible
        }
        result = append(result, last.Char)
    }

    return string(result)
}

func main() {
    tests := []string{"aab", "aaab", "aaabbc"}
    for _, s := range tests {
        fmt.Printf("%-10s → %q\n", s, reorganizeString(s))
    }
    // "aab" → "aba"
    // "aaab" → "" (impossible)
    // "aaabbc" → "ababac" or similar
}
```

---

## Example 10: K Closest Points to Origin (LeetCode 973)

```go
package main

import (
    "container/heap"
    "fmt"
)

type Point struct {
    X, Y int
    Dist int
}

type MaxDistHeap []*Point
func (h MaxDistHeap) Len() int            { return len(h) }
func (h MaxDistHeap) Less(i, j int) bool  { return h[i].Dist > h[j].Dist } // max-heap
func (h MaxDistHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxDistHeap) Push(x interface{}) { *h = append(*h, x.(*Point)) }
func (h *MaxDistHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

func kClosest(points [][]int, k int) [][]int {
    h := &MaxDistHeap{}
    heap.Init(h)

    for _, p := range points {
        dist := p[0]*p[0] + p[1]*p[1]
        heap.Push(h, &Point{X: p[0], Y: p[1], Dist: dist})
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    result := make([][]int, h.Len())
    for i := h.Len() - 1; i >= 0; i-- {
        pt := heap.Pop(h).(*Point)
        result[i] = []int{pt.X, pt.Y}
    }
    return result
}

func main() {
    points := [][]int{{1, 3}, {-2, 2}, {5, 8}, {0, 1}}
    fmt.Println(kClosest(points, 2)) // [[0,1], [-2,2]]
}
```

---

## Priority Queue Patterns

| Pattern | Heap Type | Example |
|---------|-----------|---------|
| Kth largest | Min-heap of size K | LeetCode 215 |
| Kth smallest | Max-heap of size K | Reverse above |
| Top K frequent | Min-heap by freq | LeetCode 347 |
| Merge K sorted | Min-heap of heads | LeetCode 23 |
| Median stream | Max-heap + min-heap | LeetCode 295 |
| Task scheduling | Max-heap by freq | LeetCode 621 |

## Key Takeaways

1. Go's `container/heap` requires implementing 5 methods: `Len`, `Less`, `Swap`, `Push`, `Pop`
2. **Min-heap**: `Less` returns `h[i] < h[j]`; **Max-heap**: reverse with `>`
3. **Kth largest**: maintain a min-heap of size K — top is the answer
4. **Two heaps** (median): max-heap for lower half, min-heap for upper half
5. Always call `heap.Push`/`heap.Pop` (not the method directly) — these maintain the heap invariant
6. **Time**: Push/Pop are O(log n), Peek (index 0) is O(1)

> **Next up:** Heap-Based Queues →
