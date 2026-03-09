# Phase 17: Greedy Algorithms — Interval Scheduling Maximization

## Overview

**Interval Scheduling Maximization** is the general framework for selecting the maximum number of compatible (non-overlapping) intervals. It extends activity selection to various scheduling problems.

| Variant | Strategy |
|---------|----------|
| **Max non-overlapping** | Sort by end time, greedy pick |
| **Min intervals to remove** | n − max non-overlapping |
| **Min resources** | Sort by start, min-heap of ends |
| **Min points to cover** | Sort by end, greedy scan |

---

## Example 1: Maximum Non-Overlapping Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func maxNonOverlapping(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})

	count := 1
	end := intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] >= end {
			count++
			end = intervals[i][1]
		}
	}
	return count
}

func main() {
	intervals := [][]int{{1, 3}, {2, 4}, {3, 5}, {0, 6}, {5, 7}, {8, 9}, {5, 9}}
	fmt.Println("Max non-overlapping:", maxNonOverlapping(intervals)) // 4
}
```

---

## Example 2: Minimum Removals for Non-Overlapping (LC 435)

```go
package main

import (
	"fmt"
	"sort"
)

func eraseOverlapIntervals(intervals [][]int) int {
	if len(intervals) == 0 { return 0 }

	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})

	kept := 1
	end := intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] >= end {
			kept++
			end = intervals[i][1]
		}
	}
	return len(intervals) - kept
}

func main() {
	fmt.Println(eraseOverlapIntervals([][]int{{1, 2}, {2, 3}, {3, 4}, {1, 3}})) // 1
	fmt.Println(eraseOverlapIntervals([][]int{{1, 2}, {1, 2}, {1, 2}}))          // 2
	fmt.Println(eraseOverlapIntervals([][]int{{1, 2}, {2, 3}}))                  // 0
}
```

---

## Example 3: Interval Scheduling with Weights (DP)

```go
package main

import (
	"fmt"
	"sort"
)

type Interval struct {
	Start, End, Weight int
}

func maxWeightSchedule(intervals []Interval) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i].End < intervals[j].End
	})

	n := len(intervals)
	dp := make([]int, n)
	dp[0] = intervals[0].Weight

	// Find latest compatible interval using binary search
	findLatest := func(idx int) int {
		lo, hi := 0, idx-1
		result := -1
		for lo <= hi {
			mid := (lo + hi) / 2
			if intervals[mid].End <= intervals[idx].Start {
				result = mid
				lo = mid + 1
			} else {
				hi = mid - 1
			}
		}
		return result
	}

	for i := 1; i < n; i++ {
		incl := intervals[i].Weight
		if j := findLatest(i); j != -1 {
			incl += dp[j]
		}
		dp[i] = dp[i-1]
		if incl > dp[i] {
			dp[i] = incl
		}
	}
	return dp[n-1]
}

func main() {
	intervals := []Interval{
		{1, 3, 50}, {2, 5, 20}, {4, 6, 30},
		{6, 7, 60}, {5, 8, 50}, {7, 9, 10},
	}
	fmt.Println("Max weight:", maxWeightSchedule(intervals)) // 140
}
```

---

## Example 4: Minimum Platforms / Meeting Rooms (LC 253)

```go
package main

import (
	"fmt"
	"sort"
)

func minPlatforms(arrivals, departures []int) int {
	sort.Ints(arrivals)
	sort.Ints(departures)

	platforms, maxPlat := 0, 0
	i, j := 0, 0

	for i < len(arrivals) {
		if arrivals[i] <= departures[j] {
			platforms++
			i++
		} else {
			platforms--
			j++
		}
		if platforms > maxPlat {
			maxPlat = platforms
		}
	}
	return maxPlat
}

func main() {
	arr := []int{900, 940, 950, 1100, 1500, 1800}
	dep := []int{910, 1200, 1120, 1130, 1900, 2000}
	fmt.Println("Min platforms:", minPlatforms(arr, dep)) // 3
}
```

---

## Example 5: Maximum CPU Load (Overlapping Intervals)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type Job struct {
	Start, End, Load int
}

type EndHeap []Job
func (h EndHeap) Len() int           { return len(h) }
func (h EndHeap) Less(i, j int) bool { return h[i].End < h[j].End }
func (h EndHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *EndHeap) Push(x interface{}) { *h = append(*h, x.(Job)) }
func (h *EndHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func maxCPULoad(jobs []Job) int {
	sort.Slice(jobs, func(i, j int) bool { return jobs[i].Start < jobs[j].Start })

	h := &EndHeap{}
	currentLoad, maxLoad := 0, 0

	for _, job := range jobs {
		for h.Len() > 0 && (*h)[0].End <= job.Start {
			removed := heap.Pop(h).(Job)
			currentLoad -= removed.Load
		}
		heap.Push(h, job)
		currentLoad += job.Load
		if currentLoad > maxLoad { maxLoad = currentLoad }
	}
	return maxLoad
}

func main() {
	jobs := []Job{{1, 4, 3}, {2, 5, 4}, {7, 9, 6}}
	fmt.Println("Max CPU load:", maxCPULoad(jobs)) // 7

	jobs2 := []Job{{6, 7, 10}, {2, 4, 11}, {8, 12, 15}}
	fmt.Println("Max CPU load:", maxCPULoad(jobs2)) // 15
}
```

---

## Example 6: Task Scheduler (LC 621)

```go
package main

import "fmt"

func leastInterval(tasks []byte, n int) int {
	freq := [26]int{}
	for _, t := range tasks {
		freq[t-'A']++
	}

	maxFreq := 0
	maxCount := 0
	for _, f := range freq {
		if f > maxFreq {
			maxFreq = f
			maxCount = 1
		} else if f == maxFreq {
			maxCount++
		}
	}

	// (maxFreq-1) full chunks of (n+1) + maxCount for last chunk
	result := (maxFreq-1)*(n+1) + maxCount
	if result < len(tasks) {
		result = len(tasks) // can't be less than total tasks
	}
	return result
}

func main() {
	tasks := []byte{'A', 'A', 'A', 'B', 'B', 'B'}
	fmt.Println(leastInterval(tasks, 2)) // 8: A B _ A B _ A B

	tasks2 := []byte{'A', 'A', 'A', 'B', 'B', 'B'}
	fmt.Println(leastInterval(tasks2, 0)) // 6
}
```

---

## Example 7: Merge Intervals and Count Max (Sweep Line)

```go
package main

import (
	"fmt"
	"sort"
)

func maxOverlap(intervals [][]int) int {
	events := [][2]int{}
	for _, iv := range intervals {
		events = append(events, [2]int{iv[0], 1})  // start
		events = append(events, [2]int{iv[1], -1}) // end
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i][0] != events[j][0] {
			return events[i][0] < events[j][0]
		}
		return events[i][1] < events[j][1] // end before start at same time
	})

	current, maxOvlp := 0, 0
	for _, e := range events {
		current += e[1]
		if current > maxOvlp { maxOvlp = current }
	}
	return maxOvlp
}

func main() {
	intervals := [][]int{{1, 5}, {2, 6}, {3, 7}, {4, 8}}
	fmt.Println("Max overlap:", maxOverlap(intervals)) // 4

	intervals2 := [][]int{{1, 3}, {5, 7}, {2, 4}}
	fmt.Println("Max overlap:", maxOverlap(intervals2)) // 2
}
```

---

## Example 8: Car Pooling (LC 1094)

```go
package main

import (
	"fmt"
	"sort"
)

func carPooling(trips [][]int, capacity int) bool {
	events := [][2]int{}
	for _, t := range trips {
		passengers, from, to := t[0], t[1], t[2]
		events = append(events, [2]int{from, passengers})
		events = append(events, [2]int{to, -passengers})
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i][0] != events[j][0] {
			return events[i][0] < events[j][0]
		}
		return events[i][1] < events[j][1]
	})

	current := 0
	for _, e := range events {
		current += e[1]
		if current > capacity {
			return false
		}
	}
	return true
}

func main() {
	fmt.Println(carPooling([][]int{{2, 1, 5}, {3, 3, 7}}, 4))  // false
	fmt.Println(carPooling([][]int{{2, 1, 5}, {3, 3, 7}}, 5))  // true
	fmt.Println(carPooling([][]int{{2, 1, 5}, {3, 5, 7}}, 3))  // true
}
```

---

## Example 9: Minimum Number of Arrows (LC 452)

```go
package main

import (
	"fmt"
	"sort"
)

func findMinArrowShots(points [][]int) int {
	if len(points) == 0 { return 0 }

	sort.Slice(points, func(i, j int) bool {
		return points[i][1] < points[j][1]
	})

	arrows := 1
	arrowPos := points[0][1]

	for i := 1; i < len(points); i++ {
		if points[i][0] > arrowPos {
			arrows++
			arrowPos = points[i][1]
		}
	}
	return arrows
}

func main() {
	pts := [][]int{{10, 16}, {2, 8}, {1, 6}, {7, 12}}
	fmt.Println(findMinArrowShots(pts)) // 2 — shoot at 6 and 12

	pts2 := [][]int{{1, 2}, {3, 4}, {5, 6}, {7, 8}}
	fmt.Println(findMinArrowShots(pts2)) // 4
}
```

---

## Example 10: Interval Scheduling Pattern Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Interval Scheduling Maximization Patterns ===")
	fmt.Println()

	fmt.Println("1. Max non-overlapping intervals:")
	fmt.Println("   Sort by END time → greedy pick → O(n log n)")
	fmt.Println()

	fmt.Println("2. Min removals for non-overlapping:")
	fmt.Println("   = n - maxNonOverlapping()")
	fmt.Println()

	fmt.Println("3. Min resources (rooms/platforms):")
	fmt.Println("   Sort by START → min-heap of end times")
	fmt.Println("   Or sweep line with events")
	fmt.Println()

	fmt.Println("4. Max overlap at any point:")
	fmt.Println("   Sweep line: +1 at start, -1 at end, track max")
	fmt.Println()

	fmt.Println("5. Min arrows/points to cover all intervals:")
	fmt.Println("   Sort by END → count distinct groups")
	fmt.Println()

	fmt.Println("6. Weighted scheduling:")
	fmt.Println("   DP + binary search (greedy insufficient)")
	fmt.Println()

	fmt.Println("Key insight:")
	fmt.Println("   Sort by END for selection/coverage")
	fmt.Println("   Sort by START for resource allocation")
}
```

---

## Key Takeaways

1. Sort by end time for max non-overlapping selection
2. Min removals = total − max non-overlapping
3. Sweep line for max overlap and resource counting
4. Weighted variant needs DP + binary search
5. All interval scheduling shares the same core greedy framework

> **Next up:** Huffman Coding →
