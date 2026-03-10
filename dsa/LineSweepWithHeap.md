# Phase 18: Intervals — Line Sweep with Heap

## Overview

**Line sweep with heap** combines the sweep line approach with a priority queue (heap) to efficiently track the maximum, minimum, or top-K active elements as we sweep across events. The heap maintains the active set, enabling O(log n) updates.

| Component | Purpose |
|-----------|---------|
| Events | Sorted start/end points |
| Min/Max-Heap | Track active intervals by some property |
| Lazy deletion | Remove expired entries when they reach top |

**Time:** O(n log n) — sorting + n heap operations.

---

## Example 1: Meeting Rooms II with Min-Heap

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minMeetingRooms(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	h := &MinHeap{}
	heap.Init(h)

	for _, iv := range intervals {
		// If earliest ending room is free before this meeting starts
		if h.Len() > 0 && (*h)[0] <= iv[0] {
			heap.Pop(h)
		}
		heap.Push(h, iv[1])
	}
	return h.Len()
}

func main() {
	meetings := [][]int{{0, 30}, {5, 10}, {15, 20}}
	fmt.Println("Min rooms:", minMeetingRooms(meetings)) // 2

	meetings2 := [][]int{{7, 10}, {2, 4}}
	fmt.Println("Min rooms:", minMeetingRooms(meetings2)) // 1
}
```

**Textual Figure:**

```
Meeting Rooms II with Min-Heap

Case 1: meetings=[[0,30],[5,10],[15,20]]

  Timeline:
  0    5   10   15   20   25   30
  ├────┼────┼────┼────┼────┼────┤
  M1: ██████████████████████████████
  M2:      █████
  M3:                █████

  Min-heap of end times (process sorted by start):
    [0,30]: heap empty → push 30         heap=[30]     rooms=1
    [5,10]: top=30 > 5 → push 10         heap=[10,30]  rooms=2
    [15,20]: top=10 ≤ 15 → pop, push 20  heap=[20,30]  rooms=2

  Answer: 2 rooms

Case 2: meetings=[[7,10],[2,4]]
    [2,4]: push 4              heap=[4]   rooms=1
    [7,10]: top=4 ≤ 7 → pop, push 10  heap=[10]  rooms=1
  Answer: 1 room
```

---

## Example 2: Skyline Problem with Max-Heap

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type HeightEntry struct {
	height, end int
}
type MaxHeap []HeightEntry
func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i].height > h[j].height }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(HeightEntry)) }
func (h *MaxHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func getSkyline(buildings [][]int) [][]int {
	// Create events
	events := [][]int{} // [x, height, isStart]
	for _, b := range buildings {
		events = append(events, []int{b[0], b[2], 1})  // start
		events = append(events, []int{b[1], b[2], 0})  // end
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i][0] != events[j][0] {
			return events[i][0] < events[j][0]
		}
		// At same x: starts before ends, taller starts first, shorter ends first
		if events[i][2] != events[j][2] {
			return events[i][2] > events[j][2]
		}
		if events[i][2] == 1 {
			return events[i][1] > events[j][1]
		}
		return events[i][1] < events[j][1]
	})

	h := &MaxHeap{}
	heap.Push(h, HeightEntry{0, 1<<31 - 1}) // ground level

	result := [][]int{}
	prevMax := 0

	for _, e := range events {
		x, ht := e[0], e[1]
		if e[2] == 1 {
			heap.Push(h, HeightEntry{ht, 0}) // track end later via lazy deletion
		} else {
			// Mark for lazy deletion - we'll use a simpler approach
			// Add negative height marker
		}

		// Simplified: just track using active set approach
		// For full solution, use lazy deletion or balanced BST
		_ = x
		_ = ht
	}

	// Simpler approach using sorted multiset simulation
	h2 := &MaxHeap{}
	heap.Push(h2, HeightEntry{0, 1<<31 - 1})
	result = [][]int{}
	prevMax = 0

	type Event2 struct {
		x, height int
		isStart   bool
	}
	events2 := []Event2{}
	for _, b := range buildings {
		events2 = append(events2, Event2{b[0], b[2], true})
		events2 = append(events2, Event2{b[1], b[2], false})
	}
	sort.Slice(events2, func(i, j int) bool {
		if events2[i].x != events2[j].x {
			return events2[i].x < events2[j].x
		}
		if events2[i].isStart && events2[j].isStart {
			return events2[i].height > events2[j].height
		}
		if !events2[i].isStart && !events2[j].isStart {
			return events2[i].height < events2[j].height
		}
		return events2[i].isStart
	})

	active := map[int]int{0: 1}
	getMax := func() int {
		mx := 0
		for h, c := range active {
			if c > 0 && h > mx { mx = h }
		}
		return mx
	}

	for _, e := range events2 {
		if e.isStart {
			active[e.height]++
		} else {
			active[e.height]--
			if active[e.height] == 0 { delete(active, e.height) }
		}
		currMax := getMax()
		if currMax != prevMax {
			result = append(result, []int{e.x, currMax})
			prevMax = currMax
		}
	}

	fmt.Println("Skyline:", result)
	return result
}

func main() {
	buildings := [][]int{{2, 9, 10}, {3, 7, 15}, {5, 12, 12}}
	getSkyline(buildings)
}
```

**Textual Figure:**

```
Skyline Problem with Max-Heap: [[2,9,10],[3,7,15],[5,12,12]]

  Height
   15 │     ┌─────┐
      │     │     │
   12 │     │     │┌───────┐
      │     │     ││        │
   10 │  ┌──┤     ││        │
      │  │  │     ││        │
    0 │──┴──┴─────┴┴────────┴───
       2  3  5    7        12

  Events (sorted by x):
    x=2: START h=10 → push to max-heap
      heap={10,0}, max=10, prev=0 → output [2,10]
    x=3: START h=15 → push
      heap={15,10,0}, max=15 → output [3,15]
    x=5: START h=12 → push
      heap={15,12,10,0}, max=15 (no change)
    x=7: END h=15 → remove
      heap={12,10,0}, max=12 → output [7,12]
    x=9: END h=10 → remove
      heap={12,0}, max=12 (no change)
    x=12: END h=12 → remove
      heap={0}, max=0 → output [12,0]

  Skyline: [[2,10],[3,15],[7,12],[12,0]]
```

---

## Example 3: Minimum Interval to Include Each Query

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type Item struct {
	size, right int
}
type PQ []Item
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].size < h[j].size }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minInterval(intervals [][]int, queries []int) []int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	// Sort queries but keep original index
	qi := make([]int, len(queries))
	for i := range qi { qi[i] = i }
	sort.Slice(qi, func(a, b int) bool {
		return queries[qi[a]] < queries[qi[b]]
	})

	result := make([]int, len(queries))
	for i := range result { result[i] = -1 }

	h := &PQ{}
	j := 0

	for _, idx := range qi {
		q := queries[idx]

		// Add all intervals that start ≤ q
		for j < len(intervals) && intervals[j][0] <= q {
			size := intervals[j][1] - intervals[j][0] + 1
			heap.Push(h, Item{size, intervals[j][1]})
			j++
		}

		// Remove intervals that ended before q (lazy deletion)
		for h.Len() > 0 && (*h)[0].right < q {
			heap.Pop(h)
		}

		if h.Len() > 0 {
			result[idx] = (*h)[0].size
		}
	}
	return result
}

func main() {
	intervals := [][]int{{1, 4}, {2, 4}, {3, 6}, {4, 4}}
	queries := []int{2, 3, 4, 5}
	fmt.Println(minInterval(intervals, queries))
	// [3, 3, 1, 4]
}
```

**Textual Figure:**

```
Minimum Interval to Include Each Query
Intervals: [[1,4],[2,4],[3,6],[4,4]]
Queries:   [2, 3, 4, 5]

  0    1    2    3    4    5    6
  ├────┼────┼────┼────┼────┼────┤
  [1,4] ├────────────┤   size=4
  [2,4]      ├────────┤   size=3
  [3,6]           ├────────┤ size=4
  [4,4]                ┤   size=1

  Sort queries: [2,3,4,5], sort intervals by start

  q=2: add intervals starting ≤2: [1,4](sz4), [2,4](sz3)
       heap: {(3,[2,4]),(4,[1,4])}
       smallest size=3 → answer[0]=3

  q=3: add [3,6](sz4)
       heap: {(3,[2,4]),(4,[1,4]),(4,[3,6])}
       smallest=3 → answer[1]=3

  q=4: add [4,4](sz1)
       heap: {(1,[4,4]),(3,[2,4]),...}
       smallest=1 → answer[2]=1

  q=5: lazy-delete expired ([2,4],[1,4],[4,4] all end<5)
       heap after deletion: {(4,[3,6])}
       smallest=4 → answer[3]=4

  Result: [3, 3, 1, 4]
```

---

## Example 4: Maximum CPU Load

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type Job struct {
	start, end, load int
}

type EndHeap []Job
func (h EndHeap) Len() int           { return len(h) }
func (h EndHeap) Less(i, j int) bool { return h[i].end < h[j].end }
func (h EndHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *EndHeap) Push(x interface{}) { *h = append(*h, x.(Job)) }
func (h *EndHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func maxCPULoad(jobs []Job) int {
	sort.Slice(jobs, func(i, j int) bool {
		return jobs[i].start < jobs[j].start
	})

	h := &EndHeap{}
	currentLoad, maxLoad := 0, 0

	for _, job := range jobs {
		// Remove jobs that ended before current starts
		for h.Len() > 0 && (*h)[0].end <= job.start {
			removed := heap.Pop(h).(Job)
			currentLoad -= removed.load
		}
		heap.Push(h, job)
		currentLoad += job.load
		if currentLoad > maxLoad {
			maxLoad = currentLoad
		}
	}
	return maxLoad
}

func main() {
	jobs := []Job{
		{1, 4, 3}, {2, 5, 4}, {7, 9, 6},
	}
	fmt.Println("Max CPU load:", maxCPULoad(jobs)) // 7 (3+4)
}
```

**Textual Figure:**

```
Maximum CPU Load with Heap: [{1,4,3},{2,5,4},{7,9,6}]

  Timeline:
  0    1    2    3    4    5    6    7    8    9
  ├────┼────┼────┼────┼────┼────┼────┼────┼────┤
  J1:  ├──load=3──────┤
  J2:       ├──load=4────────┤
  J3:                              ├──load=6────┤

  Min-heap of (end, load), sweep sorted by start:
    Process J1 (start=1): heap empty
      push (end=4, load=3), currentLoad=3
    Process J2 (start=2): top.end=4 > 2 (no removal)
      push (end=5, load=4), currentLoad=3+4=7 ← MAX
    Process J3 (start=7): top.end=4 ≤ 7 → pop (load-=3)
                          top.end=5 ≤ 7 → pop (load-=4)
      currentLoad=0, push (end=9, load=6)
      currentLoad=6

  Load graph:
  7 │   ┌─────┐
  6 │   │     │                     ┌─────┐
  3 │┌──┤     │                     │     │
  0 │└──┴─────┴─────────────────┴─────┘
    1    2     4  5               7       9

  Answer: max CPU load = 7
```

---

## Example 5: K Closest Points Using Sweep

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type Event struct {
	dist  int
	pIdx  int
	start bool
}

type MaxH []int
func (h MaxH) Len() int           { return len(h) }
func (h MaxH) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func processIntervalsByEnd(intervals [][]int, k int) []int {
	// Sort by end, use min-heap to track ends
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	type EndItem struct {
		end, idx int
	}
	type EndHeap []EndItem
	// Simple approach: process in start order, maintain k smallest end times
	h := &MaxH{}

	result := []int{}
	for i, iv := range intervals {
		heap.Push(h, iv[1])
		if h.Len() > k {
			heap.Pop(h) // remove largest end
		}
		if h.Len() == k && i >= k-1 {
			result = append(result, (*h)[0]) // current k-th smallest end
		}
	}
	return result
}

func main() {
	intervals := [][]int{{1, 3}, {2, 5}, {4, 7}, {6, 8}, {5, 9}}
	fmt.Println("K=3 tracking:", processIntervalsByEnd(intervals, 3))
}
```

**Textual Figure:**

```
K Closest Points Using Sweep: intervals=[[1,3],[2,5],[4,7],[6,8],[5,9]], k=3

Sort by start: [[1,3],[2,5],[4,7],[5,9],[6,8]]

  0    2    4    6    8   10
  ├────┼────┼────┼────┼────┤
  [1,3] ├────┤       end=3
  [2,5]    ├──────┤    end=5
  [4,7]         ├──────┤  end=7
  [5,9]           ├────────┤ end=9
  [6,8]              ├────┤  end=8

  Max-heap tracking k=3 smallest end times:
    i=0: push 3         heap={3}
    i=1: push 5         heap={5,3}
    i=2: push 7         heap={7,5,3}  → k reached
         k-th smallest end = top = 7
    i=3: push 9, heap size>3 → pop 9
         heap={7,5,3}, top=7
    i=4: push 8, heap size>3 → pop 8
         heap={7,5,3}, top=7

  Result tracks k-th smallest end at each window
```

---

## Example 6: Merge K Sorted Interval Lists

```go
package main

import (
	"container/heap"
	"fmt"
)

type Entry struct {
	start, end int
	listIdx    int
	elemIdx    int
}
type EntryHeap []Entry
func (h EntryHeap) Len() int           { return len(h) }
func (h EntryHeap) Less(i, j int) bool { return h[i].start < h[j].start }
func (h EntryHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *EntryHeap) Push(x interface{}) { *h = append(*h, x.(Entry)) }
func (h *EntryHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func mergeKIntervalLists(lists [][][]int) [][]int {
	h := &EntryHeap{}
	for i, list := range lists {
		if len(list) > 0 {
			heap.Push(h, Entry{list[0][0], list[0][1], i, 0})
		}
	}

	merged := [][]int{}
	for h.Len() > 0 {
		e := heap.Pop(h).(Entry)

		// Merge with last if overlapping
		if len(merged) > 0 && e.start <= merged[len(merged)-1][1] {
			if e.end > merged[len(merged)-1][1] {
				merged[len(merged)-1][1] = e.end
			}
		} else {
			merged = append(merged, []int{e.start, e.end})
		}

		// Push next from same list
		nextIdx := e.elemIdx + 1
		if nextIdx < len(lists[e.listIdx]) {
			next := lists[e.listIdx][nextIdx]
			heap.Push(h, Entry{next[0], next[1], e.listIdx, nextIdx})
		}
	}
	return merged
}

func main() {
	lists := [][][]int{
		{{1, 3}, {6, 9}},
		{{2, 5}, {8, 12}},
		{{4, 7}, {11, 14}},
	}
	fmt.Println("Merged:", mergeKIntervalLists(lists))
	// [[1,14]] — all overlap transitively
}
```

**Textual Figure:**

```
Merge K Sorted Interval Lists
List 1: [[1,3],[6,9]]
List 2: [[2,5],[8,12]]
List 3: [[4,7],[11,14]]

  0    2    4    6    8   10   12   14
  ├────┼────┼────┼────┼────┼────┼────┤
  L1: ├──┤       ├────┤
  L2:    ├────┤       ├──────┤
  L3:         ├────┤        ├────┤

  Min-heap merge (sorted by start):
    heap: {[1,3,L1],[2,5,L2],[4,7,L3]}

    Pop [1,3]: merged=[[1,3]]     push [6,9] from L1
    Pop [2,5]: 2≤3 overlap → [1,5]  push [8,12] from L2
    Pop [4,7]: 4≤5 overlap → [1,7]  push [11,14] from L3
    Pop [6,9]: 6≤7 overlap → [1,9]
    Pop [8,12]: 8≤9 overlap → [1,12]
    Pop [11,14]: 11≤12 overlap → [1,14]

  Result:
  [1,14] ├──────────────────────────┤
  All intervals merge transitively into one!
```

---

## Example 7: IPO (Initial Public Offering)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type Project struct {
	profit, capital int
}

type ProfitHeap []int
func (h ProfitHeap) Len() int           { return len(h) }
func (h ProfitHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h ProfitHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *ProfitHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *ProfitHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func findMaximizedCapital(k, w int, profits, capital []int) int {
	projects := make([]Project, len(profits))
	for i := range profits {
		projects[i] = Project{profits[i], capital[i]}
	}

	// Sort by capital requirement
	sort.Slice(projects, func(i, j int) bool {
		return projects[i].capital < projects[j].capital
	})

	h := &ProfitHeap{}
	idx := 0

	for i := 0; i < k; i++ {
		// Add all affordable projects
		for idx < len(projects) && projects[idx].capital <= w {
			heap.Push(h, projects[idx].profit)
			idx++
		}
		if h.Len() == 0 { break }

		w += heap.Pop(h).(int)
	}
	return w
}

func main() {
	profits := []int{1, 2, 3}
	capital := []int{0, 1, 1}
	fmt.Println(findMaximizedCapital(2, 0, profits, capital)) // 4
}
```

**Textual Figure:**

```
IPO (Initial Public Offering)
profits=[1,2,3], capital=[0,1,1], k=2, w=0

Sort by capital: [{p=1,c=0},{p=2,c=1},{p=3,c=1}]

  Round 1 (w=0, pick best affordable project):
    Affordable: c≤0 → project {p=1,c=0}
    Max-heap of profits: {1}
    Pick top: profit=1, w = 0+1 = 1

  Round 2 (w=1, pick best affordable project):
    Affordable: c≤1 → projects {p=2,c=1},{p=3,c=1}
    Max-heap: {3, 2}
    Pick top: profit=3, w = 1+3 = 4

  Capital growth:
  w=0 ───[+1]───→ w=1 ───[+3]───→ w=4
       project 1       project 3

  Answer: final capital = 4
```

---

## Example 8: Sliding Window Maximum with Heap

```go
package main

import (
	"container/heap"
	"fmt"
)

type Pair struct {
	val, idx int
}
type PairHeap []Pair
func (h PairHeap) Len() int           { return len(h) }
func (h PairHeap) Less(i, j int) bool { return h[i].val > h[j].val }
func (h PairHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PairHeap) Push(x interface{}) { *h = append(*h, x.(Pair)) }
func (h *PairHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func maxSlidingWindow(nums []int, k int) []int {
	h := &PairHeap{}
	result := []int{}

	for i := 0; i < len(nums); i++ {
		heap.Push(h, Pair{nums[i], i})

		// Remove elements outside window (lazy deletion)
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
	nums := []int{1, 3, -1, -3, 5, 3, 6, 7}
	fmt.Println(maxSlidingWindow(nums, 3))
	// [3, 3, 5, 5, 6, 7]
}
```

**Textual Figure:**

```
Sliding Window Maximum with Heap: nums=[1,3,-1,-3,5,3,6,7], k=3

  Index:  0   1   2   3   4   5   6   7
  Value: [1] [3] [-1][-3] [5] [3] [6] [7]

  Window slides →
  ┌───────────┐
  │ 1  3  -1 │ -3  5  3  6  7   max-heap top: 3  (idx 1)
  └───────────┘
     ┌───────────┐
   1 │ 3  -1 -3 │ 5  3  6  7   top: 3  (idx 1)
     └───────────┘
        ┌───────────┐
   1  3 │-1  -3  5 │ 3  6  7   top: 5  (idx 4)
        └───────────┘
           ┌───────────┐
   1  3 -1 │-3   5  3 │ 6  7   top: 5  (idx 4)
           └───────────┘
              ┌──────────┐
              │ 5   3  6 │ 7   top: 6  (idx 6)
              └──────────┘
                 ┌─────────┐
                 │ 3  6  7 │   top: 7  (idx 7)
                 └─────────┘

  Lazy deletion: when heap top index < window start, pop it.
  Result: [3, 3, 5, 5, 6, 7]
```

---

## Example 9: Lazy Deletion Pattern

```go
package main

import (
	"container/heap"
	"fmt"
)

type Element struct {
	val, id int
}
type LazyHeap struct {
	data    []Element
	removed map[int]bool
}
func (h LazyHeap) Len() int           { return len(h.data) }
func (h LazyHeap) Less(i, j int) bool { return h.data[i].val > h.data[j].val }
func (h LazyHeap) Swap(i, j int)      { h.data[i], h.data[j] = h.data[j], h.data[i] }
func (h *LazyHeap) Push(x interface{}) { h.data = append(h.data, x.(Element)) }
func (h *LazyHeap) Pop() interface{} {
	old := h.data; n := len(old); x := old[n-1]; h.data = old[:n-1]; return x
}

func (h *LazyHeap) Add(val, id int) {
	heap.Push(h, Element{val, id})
}

func (h *LazyHeap) Remove(id int) {
	h.removed[id] = true
}

func (h *LazyHeap) Top() (int, bool) {
	for h.Len() > 0 && h.removed[h.data[0].id] {
		heap.Pop(h)
	}
	if h.Len() == 0 {
		return 0, false
	}
	return h.data[0].val, true
}

func main() {
	h := &LazyHeap{removed: map[int]bool{}}

	h.Add(10, 1)
	h.Add(20, 2)
	h.Add(15, 3)

	if v, ok := h.Top(); ok {
		fmt.Println("Top:", v) // 20
	}

	h.Remove(2) // lazily remove id=2 (val 20)

	if v, ok := h.Top(); ok {
		fmt.Println("Top after remove:", v) // 15
	}

	h.Add(25, 4)
	if v, ok := h.Top(); ok {
		fmt.Println("Top after add:", v) // 25
	}
}
```

**Textual Figure:**

```
Lazy Deletion Pattern

  Operation trace:

  Add(10,id=1), Add(20,id=2), Add(15,id=3)
  ┌─────────────────────┐
  │ Max-Heap:  20        │
  │          /    \       │
  │        10      15     │
  └─────────────────────┘
  Top() → 20 ✓

  Remove(id=2):  mark id=2 in removed set (lazy!)
  ┌─────────────────────┐
  │ Heap unchanged!      │    removed = {2}
  │ Still:  20            │
  │        /    \          │
  │      10      15        │
  └─────────────────────┘

  Top(): peek 20 (id=2) → id in removed! Pop it.
         peek 15 (id=3) → not removed → return 15 ✓

  Add(25,id=4):
  ┌─────────────────────┐
  │ Heap:   25            │
  │        /    \          │
  │      10      15        │
  └─────────────────────┘
  Top() → 25 (id=4, not removed) → return 25 ✓

  Key: actual removal is O(1), deferred to when element reaches top
```

---

## Example 10: When to Use Heap vs Other Structures

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Line Sweep: Choosing the Right Data Structure ===")
	fmt.Println()

	choices := []struct{ need, structure, complexity, example string }{
		{
			"Track max/min active",
			"Max-Heap / Min-Heap",
			"O(log n) per operation",
			"Meeting rooms, skyline, CPU load",
		},
		{
			"Track max with deletions",
			"Balanced BST (TreeMap) or Lazy Heap",
			"O(log n) per operation",
			"Skyline with proper deletion",
		},
		{
			"Count active elements",
			"Simple counter",
			"O(1) per operation",
			"Max overlap, car pooling",
		},
		{
			"Range updates on array",
			"Difference array + prefix sum",
			"O(1) update, O(n) query",
			"Flight bookings, range addition",
		},
		{
			"Track K-th element",
			"Two heaps or order-statistic tree",
			"O(log n) per operation",
			"Median maintenance",
		},
		{
			"Complex range queries",
			"Segment tree",
			"O(log n) per operation",
			"Rectangle union area",
		},
	}

	for _, c := range choices {
		fmt.Printf("Need: %s\n", c.need)
		fmt.Printf("  Use: %s (%s)\n", c.structure, c.complexity)
		fmt.Printf("  Example: %s\n\n", c.example)
	}
}
```

**Textual Figure:**

```
When to Use Heap vs Other Structures

┌────────────────────┐
│  What do you need?  │
└──────┬──────┬──────┘
       │      │
  Max/Min?  Count?  K-th element?
       │      │         │
  ┌────┴──┐ ┌┴────┐ ┌─┴───────┐
  │ Heap  │ │Counter│ │Two Heaps  │
  │O(logn)│ │ O(1)  │ │  O(logn)  │
  └───────┘ └──────┘ └──────────┘
       │             │
   With deletions?  Range queries?
       │             │
  ┌────┴──────┐ ┌─┴──────────┐
  │ Lazy Heap  │ │Segment Tree │
  │ or BST     │ │  O(logn)    │
  └───────────┘ └────────────┘

  Summary:
  ┌───────────────────┬─────────────────┬───────────────────┐
  │ Need              │ Structure       │ Example             │
  ├───────────────────┼─────────────────┼───────────────────┤
  │ Track max/min     │ Heap            │ Meeting rooms       │
  │ Max w/ deletions  │ Lazy Heap / BST │ Skyline             │
  │ Count active      │ Counter         │ Max overlap         │
  │ Range updates     │ Diff array      │ Flight bookings     │
  │ K-th element      │ Two heaps       │ Median maintenance  │
  │ Complex ranges    │ Segment tree    │ Rectangle area      │
  └───────────────────┴─────────────────┴───────────────────┘
```

---
2. Max-heap tracks highest active element → skyline problem
3. Lazy deletion: mark as removed, pop when it reaches the top
4. Sort events by time, process with heap for O(n log n)
5. Heap + sweep line = powerful combo for interval scheduling problems

> **Next up:** Meeting Rooms Model →
