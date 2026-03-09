# Phase 17: Greedy Algorithms — Activity Selection

## Overview

The **Activity Selection Problem** is the classic example of greedy algorithms: given activities with start and finish times, select the maximum number of non-overlapping activities. Greedy strategy: always pick the activity that finishes earliest.

| Aspect | Detail |
|--------|--------|
| **Strategy** | Sort by finish time, pick earliest finishing |
| **Time** | O(n log n) for sorting |
| **Proof** | Greedy stays ahead (earliest finish leaves most room) |

---

## Example 1: Classic Activity Selection

```go
package main

import (
	"fmt"
	"sort"
)

type Activity struct {
	Start, Finish int
}

func activitySelection(activities []Activity) []Activity {
	// Sort by finish time
	sort.Slice(activities, func(i, j int) bool {
		return activities[i].Finish < activities[j].Finish
	})

	selected := []Activity{activities[0]}
	lastFinish := activities[0].Finish

	for i := 1; i < len(activities); i++ {
		if activities[i].Start >= lastFinish {
			selected = append(selected, activities[i])
			lastFinish = activities[i].Finish
		}
	}
	return selected
}

func main() {
	activities := []Activity{
		{1, 4}, {3, 5}, {0, 6}, {5, 7}, {3, 9}, {5, 9},
		{6, 10}, {8, 11}, {8, 12}, {2, 14}, {12, 16},
	}
	result := activitySelection(activities)
	fmt.Println("Max activities:", len(result))
	for _, a := range result {
		fmt.Printf("  [%d, %d)\n", a.Start, a.Finish)
	}
}
```

---

## Example 2: Recursive Activity Selection

```go
package main

import (
	"fmt"
	"sort"
)

type Activity struct {
	Start, Finish int
}

func selectActivities(activities []Activity, k int, n int) []Activity {
	// Find first activity that starts after activity k finishes
	m := k + 1
	for m < n && activities[m].Start < activities[k].Finish {
		m++
	}

	if m < n {
		rest := selectActivities(activities, m, n)
		return append([]Activity{activities[m]}, rest...)
	}
	return nil
}

func main() {
	acts := []Activity{
		{1, 4}, {3, 5}, {0, 6}, {5, 7}, {3, 9},
		{5, 9}, {6, 10}, {8, 11}, {8, 12}, {2, 14}, {12, 16},
	}
	sort.Slice(acts, func(i, j int) bool {
		return acts[i].Finish < acts[j].Finish
	})

	// Start with a dummy activity that finishes at 0
	dummy := Activity{-1, 0}
	acts = append([]Activity{dummy}, acts...)
	result := selectActivities(acts, 0, len(acts))

	fmt.Println("Selected:", len(result))
	for _, a := range result {
		fmt.Printf("  [%d, %d)\n", a.Start, a.Finish)
	}
}
```

---

## Example 3: Weighted Activity Selection (DP needed)

```go
package main

import (
	"fmt"
	"sort"
)

type Activity struct {
	Start, Finish, Weight int
}

func weightedActivitySelection(acts []Activity) int {
	sort.Slice(acts, func(i, j int) bool {
		return acts[i].Finish < acts[j].Finish
	})

	n := len(acts)
	dp := make([]int, n)
	dp[0] = acts[0].Weight

	for i := 1; i < n; i++ {
		// Include current activity
		incl := acts[i].Weight
		// Find latest non-conflicting activity (binary search)
		lo, hi := 0, i-1
		latest := -1
		for lo <= hi {
			mid := (lo + hi) / 2
			if acts[mid].Finish <= acts[i].Start {
				latest = mid
				lo = mid + 1
			} else {
				hi = mid - 1
			}
		}
		if latest != -1 {
			incl += dp[latest]
		}

		// Exclude current
		dp[i] = dp[i-1]
		if incl > dp[i] {
			dp[i] = incl
		}
	}
	return dp[n-1]
}

func main() {
	acts := []Activity{
		{1, 3, 5}, {2, 5, 6}, {4, 6, 5}, {6, 7, 4},
		{5, 8, 11}, {7, 9, 2},
	}
	fmt.Println("Max weight:", weightedActivitySelection(acts)) // 17

	fmt.Println()
	fmt.Println("Note: Weighted version needs DP, not pure greedy")
	fmt.Println("Pure greedy only works for unweighted (max count)")
}
```

---

## Example 4: Meeting Rooms — Can Attend All? (LC 252)

```go
package main

import (
	"fmt"
	"sort"
)

func canAttendAll(intervals [][]int) bool {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] < intervals[i-1][1] {
			return false // overlap
		}
	}
	return true
}

func main() {
	fmt.Println(canAttendAll([][]int{{0, 30}, {5, 10}, {15, 20}}))  // false
	fmt.Println(canAttendAll([][]int{{7, 10}, {2, 4}}))              // true
}
```

---

## Example 5: Maximum Number of Events (LC 1353)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func maxEvents(events [][]int) int {
	sort.Slice(events, func(i, j int) bool {
		return events[i][0] < events[j][0]
	})

	h := &IntHeap{}
	count, i := 0, 0
	n := len(events)

	for day := 1; day <= 100000; day++ {
		// Add all events starting today
		for i < n && events[i][0] == day {
			heap.Push(h, events[i][1])
			i++
		}
		// Remove expired events
		for h.Len() > 0 && (*h)[0] < day {
			heap.Pop(h)
		}
		// Attend event with earliest end
		if h.Len() > 0 {
			heap.Pop(h)
			count++
		}
		if i >= n && h.Len() == 0 { break }
	}
	return count
}

func main() {
	fmt.Println(maxEvents([][]int{{1, 2}, {2, 3}, {3, 4}}))           // 3
	fmt.Println(maxEvents([][]int{{1, 2}, {2, 3}, {3, 4}, {1, 2}}))   // 4
}
```

---

## Example 6: Non-overlapping Intervals — Min Removals (LC 435)

```go
package main

import (
	"fmt"
	"sort"
)

func eraseOverlapIntervals(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})

	count := 0
	lastEnd := intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] < lastEnd {
			count++ // remove this interval
		} else {
			lastEnd = intervals[i][1]
		}
	}
	return count
}

func main() {
	fmt.Println(eraseOverlapIntervals([][]int{{1, 2}, {2, 3}, {3, 4}, {1, 3}})) // 1
	fmt.Println(eraseOverlapIntervals([][]int{{1, 2}, {1, 2}, {1, 2}}))          // 2

	fmt.Println()
	fmt.Println("Same as activity selection: max non-overlapping = n - removals")
}
```

---

## Example 7: Minimum Arrows to Burst Balloons (LC 452)

```go
package main

import (
	"fmt"
	"sort"
)

func findMinArrowShots(points [][]int) int {
	sort.Slice(points, func(i, j int) bool {
		return points[i][1] < points[j][1]
	})

	arrows := 1
	lastEnd := points[0][1]

	for i := 1; i < len(points); i++ {
		if points[i][0] > lastEnd {
			arrows++
			lastEnd = points[i][1]
		}
	}
	return arrows
}

func main() {
	fmt.Println(findMinArrowShots([][]int{{10, 16}, {2, 8}, {1, 6}, {7, 12}})) // 2
	fmt.Println(findMinArrowShots([][]int{{1, 2}, {3, 4}, {5, 6}, {7, 8}}))    // 4

	fmt.Println()
	fmt.Println("Same pattern: sort by end, count non-overlapping groups")
}
```

---

## Example 8: Job Sequencing with Deadlines

```go
package main

import (
	"fmt"
	"sort"
)

type Job struct {
	ID       string
	Deadline int
	Profit   int
}

func jobSequencing(jobs []Job) (int, []string) {
	// Sort by profit descending
	sort.Slice(jobs, func(i, j int) bool {
		return jobs[i].Profit > jobs[j].Profit
	})

	maxDeadline := 0
	for _, j := range jobs {
		if j.Deadline > maxDeadline { maxDeadline = j.Deadline }
	}

	slots := make([]string, maxDeadline+1)
	totalProfit := 0

	for _, j := range jobs {
		// Find latest available slot
		for t := j.Deadline; t >= 1; t-- {
			if slots[t] == "" {
				slots[t] = j.ID
				totalProfit += j.Profit
				break
			}
		}
	}

	scheduled := []string{}
	for _, s := range slots {
		if s != "" {
			scheduled = append(scheduled, s)
		}
	}
	return totalProfit, scheduled
}

func main() {
	jobs := []Job{
		{"a", 2, 100}, {"b", 1, 19}, {"c", 2, 27},
		{"d", 1, 25}, {"e", 3, 15},
	}
	profit, sched := jobSequencing(jobs)
	fmt.Println("Max profit:", profit)
	fmt.Println("Jobs:", sched)
}
```

---

## Example 9: Activity Selection with K Rooms

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minRooms(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	h := &IntHeap{}

	for _, interval := range intervals {
		if h.Len() > 0 && (*h)[0] <= interval[0] {
			heap.Pop(h) // reuse room
		}
		heap.Push(h, interval[1])
	}
	return h.Len()
}

func main() {
	intervals := [][]int{{0, 30}, {5, 10}, {15, 20}}
	fmt.Println("Min rooms:", minRooms(intervals)) // 2

	intervals2 := [][]int{{7, 10}, {2, 4}}
	fmt.Println("Min rooms:", minRooms(intervals2)) // 1
}
```

---

## Example 10: Activity Selection Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Activity Selection Patterns ===")
	fmt.Println()

	fmt.Println("Pattern 1: Max non-overlapping activities")
	fmt.Println("  Sort by: FINISH TIME")
	fmt.Println("  Greedy: pick earliest finish that doesn't overlap")
	fmt.Println("  Problems: Activity selection, Non-overlapping intervals")
	fmt.Println()

	fmt.Println("Pattern 2: Min rooms / resources")
	fmt.Println("  Sort by: START TIME")
	fmt.Println("  Use: Min-heap of end times")
	fmt.Println("  Problems: Meeting Rooms II, Task scheduler")
	fmt.Println()

	fmt.Println("Pattern 3: Min arrows / groups")
	fmt.Println("  Sort by: END TIME")
	fmt.Println("  Count: non-overlapping groups")
	fmt.Println("  Problems: Burst balloons, Min platforms")
	fmt.Println()

	fmt.Println("Pattern 4: Weighted selection (needs DP)")
	fmt.Println("  Sort by: FINISH TIME")
	fmt.Println("  DP: max(include + dp[last_compatible], dp[i-1])")
	fmt.Println("  Problems: Weighted job scheduling")
	fmt.Println()

	fmt.Println("Key insight: ALWAYS sort by finish time for")
	fmt.Println("non-overlapping selection problems.")
}
```

---

## Key Takeaways

1. Always sort by finish time for maximum non-overlapping selection
2. Greedy stays ahead: earliest finish leaves most room for future activities
3. Weighted variant needs DP — greedy alone is insufficient
4. Same pattern: min arrows, non-overlapping intervals, meeting rooms
5. Activity selection is the canonical example proving greedy choice property

> **Next up:** Interval Scheduling Maximization →
