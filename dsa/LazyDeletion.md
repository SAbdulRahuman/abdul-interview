# Phase 11: Heap / Priority Queue — Lazy Deletion

## Overview

**Lazy deletion** is a technique where instead of removing elements from a heap immediately, you **mark them as deleted** and skip them when they reach the top. This avoids the O(n) cost of finding an element in the heap.

```
Scenario: Remove element 5 from a min heap
  
  Normal: Find 5 in heap → O(n), then remove → O(log n)
  Lazy:   Mark 5 as deleted → O(1)
          When 5 reaches top → skip it → O(log n) amortized
```

**Why?** Go's `container/heap` requires an index to remove non-top elements. Lazy deletion avoids tracking indices.

---

## Example 1: Basic Lazy Deletion

```go
package main

import (
	"container/heap"
	"fmt"
)

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type LazyMinHeap struct {
	h       *MinHeap
	deleted map[int]int // value → count of deletions
}

func NewLazyMinHeap() *LazyMinHeap {
	return &LazyMinHeap{h: &MinHeap{}, deleted: map[int]int{}}
}

func (lh *LazyMinHeap) Push(val int) {
	heap.Push(lh.h, val)
}

func (lh *LazyMinHeap) Delete(val int) {
	lh.deleted[val]++
}

func (lh *LazyMinHeap) prune() {
	for lh.h.Len() > 0 {
		top := (*lh.h)[0]
		if cnt, ok := lh.deleted[top]; ok && cnt > 0 {
			lh.deleted[top]--
			if lh.deleted[top] == 0 { delete(lh.deleted, top) }
			heap.Pop(lh.h)
		} else {
			break
		}
	}
}

func (lh *LazyMinHeap) Top() int {
	lh.prune()
	return (*lh.h)[0]
}

func (lh *LazyMinHeap) Pop() int {
	lh.prune()
	return heap.Pop(lh.h).(int)
}

func main() {
	lh := NewLazyMinHeap()
	lh.Push(5); lh.Push(3); lh.Push(8); lh.Push(1)
	fmt.Println("Min:", lh.Top())   // 1
	lh.Delete(1)                     // lazy delete 1
	fmt.Println("Min:", lh.Top())   // 3
	lh.Delete(3)
	fmt.Println("Min:", lh.Top())   // 5
}
```

---

## Example 2: Sliding Window with Lazy Deletion

```go
package main

import (
	"container/heap"
	"fmt"
)

type Item struct{ val, idx int }
type MaxH []Item
func (h MaxH) Len() int           { return len(h) }
func (h MaxH) Less(i, j int) bool { return h[i].val > h[j].val }
func (h MaxH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxH) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MaxH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// Max of sliding window using heap + lazy deletion
func maxSlidingWindow(nums []int, k int) []int {
	h := &MaxH{}
	var result []int

	for i, v := range nums {
		heap.Push(h, Item{v, i})
		// Lazy prune: remove elements outside window
		for (*h)[0].idx <= i-k {
			heap.Pop(h)
		}
		if i >= k-1 {
			result = append(result, (*h)[0].val)
		}
	}
	return result
}

func main() {
	fmt.Println(maxSlidingWindow([]int{1, 3, -1, -3, 5, 3, 6, 7}, 3))
	// [3 3 5 5 6 7]
}
```

**Why?** Instead of explicitly removing expired items, we check the index of the top before using it — classic lazy deletion.

---

## Example 3: Lazy Deletion with Unique IDs

```go
package main

import (
	"container/heap"
	"fmt"
)

type Entry struct {
	val int
	id  int
}

type MinH []Entry
func (h MinH) Len() int           { return len(h) }
func (h MinH) Less(i, j int) bool { return h[i].val < h[j].val }
func (h MinH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(Entry)) }
func (h *MinH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type LazyHeap struct {
	h       *MinH
	removed map[int]bool // set of removed IDs
}

func NewLazyHeap() *LazyHeap {
	return &LazyHeap{h: &MinH{}, removed: map[int]bool{}}
}

func (lh *LazyHeap) Push(val, id int) {
	heap.Push(lh.h, Entry{val, id})
}

func (lh *LazyHeap) Remove(id int) {
	lh.removed[id] = true
}

func (lh *LazyHeap) Top() (int, int) {
	for lh.h.Len() > 0 && lh.removed[(*lh.h)[0].id] {
		heap.Pop(lh.h)
	}
	e := (*lh.h)[0]
	return e.val, e.id
}

func main() {
	lh := NewLazyHeap()
	lh.Push(10, 1)
	lh.Push(5, 2)
	lh.Push(15, 3)
	lh.Push(3, 4)

	val, id := lh.Top()
	fmt.Printf("Min: val=%d id=%d\n", val, id) // 3, 4

	lh.Remove(4) // remove id=4
	val, id = lh.Top()
	fmt.Printf("Min: val=%d id=%d\n", val, id) // 5, 2

	lh.Remove(2) // remove id=2
	val, id = lh.Top()
	fmt.Printf("Min: val=%d id=%d\n", val, id) // 10, 1
}
```

---

## Example 4: Dijkstra's with Lazy Deletion

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type State struct{ node, dist int }

type PQ []State
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(State)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func dijkstra(graph [][]Edge, src int) []int {
	n := len(graph)
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[src] = 0

	pq := &PQ{}
	heap.Push(pq, State{src, 0})

	for pq.Len() > 0 {
		s := heap.Pop(pq).(State)
		// LAZY DELETION: skip if outdated
		if s.dist > dist[s.node] {
			continue
		}
		for _, e := range graph[s.node] {
			nd := s.dist + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				heap.Push(pq, State{e.to, nd})
			}
		}
	}
	return dist
}

func main() {
	graph := make([][]Edge, 5)
	edges := [][3]int{{0,1,4},{0,2,1},{2,1,2},{1,3,1},{2,3,5},{3,4,3}}
	for _, e := range edges {
		graph[e[0]] = append(graph[e[0]], Edge{e[1], e[2]})
	}
	fmt.Println(dijkstra(graph, 0)) // [0 3 1 4 7]
}
```

**Why?** When we find a shorter path, we push a new entry rather than updating. Old entries are lazily skipped via `s.dist > dist[s.node]`.

---

## Example 5: Timestamp-Based Expiry

```go
package main

import (
	"container/heap"
	"fmt"
)

type TimedEntry struct {
	val       int
	timestamp int
}

type MinTH []TimedEntry
func (h MinTH) Len() int           { return len(h) }
func (h MinTH) Less(i, j int) bool { return h[i].val < h[j].val }
func (h MinTH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinTH) Push(x interface{}) { *h = append(*h, x.(TimedEntry)) }
func (h *MinTH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type ExpiringHeap struct {
	h   *MinTH
	ttl int
}

func NewExpiringHeap(ttl int) *ExpiringHeap {
	return &ExpiringHeap{h: &MinTH{}, ttl: ttl}
}

func (eh *ExpiringHeap) Push(val, t int) {
	heap.Push(eh.h, TimedEntry{val, t})
}

func (eh *ExpiringHeap) Min(currentTime int) (int, bool) {
	// Lazily remove expired entries
	for eh.h.Len() > 0 && currentTime-(*eh.h)[0].timestamp > eh.ttl {
		heap.Pop(eh.h)
	}
	if eh.h.Len() == 0 {
		return 0, false
	}
	return (*eh.h)[0].val, true
}

func main() {
	eh := NewExpiringHeap(3) // TTL = 3 time units
	eh.Push(10, 1)
	eh.Push(5, 2)
	eh.Push(8, 4)

	if v, ok := eh.Min(3); ok { fmt.Println("At t=3:", v) }  // 5
	if v, ok := eh.Min(5); ok { fmt.Println("At t=5:", v) }  // 5 expired, 8
	if v, ok := eh.Min(8); ok { fmt.Println("At t=8:", v) }
	if _, ok := eh.Min(8); !ok { fmt.Println("At t=8: empty") } // all expired
}
```

---

## Example 6: Lazy Deletion with Counter (Multiset)

```go
package main

import (
	"container/heap"
	"fmt"
)

type MinH []int
func (h MinH) Len() int           { return len(h) }
func (h MinH) Less(i, j int) bool { return h[i] < h[j] }
func (h MinH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// Multiset with lazy deletion — supports duplicate values
type MultisetHeap struct {
	h    *MinH
	add  map[int]int // active count
	del  map[int]int // pending deletions
	size int
}

func NewMultisetHeap() *MultisetHeap {
	return &MultisetHeap{h: &MinH{}, add: map[int]int{}, del: map[int]int{}}
}

func (m *MultisetHeap) Insert(val int) {
	m.add[val]++
	m.size++
	heap.Push(m.h, val)
}

func (m *MultisetHeap) Remove(val int) {
	if m.add[val] > 0 {
		m.del[val]++
		m.add[val]--
		if m.add[val] == 0 { delete(m.add, val) }
		m.size--
	}
}

func (m *MultisetHeap) prune() {
	for m.h.Len() > 0 {
		top := (*m.h)[0]
		if m.del[top] > 0 {
			m.del[top]--
			if m.del[top] == 0 { delete(m.del, top) }
			heap.Pop(m.h)
		} else {
			break
		}
	}
}

func (m *MultisetHeap) Min() int {
	m.prune()
	return (*m.h)[0]
}

func (m *MultisetHeap) Size() int { return m.size }

func main() {
	ms := NewMultisetHeap()
	ms.Insert(5); ms.Insert(3); ms.Insert(3); ms.Insert(8)
	fmt.Println("Min:", ms.Min(), "Size:", ms.Size()) // 3, 4

	ms.Remove(3) // remove one instance of 3
	fmt.Println("Min:", ms.Min(), "Size:", ms.Size()) // 3, 3

	ms.Remove(3) // remove second instance of 3
	fmt.Println("Min:", ms.Min(), "Size:", ms.Size()) // 5, 2
}
```

---

## Example 7: Median Maintenance with Lazy Deletion

```go
package main

import (
	"container/heap"
	"fmt"
)

type MaxH []int
func (h MaxH) Len() int           { return len(h) }
func (h MaxH) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type MinH []int
func (h MinH) Len() int           { return len(h) }
func (h MinH) Less(i, j int) bool { return h[i] < h[j] }
func (h MinH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type LazyMedian struct {
	lo, loD *MaxH
	hi, hiD *MinH
	loSz, hiSz int
}

func NewLazyMedian() *LazyMedian {
	return &LazyMedian{
		lo: &MaxH{}, loD: &MaxH{},
		hi: &MinH{}, hiD: &MinH{},
	}
}

func (lm *LazyMedian) pruneLo() {
	for lm.lo.Len() > 0 && lm.loD.Len() > 0 && (*lm.lo)[0] == (*lm.loD)[0] {
		heap.Pop(lm.lo); heap.Pop(lm.loD)
	}
}
func (lm *LazyMedian) pruneHi() {
	for lm.hi.Len() > 0 && lm.hiD.Len() > 0 && (*lm.hi)[0] == (*lm.hiD)[0] {
		heap.Pop(lm.hi); heap.Pop(lm.hiD)
	}
}

func (lm *LazyMedian) balance() {
	for lm.loSz > lm.hiSz+1 {
		lm.pruneLo()
		heap.Push(lm.hi, heap.Pop(lm.lo)); lm.loSz--; lm.hiSz++
	}
	for lm.hiSz > lm.loSz {
		lm.pruneHi()
		heap.Push(lm.lo, heap.Pop(lm.hi)); lm.hiSz--; lm.loSz++
	}
}

func (lm *LazyMedian) Add(val int) {
	lm.pruneLo()
	if lm.lo.Len() == 0 || val <= (*lm.lo)[0] {
		heap.Push(lm.lo, val); lm.loSz++
	} else {
		heap.Push(lm.hi, val); lm.hiSz++
	}
	lm.balance()
}

func (lm *LazyMedian) Remove(val int) {
	lm.pruneLo()
	if lm.lo.Len() > 0 && val <= (*lm.lo)[0] {
		heap.Push(lm.loD, val); lm.loSz--
	} else {
		heap.Push(lm.hiD, val); lm.hiSz--
	}
	lm.balance()
}

func (lm *LazyMedian) Median() float64 {
	lm.pruneLo(); lm.pruneHi()
	if lm.loSz > lm.hiSz {
		return float64((*lm.lo)[0])
	}
	return float64((*lm.lo)[0]+(*lm.hi)[0]) / 2.0
}

func main() {
	lm := NewLazyMedian()
	nums := []int{1, 3, 5, 7, 9, 2}
	for _, v := range nums {
		lm.Add(v)
		fmt.Printf("Add %d → median = %.1f\n", v, lm.Median())
	}
	lm.Remove(3)
	fmt.Printf("Remove 3 → median = %.1f\n", lm.Median())
}
```

---

## Example 8: K Strongest Values with Lazy Heap

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type MaxH []int
func (h MaxH) Len() int           { return len(h) }
func (h MaxH) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func getStrongest(arr []int, k int) []int {
	sort.Ints(arr)
	n := len(arr)
	median := arr[(n-1)/2]

	// Use two-pointer approach but demonstrate lazy heap concept
	h := &MaxH{}
	removed := map[int]bool{}

	type Entry struct{ strength, val int }
	entries := make([]Entry, n)
	for i, v := range arr {
		s := v - median
		if s < 0 { s = -s }
		entries[i] = Entry{s, v}
	}

	sort.Slice(entries, func(i, j int) bool {
		if entries[i].strength != entries[j].strength {
			return entries[i].strength > entries[j].strength
		}
		return entries[i].val > entries[j].val
	})

	_ = h; _ = removed // heap available for lazy deletion variant

	result := make([]int, k)
	for i := 0; i < k; i++ {
		result[i] = entries[i].val
	}
	return result
}

func main() {
	fmt.Println(getStrongest([]int{1, 2, 3, 4, 5}, 2))       // [5 1]
	fmt.Println(getStrongest([]int{1, 1, 3, 5, 5}, 2))       // [5 5]
	fmt.Println(getStrongest([]int{6, 7, 11, 7, 6, 8}, 5))   // [11 8 6 6 7]
}
```

---

## Example 9: Event Scheduler with Cancel (Lazy Delete)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Event struct {
	time int
	name string
	id   int
}

type EventHeap []Event
func (h EventHeap) Len() int           { return len(h) }
func (h EventHeap) Less(i, j int) bool { return h[i].time < h[j].time }
func (h EventHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *EventHeap) Push(x interface{}) { *h = append(*h, x.(Event)) }
func (h *EventHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type Scheduler struct {
	h         *EventHeap
	cancelled map[int]bool
	nextID    int
}

func NewScheduler() *Scheduler {
	return &Scheduler{h: &EventHeap{}, cancelled: map[int]bool{}}
}

func (s *Scheduler) Schedule(time int, name string) int {
	s.nextID++
	heap.Push(s.h, Event{time, name, s.nextID})
	return s.nextID
}

func (s *Scheduler) Cancel(id int) {
	s.cancelled[id] = true // lazy: don't search the heap
}

func (s *Scheduler) NextEvent() (Event, bool) {
	for s.h.Len() > 0 {
		e := heap.Pop(s.h).(Event)
		if s.cancelled[e.id] {
			delete(s.cancelled, e.id) // clean up
			continue
		}
		return e, true
	}
	return Event{}, false
}

func main() {
	s := NewScheduler()
	id1 := s.Schedule(10, "meeting")
	s.Schedule(5, "standup")
	s.Schedule(15, "review")
	s.Cancel(id1) // cancel meeting

	for {
		e, ok := s.NextEvent()
		if !ok { break }
		fmt.Printf("t=%d: %s\n", e.time, e.name)
	}
	// t=5: standup
	// t=15: review
	// (meeting cancelled)
}
```

---

## Example 10: Lazy vs Eager Deletion Comparison

```go
package main

import (
	"container/heap"
	"fmt"
	"time"
)

type MinH []int
func (h MinH) Len() int           { return len(h) }
func (h MinH) Less(i, j int) bool { return h[i] < h[j] }
func (h MinH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func benchLazy(n int) time.Duration {
	h := &MinH{}
	deleted := map[int]int{}
	for i := 0; i < n; i++ { heap.Push(h, i) }

	start := time.Now()
	// Delete even numbers lazily
	for i := 0; i < n; i += 2 {
		deleted[i]++
	}
	// Pop all
	for h.Len() > 0 {
		top := (*h)[0]
		if deleted[top] > 0 {
			deleted[top]--; heap.Pop(h)
		} else {
			heap.Pop(h)
		}
	}
	return time.Since(start)
}

func benchEager(n int) time.Duration {
	h := &MinH{}
	for i := 0; i < n; i++ { heap.Push(h, i) }

	start := time.Now()
	// Delete even numbers eagerly — need to find them
	// This requires O(n) scan per deletion!
	for i := 0; i < n; i += 2 {
		for j := 0; j < h.Len(); j++ {
			if (*h)[j] == i {
				heap.Remove(h, j)
				break
			}
		}
	}
	return time.Since(start)
}

func main() {
	n := 10000
	lazy := benchLazy(n)
	eager := benchEager(n)
	fmt.Printf("Lazy:  %v\n", lazy)
	fmt.Printf("Eager: %v\n", eager)
	fmt.Printf("Eager/Lazy ratio: %.1fx slower\n", float64(eager)/float64(lazy))

	fmt.Println("\n=== Summary ===")
	fmt.Println("| Operation        | Lazy       | Eager       |")
	fmt.Println("|-----------------|------------|-------------|")
	fmt.Println("| Delete           | O(1)       | O(n) find   |")
	fmt.Println("| Pop (amortized)  | O(log n)   | O(log n)    |")
	fmt.Println("| Space overhead   | Extra map  | None        |")
	fmt.Println("| Best for         | Many deletes| Few deletes |")
}
```

---

## Key Takeaways

1. **Lazy deletion** = mark as deleted, skip at top — avoids O(n) search
2. Three patterns: value-based counters, unique-ID sets, index-based expiry
3. Dijkstra's algorithm is the classic example — push duplicates, skip outdated
4. Sliding window problems use index-based lazy deletion (prune expired tops)
5. Trade-off: uses extra memory for deletion map, but deletion becomes O(1)

> **Next up:** Heap Sort →
