# Phase 11: Heap / Priority Queue — Heap Operations

## Overview

Core operations on a heap:

| Operation | Time | Description |
|-----------|------|-------------|
| **Push** (Insert) | O(log n) | Add element, sift up |
| **Pop** (Extract) | O(log n) | Remove root, sift down |
| **Peek** (Top) | O(1) | View root without removing |
| **Remove** (arbitrary) | O(log n) | Remove specific element, sift up/down |
| **Update** (change key) | O(log n) | Change value, sift up or down |
| **Build** (Heapify) | O(n) | Convert array to heap |

---

## Example 1: All Basic Operations

```go
package main

import "fmt"

type Heap struct {
	data []int
}

func NewHeap() *Heap { return &Heap{} }

func (h *Heap) Push(val int) {
	h.data = append(h.data, val)
	h.siftUp(len(h.data) - 1)
}

func (h *Heap) Pop() int {
	root := h.data[0]
	n := len(h.data) - 1
	h.data[0] = h.data[n]
	h.data = h.data[:n]
	if n > 0 {
		h.siftDown(0)
	}
	return root
}

func (h *Heap) Peek() int  { return h.data[0] }
func (h *Heap) Len() int   { return len(h.data) }
func (h *Heap) Empty() bool { return len(h.data) == 0 }

func (h *Heap) siftUp(i int) {
	for i > 0 {
		p := (i - 1) / 2
		if h.data[i] < h.data[p] {
			h.data[i], h.data[p] = h.data[p], h.data[i]
			i = p
		} else {
			break
		}
	}
}

func (h *Heap) siftDown(i int) {
	n := len(h.data)
	for {
		min := i
		l, r := 2*i+1, 2*i+2
		if l < n && h.data[l] < h.data[min] { min = l }
		if r < n && h.data[r] < h.data[min] { min = r }
		if min == i { break }
		h.data[i], h.data[min] = h.data[min], h.data[i]
		i = min
	}
}

func main() {
	h := NewHeap()
	h.Push(5); h.Push(3); h.Push(8); h.Push(1)
	fmt.Println("Peek:", h.Peek())  // 1
	fmt.Println("Pop:", h.Pop())    // 1
	fmt.Println("Peek:", h.Peek())  // 3
	fmt.Println("Size:", h.Len())   // 3
}
```

**Textual Figure:**

```
Min Heap — Basic Operations Trace:

  Push 5:   Push 3:     Push 8:     Push 1:
    5         3            3           1
             / \          / \         / \
            5  (siftUp)  5   8       3   8
                         3                5
                        / \              /
                       5   8            (siftUp 1→3→1)

  State after all pushes:     Array: [1, 3, 8, 5]
         1(0)
        / \
      3(1)  8(2)
     /
    5(3)

  Operations:
  ┌───────────┬────────┬───────────────────────┐
  │ Operation │ Result │ Heap after             │
  ├───────────┼────────┼───────────────────────┤
  │ Peek()    │   1    │ [1, 3, 8, 5] (no chg)  │
  │ Pop()     │   1    │ [3, 5, 8]    (siftDown)│
  │ Peek()    │   3    │ [3, 5, 8]              │
  │ Len()     │   3    │ [3, 5, 8]              │
  └───────────┴────────┴───────────────────────┘
```

---

## Example 2: Remove Arbitrary Element

```go
package main

import (
	"container/heap"
	"fmt"
)

type IndexedHeap struct {
	data  []int
	index map[int]int // val -> position (assumes unique values)
}

func (h IndexedHeap) Len() int           { return len(h.data) }
func (h IndexedHeap) Less(i, j int) bool { return h.data[i] < h.data[j] }
func (h IndexedHeap) Swap(i, j int) {
	h.index[h.data[i]] = j
	h.index[h.data[j]] = i
	h.data[i], h.data[j] = h.data[j], h.data[i]
}
func (h *IndexedHeap) Push(x interface{}) {
	h.index[x.(int)] = len(h.data)
	h.data = append(h.data, x.(int))
}
func (h *IndexedHeap) Pop() interface{} {
	old := h.data
	n := len(old)
	x := old[n-1]
	h.data = old[:n-1]
	delete(h.index, x)
	return x
}

func (h *IndexedHeap) Remove(val int) {
	if idx, ok := h.index[val]; ok {
		heap.Remove(h, idx)
	}
}

func main() {
	h := &IndexedHeap{index: map[int]int{}}
	for _, v := range []int{5, 3, 8, 1, 2, 7} {
		heap.Push(h, v)
	}
	fmt.Println("Before remove:", h.data) // min heap

	h.Remove(3) // remove element 3
	fmt.Println("After remove 3:", h.data)

	for h.Len() > 0 {
		fmt.Print(heap.Pop(h), " ") // 1 2 5 7 8
	}
	fmt.Println()
}
```

**Textual Figure:**

```
Input: [5, 3, 8, 1, 2, 7] → build min heap with index tracking

  Min heap with index map:
         1(0)              index map: {1:0, 2:1, 7:2, 5:3, 3:4, 8:5}
        / \
      2(1)  7(2)
     / \   /
    5(3) 3(4) 8(5)

  Remove(3) — remove element with value 3:
  ┌────────────────────────────────────────┐
  │ 1. Look up index: index[3] = 4        │
  │ 2. Swap with last: swap(4, 5)          │
  │    [1, 2, 7, 5, 8, 3] → remove last   │
  │ 3. [1, 2, 7, 5, 8] → sift up/down     │
  │    8 at pos 4: parent=2 → 8>2 no sift │
  └────────────────────────────────────────┘

  After remove 3:           Array: [1, 2, 7, 5, 8]
         1(0)
        / \
      2(1)  7(2)
     / \
    5(3) 8(4)

  Pop order: 1 2 5 7 8
```

---

## Example 3: Update Key (Decrease Key / Increase Key)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Item struct {
	value    string
	priority int
	index    int // position in heap
}

type PQ []*Item
func (pq PQ) Len() int           { return len(pq) }
func (pq PQ) Less(i, j int) bool { return pq[i].priority < pq[j].priority }
func (pq PQ) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}
func (pq *PQ) Push(x interface{}) {
	item := x.(*Item)
	item.index = len(*pq)
	*pq = append(*pq, item)
}
func (pq *PQ) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	item.index = -1
	*pq = old[:n-1]
	return item
}

func (pq *PQ) UpdatePriority(item *Item, newPriority int) {
	item.priority = newPriority
	heap.Fix(pq, item.index) // O(log n) - sifts up or down
}

func main() {
	items := []*Item{
		{value: "task1", priority: 3},
		{value: "task2", priority: 1},
		{value: "task3", priority: 2},
	}
	pq := &PQ{}
	for _, item := range items {
		heap.Push(pq, item)
	}

	fmt.Println("Before update:")
	fmt.Printf("  Top: %s (pri=%d)\n", (*pq)[0].value, (*pq)[0].priority)

	// Decrease priority of task1 from 3 to 0
	pq.UpdatePriority(items[0], 0)
	fmt.Println("After update task1 priority to 0:")
	fmt.Printf("  Top: %s (pri=%d)\n", (*pq)[0].value, (*pq)[0].priority)

	for pq.Len() > 0 {
		item := heap.Pop(pq).(*Item)
		fmt.Printf("  %s (pri=%d)\n", item.value, item.priority)
	}
}
```

**Textual Figure:**

```
Priority Queue with Update Key:

  Initial heap (min priority):
       (task2, pri=1)
        /           \
  (task1, pri=3)  (task3, pri=2)

  UpdatePriority(task1, 3 → 0):
  ┌───────────────────────────────────────────┐
  │ task1.priority = 0                         │
  │ heap.Fix(pq, task1.index)                  │
  │ 0 < 1 (parent) → sift UP                  │
  │ Swap task1 ↔ task2                         │
  └───────────────────────────────────────────┘

  After update:
       (task1, pri=0)    ← new root!
        /           \
  (task2, pri=1)  (task3, pri=2)

  Pop order: task1(0) → task2(1) → task3(2)
```

---

## Example 4: heap.Fix for Dijkstra's Algorithm

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, weight int }
type Node struct {
	id, dist, index int
}
type NodeHeap []*Node
func (h NodeHeap) Len() int           { return len(h) }
func (h NodeHeap) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h NodeHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
	h[i].index = i; h[j].index = j
}
func (h *NodeHeap) Push(x interface{}) {
	node := x.(*Node); node.index = len(*h); *h = append(*h, node)
}
func (h *NodeHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func dijkstra(graph [][]Edge, src int) []int {
	n := len(graph)
	dist := make([]int, n)
	nodes := make([]*Node, n)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[src] = 0

	h := &NodeHeap{}
	for i := 0; i < n; i++ {
		nodes[i] = &Node{i, dist[i], -1}
		heap.Push(h, nodes[i])
	}

	for h.Len() > 0 {
		u := heap.Pop(h).(*Node)
		for _, e := range graph[u.id] {
			newDist := dist[u.id] + e.weight
			if newDist < dist[e.to] {
				dist[e.to] = newDist
				nodes[e.to].dist = newDist
				if nodes[e.to].index >= 0 {
					heap.Fix(h, nodes[e.to].index)
				}
			}
		}
	}
	return dist
}

func main() {
	graph := [][]Edge{
		{{1, 4}, {2, 1}},
		{{3, 1}},
		{{1, 2}, {3, 5}},
		{},
	}
	fmt.Println(dijkstra(graph, 0)) // [0 3 1 4]
}
```

**Textual Figure:**

```
Graph (adjacency list):
     0 ──4──→ 1
     │           │
     1           1
     │           │
     ▼           ▼
     2 ──5──→ 3
     │           ▲
     2           │
     │           │
     └─────────┘
         via 1

  Dijkstra with heap.Fix:
  ┌──────┬──────────────┬────────────────────────┐
  │ Pop  │  dist[]     │ Updates (heap.Fix)     │
  ├──────┼──────────────┼────────────────────────┤
  │  0   │ [0,∞,∞,∞]   │ d[1]=4, d[2]=1         │
  │  2   │ [0,4,1,∞]   │ d[1]=min(4,1+2)=3     │
  │      │              │ d[3]=min(∞,1+5)=6     │
  │  1   │ [0,3,1,6]   │ d[3]=min(6,3+1)=4     │
  │  3   │ [0,3,1,4]   │ (done)                 │
  └──────┴──────────────┴────────────────────────┘

  Result: dist = [0, 3, 1, 4]
  Path: 0→1 costs 3 (via 0→2→1)
  Path: 0→3 costs 4 (via 0→2→1→3)
```

---

## Example 5: Push and Pop Combined (Replace Top)

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// ReplaceTop: more efficient than Pop + Push
func replaceTop(h *IntHeap, val int) int {
	old := (*h)[0]
	(*h)[0] = val
	heap.Fix(h, 0)
	return old
}

func main() {
	h := &IntHeap{1, 3, 5, 7, 9}
	heap.Init(h)
	fmt.Println("Before:", *h)

	// Keep top-3 largest using min heap of size 3
	stream := []int{2, 8, 4, 6, 10, 1}
	topK := &IntHeap{}
	heap.Init(topK)
	
	// Initialize with first 3
	for i := 0; i < 3 && i < len(stream); i++ {
		heap.Push(topK, stream[i])
	}
	
	for i := 3; i < len(stream); i++ {
		if stream[i] > (*topK)[0] {
			replaceTop(topK, stream[i])
		}
	}
	fmt.Println("Top 3:", *topK) // min heap with 3 largest values
}
```

**Textual Figure:**

```
ReplaceTop: more efficient than Pop + Push

Stream: [2, 8, 4, 6, 10, 1]    Keep top-3 largest (min heap of size 3)

  Init with first 3:  [2, 8, 4] → min heap: [2, 8, 4]

         2               2 = min of top-3
        / \
       8   4

  Process remaining:
  ┌──────┬────────────┬─────────────────────────┐
  │ Elem │ vs top (2) │ Action                  │
  ├──────┼────────────┼─────────────────────────┤
  │   6  │   6 > 2    │ replaceTop(6) → [4,8,6] │
  │  10  │  10 > 4    │ replaceTop(10)→ [6,8,10]│
  │   1  │   1 < 6    │ skip                    │
  └──────┴────────────┴─────────────────────────┘

  Final top-3 heap:     6
                       / \
                      8   10

  replaceTop = O(log n) vs Pop+Push = O(2 log n)
```

---

## Example 6: Merge K Sorted Arrays

```go
package main

import (
	"container/heap"
	"fmt"
)

type Entry struct {
	val, arrIdx, elemIdx int
}
type EntryHeap []Entry
func (h EntryHeap) Len() int           { return len(h) }
func (h EntryHeap) Less(i, j int) bool { return h[i].val < h[j].val }
func (h EntryHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *EntryHeap) Push(x interface{}) { *h = append(*h, x.(Entry)) }
func (h *EntryHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func mergeKSorted(arrays [][]int) []int {
	h := &EntryHeap{}
	for i, arr := range arrays {
		if len(arr) > 0 {
			heap.Push(h, Entry{arr[0], i, 0})
		}
	}
	var result []int
	for h.Len() > 0 {
		e := heap.Pop(h).(Entry)
		result = append(result, e.val)
		if e.elemIdx+1 < len(arrays[e.arrIdx]) {
			heap.Push(h, Entry{arrays[e.arrIdx][e.elemIdx+1], e.arrIdx, e.elemIdx + 1})
		}
	}
	return result
}

func main() {
	arrays := [][]int{
		{1, 4, 7},
		{2, 5, 8},
		{3, 6, 9},
	}
	fmt.Println(mergeKSorted(arrays)) // [1 2 3 4 5 6 7 8 9]
}
```

**Textual Figure:**

```
Merge K=3 sorted arrays:
  Array 0: [1, 4, 7]
  Array 1: [2, 5, 8]
  Array 2: [3, 6, 9]

  Min heap of (val, arrIdx, elemIdx):
  ┌──────┬──────────────┬────────────┬──────────┐
  │ Step │  Pop         │ Push next  │ Result   │
  ├──────┼──────────────┼────────────┼──────────┤
  │ Init │  -           │ [1,2,3]    │ []       │
  │   1  │ (1,arr0,0)   │ (4,arr0,1) │ [1]      │
  │   2  │ (2,arr1,0)   │ (5,arr1,1) │ [1,2]    │
  │   3  │ (3,arr2,0)   │ (6,arr2,1) │ [1,2,3]  │
  │   4  │ (4,arr0,1)   │ (7,arr0,2) │ [..4]    │
  │  ..  │  ...         │ ...        │ ...      │
  │   9  │ (9,arr2,2)   │ (none)     │ [1..9]   │
  └──────┴──────────────┴────────────┴──────────┘

  Heap always has ≤ k nodes → O(N log k)
  Result: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## Example 7: Peek and Conditional Pop

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// Simulate a timer/event system — pop only if event time <= current time
func processEvents(events []int, currentTime int) ([]int, *IntHeap) {
	h := &IntHeap{}
	for _, e := range events {
		heap.Push(h, e)
	}
	var processed []int
	for h.Len() > 0 && (*h)[0] <= currentTime {
		processed = append(processed, heap.Pop(h).(int))
	}
	return processed, h
}

func main() {
	events := []int{5, 1, 3, 7, 2, 8, 4}
	processed, remaining := processEvents(events, 4)
	fmt.Println("Processed (≤4):", processed)    // [1 2 3 4]
	fmt.Println("Remaining:", remaining.Len(), "events") // 3 events
}
```

**Textual Figure:**

```
Event times: [5, 1, 3, 7, 2, 8, 4]    currentTime = 4

  Build min heap:
         1(0)
        / \
      2(1)  3(2)
     / \   / \
    7   5  8   4

  Peek and pop while top ≤ 4:
  ┌──────┬───────┬──────────┬───────────────┐
  │ Peek │ ≤ 4?  │ Action   │ Processed     │
  ├──────┼───────┼──────────┼───────────────┤
  │   1  │  Yes  │ Pop      │ [1]           │
  │   2  │  Yes  │ Pop      │ [1,2]         │
  │   3  │  Yes  │ Pop      │ [1,2,3]       │
  │   4  │  Yes  │ Pop      │ [1,2,3,4]     │
  │   5  │  No   │ Stop     │ [1,2,3,4]     │
  └──────┴───────┴──────────┴───────────────┘

  Remaining: 3 events [5, 7, 8]
```

---

## Example 8: Multi-way Merge with Tracking

```go
package main

import (
	"container/heap"
	"fmt"
)

type StreamItem struct {
	val      int
	streamID int
}
type StreamHeap []StreamItem
func (h StreamHeap) Len() int           { return len(h) }
func (h StreamHeap) Less(i, j int) bool { return h[i].val < h[j].val }
func (h StreamHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *StreamHeap) Push(x interface{}) { *h = append(*h, x.(StreamItem)) }
func (h *StreamHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func main() {
	// Simulate reading from multiple sorted streams
	streams := map[int][]int{
		0: {1, 5, 9, 13},
		1: {2, 6, 10, 14},
		2: {3, 7, 11, 15},
	}
	cursors := map[int]int{0: 0, 1: 0, 2: 0}

	h := &StreamHeap{}
	for id, data := range streams {
		heap.Push(h, StreamItem{data[0], id})
		cursors[id] = 1
	}

	for h.Len() > 0 {
		item := heap.Pop(h).(StreamItem)
		fmt.Printf("val=%2d from stream %d\n", item.val, item.streamID)

		// Push next from same stream
		if cursors[item.streamID] < len(streams[item.streamID]) {
			next := streams[item.streamID][cursors[item.streamID]]
			heap.Push(h, StreamItem{next, item.streamID})
			cursors[item.streamID]++
		}
	}
}
```

**Textual Figure:**

```
3 sorted streams:
  Stream 0: [1, 5, 9, 13]
  Stream 1: [2, 6, 10, 14]
  Stream 2: [3, 7, 11, 15]

  Min heap merges all streams maintaining source ID:
  ┌──────┬─────┬──────────┬──────────────────┐
  │ Step │ Val │ Stream   │ Heap after push │
  ├──────┼─────┼──────────┼──────────────────┤
  │   1  │  1  │ stream 0 │ push 5 [2,3,5]  │
  │   2  │  2  │ stream 1 │ push 6 [3,5,6]  │
  │   3  │  3  │ stream 2 │ push 7 [5,6,7]  │
  │   4  │  5  │ stream 0 │ push 9 [6,7,9]  │
  │  ... │ ... │ ...      │ ...             │
  │  12  │ 15  │ stream 2 │ (done)          │
  └──────┴─────┴──────────┴──────────────────┘

  Output: 1,2,3,5,6,7,9,10,11,13,14,15 (with stream tracking)
```

---

## Example 9: Bounded Heap (Fixed Size)

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type BoundedHeap struct {
	heap *IntHeap
	cap  int
}

func NewBoundedHeap(cap int) *BoundedHeap {
	h := &IntHeap{}
	return &BoundedHeap{heap: h, cap: cap}
}

func (bh *BoundedHeap) Add(val int) {
	if bh.heap.Len() < bh.cap {
		heap.Push(bh.heap, val)
	} else if val > (*bh.heap)[0] {
		(*bh.heap)[0] = val
		heap.Fix(bh.heap, 0)
	}
}

func (bh *BoundedHeap) TopK() []int {
	result := make([]int, bh.heap.Len())
	copy(result, *bh.heap)
	return result
}

func main() {
	bh := NewBoundedHeap(3)
	for _, v := range []int{5, 2, 8, 1, 9, 3, 7, 4, 6} {
		bh.Add(v)
		fmt.Printf("Add %d → top-3: %v\n", v, bh.TopK())
	}
}
```

**Textual Figure:**

```
Bounded Heap (cap=3): maintains top-3 largest

Stream: [5, 2, 8, 1, 9, 3, 7, 4, 6]

  ┌──────┬──────────────┬────────────────────────┐
  │ Add  │ Action       │ Min heap (cap=3)       │
  ├──────┼──────────────┼────────────────────────┤
  │   5  │ push         │ [5]                    │
  │   2  │ push         │ [2, 5]                 │
  │   8  │ push (full)  │ [2, 5, 8]              │
  │   1  │ 1<2 skip     │ [2, 5, 8]              │
  │   9  │ 9>2 replace  │ [5, 8, 9]  (Fix idx 0) │
  │   3  │ 3<5 skip     │ [5, 8, 9]              │
  │   7  │ 7>5 replace  │ [7, 8, 9]              │
  │   4  │ 4<7 skip     │ [7, 8, 9]              │
  │   6  │ 6<7 skip     │ [7, 8, 9]              │
  └──────┴──────────────┴────────────────────────┘

  Final: top-3 = [7, 8, 9]   O(n log k) total
```

---

## Example 10: Operation Complexity Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Heap Operations Complexity ===")
	fmt.Println()
	
	ops := []struct {
		name string
		time string
		desc string
	}{
		{"Push", "O(log n)", "Insert + sift up"},
		{"Pop", "O(log n)", "Extract root + sift down"},
		{"Peek", "O(1)", "View root"},
		{"heap.Init", "O(n)", "Bottom-up heapify"},
		{"heap.Fix", "O(log n)", "Update key at index i"},
		{"heap.Remove", "O(log n)", "Remove element at index i"},
		{"ReplaceTop", "O(log n)", "Change root + sift down (via Fix)"},
		{"Build from n pushes", "O(n log n)", "n insertions"},
	}
	
	for _, op := range ops {
		fmt.Printf("%-20s %-12s %s\n", op.name, op.time, op.desc)
	}

	fmt.Println()
	fmt.Println("Key: heap.Fix is the most versatile — handles update, decrease/increase key")
	fmt.Println("Key: For top-K, bounded min heap is O(n log k) — better than sorting O(n log n)")
}
```

**Textual Figure:**

```
Heap Operations Complexity Summary:

  ┌────────────────────┬────────────┬──────────────────────────┐
  │     Operation      │    Time    │       How it works       │
  ├────────────────────┼────────────┼──────────────────────────┤
  │ Push (Insert)      │ O(log n)   │ append + sift UP    ↑    │
  │ Pop (Extract)      │ O(log n)   │ swap root + sift DOWN ↓  │
  │ Peek (Top)         │ O(1)       │ return arr[0]           │
  │ heap.Init          │ O(n)       │ bottom-up heapify       │
  │ heap.Fix           │ O(log n)   │ sift up OR down at idx  │
  │ heap.Remove        │ O(log n)   │ swap+shrink+Fix         │
  │ ReplaceTop         │ O(log n)   │ set arr[0] + Fix(0)     │
  │ Build (n pushes)   │ O(n log n) │ n × O(log n) inserts     │
  └────────────────────┴────────────┴──────────────────────────┘

  Sift Up:             Sift Down:
     ○ ← child          ○ ← parent
    / \                / \
   ○   ○              ○   ○ ← children
  Compare with       Compare with
  parent, swap up    smaller child, swap down
```

---

## Key Takeaways

1. `heap.Fix(h, i)` — the most important operation for updates: O(log n)
2. `heap.Remove(h, i)` — remove arbitrary element by index: O(log n)
3. For Dijkstra / priority updates, track `.index` in the struct
4. Bounded heap (size k) maintains top-k in O(n log k)
5. Replace-top (`Fix` at index 0) is faster than Pop + Push

> **Next up:** K-Largest Problems →
