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

---

## Key Takeaways

1. Two heap pattern: max heap (lower) + min heap (upper) split at the median
2. **Balance invariant**: `|loSize - hiSize| ≤ 1`
3. Add to appropriate side, then rebalance — O(log n) per operation
4. Sliding window variant needs lazy deletion (mark + skip expired)
5. Not just median — any problem needing a balanced partition of streaming data

> **Next up:** Lazy Deletion →
