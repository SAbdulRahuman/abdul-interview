# Phase 11: Heap / Priority Queue — Two Heap Pattern

## Overview

The **two heap pattern** uses a **max heap** for the lower half and a **min heap** for the upper half to efficiently track the **median** or maintain a balanced partition.

```
Max Heap (lower half)    |    Min Heap (upper half)
  [1, 2, 3]             |    [4, 5, 6]
  top = 3 (max of lower) |   top = 4 (min of upper)
  
  Median = (3 + 4) / 2 = 3.5
```

**Invariant:** `maxHeap.Len() == minHeap.Len()` or `maxHeap.Len() == minHeap.Len() + 1`

---

## Example 1: Find Median from Data Stream (LeetCode 295)

```go
package main

import (
	"container/heap"
	"fmt"
)

type MaxHeap []int
func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type MedianFinder struct {
	lo *MaxHeap // lower half
	hi *MinHeap // upper half
}

func Constructor() MedianFinder {
	lo, hi := &MaxHeap{}, &MinHeap{}
	return MedianFinder{lo, hi}
}

func (mf *MedianFinder) AddNum(num int) {
	heap.Push(mf.lo, num)
	// Move max of lower to upper
	heap.Push(mf.hi, heap.Pop(mf.lo))
	// Balance: lower can have 1 more
	if mf.hi.Len() > mf.lo.Len() {
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
	mf := Constructor()
	mf.AddNum(1); fmt.Println(mf.FindMedian()) // 1.0
	mf.AddNum(2); fmt.Println(mf.FindMedian()) // 1.5
	mf.AddNum(3); fmt.Println(mf.FindMedian()) // 2.0
	mf.AddNum(4); fmt.Println(mf.FindMedian()) // 2.5
	mf.AddNum(5); fmt.Println(mf.FindMedian()) // 3.0
}
```

**Textual Figure:**

```
Find Median from Data Stream — Two Heap Approach

  MaxHeap (lower half) │ MinHeap (upper half)
  top = max of lower   │ top = min of upper

  ┌─────────┬────────────────┬────────────────┬─────────┐
  │  Add(n) │  MaxHeap (lo)  │  MinHeap (hi)  │ Median  │
  ├─────────┼────────────────┼────────────────┼─────────┤
  │    1    │ [1]            │ []             │  1.0    │
  │    2    │ [1]            │ [2]            │  1.5    │
  │    3    │ [2, 1]         │ [3]            │  2.0    │
  │    4    │ [2, 1]         │ [3, 4]         │  2.5    │
  │    5    │ [3, 2, 1]      │ [4, 5]         │  3.0    │
  └─────────┴────────────────┴────────────────┴─────────┘

  After Add(5):
    MaxHeap (lo)        MinHeap (hi)
       3                   4
      / \                 /
     2   1               5

  Median: lo.Len() > hi.Len() → return lo[0] = 3.0

  Invariant: |lo.Len() - hi.Len()| ≤ 1
  Odd count  → median = lo[0]
  Even count → median = (lo[0] + hi[0]) / 2
```

---

## Example 2: Sliding Window Median (LeetCode 480)

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

type MinH []Item
func (h MinH) Len() int           { return len(h) }
func (h MinH) Less(i, j int) bool { return h[i].val < h[j].val }
func (h MinH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MinH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func medianSlidingWindow(nums []int, k int) []float64 {
	lo, hi := &MaxH{}, &MinH{}
	delayed := map[int]int{} // lazy deletion counts by index
	loSize, hiSize := 0, 0

	addNum := func(val, idx int) {
		if lo.Len() == 0 || val <= (*lo)[0].val {
			heap.Push(lo, Item{val, idx}); loSize++
		} else {
			heap.Push(hi, Item{val, idx}); hiSize++
		}
	}

	balance := func() {
		for loSize > hiSize+1 {
			heap.Push(hi, heap.Pop(lo).(Item)); loSize--; hiSize++
		}
		for hiSize > loSize {
			heap.Push(lo, heap.Pop(hi).(Item)); hiSize--; loSize++
		}
	}

	prune := func() {
		for lo.Len() > 0 {
			if _, ok := delayed[(*lo)[0].idx]; ok {
				delayed[(*lo)[0].idx]--
				if delayed[(*lo)[0].idx] == 0 { delete(delayed, (*lo)[0].idx) }
				heap.Pop(lo)
			} else { break }
		}
		for hi.Len() > 0 {
			if _, ok := delayed[(*hi)[0].idx]; ok {
				delayed[(*hi)[0].idx]--
				if delayed[(*hi)[0].idx] == 0 { delete(delayed, (*hi)[0].idx) }
				heap.Pop(hi)
			} else { break }
		}
	}

	getMedian := func() float64 {
		if k%2 == 1 { return float64((*lo)[0].val) }
		return (float64((*lo)[0].val) + float64((*hi)[0].val)) / 2.0
	}

	var result []float64
	for i := 0; i < len(nums); i++ {
		addNum(nums[i], i)
		balance()
		prune()
		if i >= k {
			outIdx := i - k
			delayed[outIdx]++
			if nums[outIdx] <= (*lo)[0].val { loSize-- } else { hiSize-- }
			balance()
			prune()
		}
		if i >= k-1 {
			result = append(result, getMedian())
		}
	}
	return result
}

func main() {
	fmt.Println(medianSlidingWindow([]int{1, 3, -1, -3, 5, 3, 6, 7}, 3))
	// [1.0 -1.0 -1.0 3.0 5.0 6.0]
}
```

**Textual Figure:**

```
Sliding Window Median — k=3
Input: [1, 3, -1, -3, 5, 3, 6, 7]

  Two heaps with lazy deletion (by index):
  ┌──────┬─────────────┬──────────────┬─────────────┬─────────┐
  │ Win  │   Window    │  MaxH (lo)   │ MinH (hi)   │ Median  │
  ├──────┼─────────────┼──────────────┼─────────────┼─────────┤
  │ 0-2  │ [1, 3, -1]  │ [1, -1]      │ [3]         │  1.0    │
  │ 1-3  │ [3, -1, -3] │ [-1, -3]     │ [3]         │ -1.0    │
  │ 2-4  │ [-1, -3, 5] │ [-1, -3]     │ [5]         │ -1.0    │
  │ 3-5  │ [-3, 5, 3]  │ [3, -3]      │ [5]         │  3.0    │
  │ 4-6  │ [5, 3, 6]   │ [5, 3]       │ [6]         │  5.0    │
  │ 5-7  │ [3, 6, 7]   │ [6, 3]       │ [7]         │  6.0    │
  └──────┴─────────────┴──────────────┴─────────────┴─────────┘

  Lazy deletion: expired indices marked in map, pruned on access
  k=3 (odd) → median = lo[0] (max of lower half)
  Result: [1.0, -1.0, -1.0, 3.0, 5.0, 6.0] ✓
```

---

## Example 3: IPO — Maximize Capital (LeetCode 502)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type MaxProfitHeap []int
func (h MaxProfitHeap) Len() int           { return len(h) }
func (h MaxProfitHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxProfitHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxProfitHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxProfitHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func findMaximizedCapital(k int, w int, profits []int, capital []int) int {
	n := len(profits)
	projects := make([][2]int, n)
	for i := range projects {
		projects[i] = [2]int{capital[i], profits[i]}
	}
	sort.Slice(projects, func(i, j int) bool {
		return projects[i][0] < projects[j][0]
	})

	h := &MaxProfitHeap{}
	idx := 0
	for i := 0; i < k; i++ {
		for idx < n && projects[idx][0] <= w {
			heap.Push(h, projects[idx][1])
			idx++
		}
		if h.Len() == 0 { break }
		w += heap.Pop(h).(int)
	}
	return w
}

func main() {
	fmt.Println(findMaximizedCapital(2, 0, []int{1, 2, 3}, []int{0, 1, 1})) // 4
	fmt.Println(findMaximizedCapital(3, 0, []int{1, 2, 3}, []int{0, 1, 2})) // 6
}
```

**Textual Figure:**

```
IPO — Maximize Capital
k=2, w=0, profits=[1,2,3], capital=[0,1,1]

  Projects sorted by capital:
  ┌─────────┬─────────┬────────┐
  │ Project │ Capital │ Profit │
  ├─────────┼─────────┼────────┤
  │   A     │    0    │   1    │
  │   B     │    1    │   2    │
  │   C     │    1    │   3    │
  └─────────┴─────────┴────────┘

  Greedy rounds (pick max profit from affordable):
  ┌───────┬──────┬────────────────────┬───────────────┬──────┐
  │ Round │  w   │ Affordable → heap  │ Pick (max)    │  w'  │
  ├───────┼──────┼────────────────────┼───────────────┼──────┤
  │   1   │  0   │ A(cap=0≤0) → [1]   │ Pop 1         │  1   │
  │   2   │  1   │ B,C(cap≤1)→ [2,3] │ Pop 3         │  4   │
  └───────┴──────┴────────────────────┴───────────────┴──────┘

  Result: w = 4 ✓
  Pattern: sort by capital → max profit heap of affordable projects
```

---

## Example 4: Balance Two Heaps — Generic Template

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

type TwoHeap struct {
	lo *MaxH
	hi *MinH
}

func NewTwoHeap() *TwoHeap {
	return &TwoHeap{lo: &MaxH{}, hi: &MinH{}}
}

func (th *TwoHeap) Add(val int) {
	if th.lo.Len() == 0 || val <= (*th.lo)[0] {
		heap.Push(th.lo, val)
	} else {
		heap.Push(th.hi, val)
	}
	th.balance()
}

func (th *TwoHeap) balance() {
	for th.lo.Len() > th.hi.Len()+1 {
		heap.Push(th.hi, heap.Pop(th.lo))
	}
	for th.hi.Len() > th.lo.Len() {
		heap.Push(th.lo, heap.Pop(th.hi))
	}
}

func (th *TwoHeap) Median() float64 {
	if th.lo.Len() > th.hi.Len() {
		return float64((*th.lo)[0])
	}
	return float64((*th.lo)[0]+(*th.hi)[0]) / 2.0
}

func main() {
	th := NewTwoHeap()
	for _, v := range []int{5, 2, 8, 1, 9, 3} {
		th.Add(v)
		fmt.Printf("Add %d → median = %.1f\n", v, th.Median())
	}
}
```

**Textual Figure:**

```
Balanced Two Heaps — Generic Template
Input stream: [5, 2, 8, 1, 9, 3]

  ┌──────┬────────────────────────────────────────────┬────────┐
  │ Add  │ MaxH (lo)         │ MinH (hi)        │ Median │
  ├──────┼─────────────────────┼─────────────────────┼────────┤
  │  5   │ [5]               │ []                │  5.0   │
  │  2   │ [2]               │ [5]               │  3.5   │
  │  8   │ [5, 2]            │ [8]               │  5.0   │
  │  1   │ [2, 1]            │ [5, 8]            │  3.5   │
  │  9   │ [5, 2, 1]         │ [8, 9]            │  5.0   │
  │  3   │ [3, 2, 1]         │ [5, 8, 9]         │  4.0   │
  └──────┴─────────────────────┴─────────────────────┴────────┘

  Final state:
    MaxH (lo)         MinH (hi)
       3                 5
      / \               / \
     2   1             8   9

  Sorted: [1, 2, 3 | 5, 8, 9]   median = (3+5)/2 = 4.0
```

---

## Example 5: Percentile Tracking

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

// Track P90: 90th percentile
type PercentileTracker struct {
	lo    *MaxH  // bottom 90%
	hi    *MinH  // top 10%
	pct   float64
	total int
}

func NewPercentileTracker(pct float64) *PercentileTracker {
	return &PercentileTracker{lo: &MaxH{}, hi: &MinH{}, pct: pct}
}

func (pt *PercentileTracker) Add(val int) {
	pt.total++
	if pt.lo.Len() == 0 || val <= (*pt.lo)[0] {
		heap.Push(pt.lo, val)
	} else {
		heap.Push(pt.hi, val)
	}
	// Rebalance: lo should have ceil(total * pct) elements
	target := int(float64(pt.total)*pt.pct + 0.5)
	for pt.lo.Len() > target {
		heap.Push(pt.hi, heap.Pop(pt.lo))
	}
	for pt.lo.Len() < target && pt.hi.Len() > 0 {
		heap.Push(pt.lo, heap.Pop(pt.hi))
	}
}

func (pt *PercentileTracker) Percentile() int {
	return (*pt.lo)[0]
}

func main() {
	pt := NewPercentileTracker(0.5) // median
	for _, v := range []int{10, 20, 30, 40, 50, 60, 70, 80, 90, 100} {
		pt.Add(v)
		fmt.Printf("Add %3d → P50 = %d\n", v, pt.Percentile())
	}
}
```

**Textual Figure:**

```
Percentile Tracking — P50 (Median) via Two Heaps
Input: [10, 20, 30, 40, 50, 60, 70, 80, 90, 100]

  MaxH (lo) = bottom pct      MinH (hi) = top (1-pct)
  target lo size = ceil(total * 0.5)

  ┌──────┬───────┬────────────────────┬────────────────────┬─────┐
  │  Add │ Total │ MaxH (lo)          │ MinH (hi)          │ P50 │
  ├──────┼───────┼────────────────────┼────────────────────┼─────┤
  │  10  │   1   │ [10]               │ []                 │  10 │
  │  20  │   2   │ [10]               │ [20]               │  10 │
  │  30  │   3   │ [20, 10]           │ [30]               │  20 │
  │  40  │   4   │ [20, 10]           │ [30, 40]           │  20 │
  │  50  │   5   │ [30, 20, 10]       │ [40, 50]           │  30 │
  │  60  │   6   │ [30, 20, 10]       │ [40, 50, 60]       │  30 │
  │  70  │   7   │ [40, 30, 20, 10]   │ [50, 60, 70]       │  40 │
  │  80  │   8   │ [40, 30, 20, 10]   │ [50, 60, 70, 80]   │  40 │
  │  90  │   9   │ [50, 40, .., 10]   │ [60, 70, 80, 90]   │  50 │
  │ 100  │  10   │ [50, 40, .., 10]   │ [60, .., 100]      │  50 │
  └──────┴───────┴────────────────────┴────────────────────┴─────┘

  P50 = lo[0] = top of max heap (max of lower half)
  Generalizes to any percentile by adjusting target size.
```

---

## Example 6: Minimize Deviation in Array (LeetCode 1675)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type MaxH []int
func (h MaxH) Len() int           { return len(h) }
func (h MaxH) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minimumDeviation(nums []int) int {
	h := &MaxH{}
	minVal := math.MaxInt32

	for _, n := range nums {
		if n%2 == 1 { n *= 2 } // make all even
		heap.Push(h, n)
		if n < minVal { minVal = n }
	}

	result := (*h)[0] - minVal
	for (*h)[0]%2 == 0 {
		top := heap.Pop(h).(int)
		top /= 2
		if top < minVal { minVal = top }
		heap.Push(h, top)
		dev := (*h)[0] - minVal
		if dev < result { result = dev }
	}
	return result
}

func main() {
	fmt.Println(minimumDeviation([]int{1, 2, 3, 4}))    // 1
	fmt.Println(minimumDeviation([]int{4, 1, 5, 20, 3})) // 3
	fmt.Println(minimumDeviation([]int{2, 10, 8}))        // 3
}
```

**Textual Figure:**

```
Minimize Deviation — Max Heap Approach
Input: [1, 2, 3, 4]

  Step 1 — Normalize: make all even (odd ×2)
  [1×2=2, 2, 3×2=6, 4]  →  [2, 2, 6, 4]
  Track minVal = 2

  Build max heap: [6, 4, 2, 2]    deviation = 6-2 = 4

  Step 2 — Shrink max while even:
  ┌──────┬──────┬────────────────────┬───────┬───────┐
  │ Iter │ Top  │ Action             │ Min   │ Dev   │
  ├──────┼──────┼────────────────────┼───────┼───────┤
  │  0   │  6   │ 6/2=3 → push 3    │   2   │ 4-2=2 │
  │  1   │  4   │ 4/2=2 → push 2    │   2   │ 3-2=1 │
  │  2   │  3   │ 3 is odd → STOP   │   2   │ 3-2=1 │
  └──────┴──────┴────────────────────┴───────┴───────┘

  Result: min deviation = 1 ✓
  Key: can only halve evens, so stop when top is odd.
```

---

## Example 7: Minimum Cost to Hire K Workers (LeetCode 857)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
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

func mincostToHireWorkers(quality []int, wage []int, k int) float64 {
	n := len(quality)
	workers := make([][2]float64, n)
	for i := range workers {
		workers[i] = [2]float64{float64(wage[i]) / float64(quality[i]), float64(quality[i])}
	}
	sort.Slice(workers, func(i, j int) bool {
		return workers[i][0] < workers[j][0]
	})

	h := &MaxH{} // max heap of quality
	sumQ := 0
	result := math.MaxFloat64

	for _, w := range workers {
		ratio, q := w[0], int(w[1])
		heap.Push(h, q)
		sumQ += q
		if h.Len() > k {
			sumQ -= heap.Pop(h).(int)
		}
		if h.Len() == k {
			cost := ratio * float64(sumQ)
			if cost < result { result = cost }
		}
	}
	return result
}

func main() {
	fmt.Printf("%.5f\n", mincostToHireWorkers([]int{10, 20, 5}, []int{70, 50, 30}, 2))
	// 105.00000
	fmt.Printf("%.5f\n", mincostToHireWorkers([]int{3, 1, 10, 10, 1}, []int{4, 8, 2, 2, 7}, 3))
	// 30.66667
}
```

**Textual Figure:**

```
Minimum Cost to Hire K Workers — k=2
quality=[10,20,5], wage=[70,50,30]

  Step 1 — Compute wage/quality ratio, sort ascending:
  ┌────────┬─────────┬──────┬─────────┐
  │ Worker │ Quality │ Wage │  Ratio  │
  ├────────┼─────────┼──────┼─────────┤
  │   B    │   20    │  50  │  2.50   │
  │   C    │    5    │  30  │  6.00   │
  │   A    │   10    │  70  │  7.00   │
  └────────┴─────────┴──────┴─────────┘

  Step 2 — Process by ratio, max heap of quality:
  ┌────────┬───────┬───────────────┬──────┬────────────────┐
  │ Worker │ Ratio │ Heap (qual)   │ sumQ │ cost=r*sumQ    │
  ├────────┼───────┼───────────────┼──────┼────────────────┤
  │   B    │ 2.50  │ [20]          │  20  │  (len<k)       │
  │   C    │ 6.00  │ [20, 5]       │  25  │  6.0*25=150.0  │
  │   A    │ 7.00  │ [10,5] pop 20 │  15  │  7.0*15=105.0✓│
  └────────┴───────┴───────────────┴──────┴────────────────┘

  Result: min cost = 105.00000 ✓
```

---

## Example 8: Maximum Profit in Job Scheduling Variant

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type MinEndHeap [][2]int // [endTime, profit]
func (h MinEndHeap) Len() int           { return len(h) }
func (h MinEndHeap) Less(i, j int) bool { return h[i][0] < h[j][0] }
func (h MinEndHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinEndHeap) Push(x interface{}) { *h = append(*h, x.([2]int)) }
func (h *MinEndHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func jobScheduling(startTime, endTime, profit []int) int {
	n := len(startTime)
	jobs := make([][3]int, n)
	for i := range jobs {
		jobs[i] = [3]int{startTime[i], endTime[i], profit[i]}
	}
	sort.Slice(jobs, func(i, j int) bool { return jobs[i][0] < jobs[j][0] })

	h := &MinEndHeap{}
	maxProfit := 0

	for _, job := range jobs {
		// Release jobs that end before current starts
		for h.Len() > 0 && (*h)[0][0] <= job[0] {
			p := heap.Pop(h).([2]int)
			if p[1] > maxProfit { maxProfit = p[1] }
		}
		heap.Push(h, [2]int{job[1], maxProfit + job[2]})
	}
	for h.Len() > 0 {
		p := heap.Pop(h).([2]int)
		if p[1] > maxProfit { maxProfit = p[1] }
	}
	return maxProfit
}

func main() {
	fmt.Println(jobScheduling([]int{1, 2, 3, 3}, []int{3, 4, 5, 6}, []int{50, 10, 40, 70})) // 120
}
```

**Textual Figure:**

```
Job Scheduling with Heap — Maximize Profit
start=[1,2,3,3], end=[3,4,5,6], profit=[50,10,40,70]

  Jobs sorted by start time:
  ┌─────┬───────┬─────┬────────┐
  │ Job │ Start │ End │ Profit │
  ├─────┼───────┼─────┼────────┤
  │  A  │   1   │  3  │   50   │
  │  B  │   2   │  4  │   10   │
  │  C  │   3   │  5  │   40   │
  │  D  │   3   │  6  │   70   │
  └─────┴───────┴─────┴────────┘

  Timeline:  1   2   3   4   5   6
             A[50]===█
                 B[10]===█
                     C[40]===█
                     D[70]========█

  Process (min heap by end time):
  ┌─────┬────────────────────────┬────────────────────┬───────────┐
  │ Job │ Release ended jobs   │ Push (end, prof)   │ maxProfit │
  ├─────┼────────────────────────┼────────────────────┼───────────┤
  │  A  │ none                 │ (3, 0+50=50)       │    0      │
  │  B  │ none                 │ (4, 0+10=10)       │    0      │
  │  C  │ pop(3,50)→maxP=50   │ (5, 50+40=90)      │   50      │
  │  D  │ (already popped)     │ (6, 50+70=120)     │   50      │
  └─────┴────────────────────────┴────────────────────┴───────────┘

  Drain heap: pop (4,10)→50, pop (5,90)→90, pop (6,120)→120
  Result: maxProfit = 120 ✓
```

---

## Example 9: Two Heaps for Balanced Partition

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
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

// Find minimum absolute difference between two group sums
func minDiffPartition(nums []int) int {
	total := 0
	for _, n := range nums { total += n }

	lo, hi := &MaxH{}, &MinH{}
	loSum, hiSum := 0, 0

	for _, n := range nums {
		// Add to smaller sum side
		if loSum <= hiSum {
			heap.Push(lo, n)
			loSum += n
		} else {
			heap.Push(hi, n)
			hiSum += n
		}
	}

	diff := loSum - hiSum
	if diff < 0 { diff = -diff }
	fmt.Printf("Lower sum=%d, Upper sum=%d\n", loSum, hiSum)
	return diff
}

func main() {
	_ = math.Abs
	fmt.Println("Diff:", minDiffPartition([]int{1, 5, 11, 5})) // approximate
}
```

**Textual Figure:**

```
Two Heaps for Balanced Partition
Input: [1, 5, 11, 5]

  Greedy: assign each element to the smaller-sum side.
  ┌───────┬───────────────────┬─────────┬─────────┬───────────┐
  │ Elem  │ Assign to         │ loSum   │ hiSum   │ |lo-hi|   │
  ├───────┼───────────────────┼─────────┼─────────┼───────────┤
  │   1   │ lo (loSum≤hiSum)  │    1    │    0    │     1     │
  │   5   │ hi (loSum>hiSum)  │    1    │    5    │     4     │
  │  11   │ lo (loSum≤hiSum)  │   12    │    5    │     7     │
  │   5   │ hi (loSum>hiSum)  │   12    │   10    │     2     │
  └───────┴───────────────────┴─────────┴─────────┴───────────┘

  Partition:  lo = {1, 11} sum=12    hi = {5, 5} sum=10
  Diff = |12 - 10| = 2

  Note: greedy with heaps is approximate — not always optimal.
  Exact solution needs DP (subset sum). Heap approach is O(n log n).
```

---

## Example 10: When to Use Two Heaps

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Two Heap Pattern Summary ===")
	fmt.Println()

	cases := []struct {
		problem string
		maxHeap string
		minHeap string
		key     string
	}{
		{"Find Median", "Lower half", "Upper half", "Balance sizes"},
		{"Sliding Median", "Lower half + lazy del", "Upper half + lazy del", "Track by index"},
		{"IPO", "Available profits", "—", "Sort by capital"},
		{"Min Deviation", "All values (normalized)", "Track min", "Shrink max"},
		{"Hire Workers", "Top-k quality", "—", "Sort by wage/quality ratio"},
	}

	for _, c := range cases {
		fmt.Printf("%-20s max=%-25s min=%-25s (%s)\n",
			c.problem, c.maxHeap, c.minHeap, c.key)
	}

	fmt.Println()
	fmt.Println("Core insight: Two heaps let you track the boundary between")
	fmt.Println("two halves of a dynamically changing dataset in O(log n).")
}
```

**Textual Figure:**

```
Two Heap Pattern — When to Use Summary

  ┌────────────────────────────────────────────────────────┐
  │   Data stream: 1  5  2  8  3  7  4  9  6 ...  │
  │                                                        │
  │        MaxHeap (lo)     │     MinHeap (hi)      │
  │   ┌───────────────────┼────────────────────┐  │
  │   │  lower half       │  upper half          │  │
  │   │  [1, 2, 3, 4]     │  [5, 6, 7, 8, 9]     │  │
  │   │  top=4 ─────────┼──────── top=5       │  │
  │   └───────────────────┴────────────────────┘  │
  │            median = (4+5)/2 = 4.5              │
  └────────────────────────────────────────────────────────┘

  Pattern applies to:
  ┌────────────────────┬──────────────────────────────────┐
  │ Problem            │ Key Insight                      │
  ├────────────────────┼──────────────────────────────────┤
  │ Streaming median   │ Balance sizes ±1               │
  │ Sliding median     │ + lazy deletion by index       │
  │ Percentile         │ Adjust target lo size          │
  │ Balanced partition │ Assign to smaller-sum side     │
  │ IPO / scheduling   │ Max profit heap + sort by cap  │
  └────────────────────┴──────────────────────────────────┘
```

---

## Key Takeaways

1. Two heap pattern: max heap (lower) + min heap (upper) split at the median
2. **Balance invariant**: `|loSize - hiSize| ≤ 1`
3. Add to appropriate side, then rebalance — O(log n) per operation
4. Sliding window variant needs lazy deletion (mark + skip expired)
5. Not just median — any problem needing a balanced partition of streaming data

> **Next up:** Lazy Deletion →
