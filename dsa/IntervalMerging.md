# Phase 18: Intervals — Interval Merging

## Overview

**Interval merging** combines overlapping or touching intervals into a minimal set of non-overlapping intervals. The key pattern: sort by start time, then merge adjacent intervals when they overlap.

| Step | Action |
|------|--------|
| 1 | Sort intervals by start time |
| 2 | Initialize result with first interval |
| 3 | For each interval: if overlaps last in result → extend end |
| 4 | Otherwise, append new interval |

**Time:** O(n log n) for sorting, O(n) for merge pass.

---

## Example 1: Basic Merge Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func merge(intervals [][]int) [][]int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	result := [][]int{intervals[0]}

	for i := 1; i < len(intervals); i++ {
		last := result[len(result)-1]
		curr := intervals[i]

		if curr[0] <= last[1] {
			// Overlapping — extend end
			if curr[1] > last[1] {
				last[1] = curr[1]
			}
		} else {
			result = append(result, curr)
		}
	}
	return result
}

func main() {
	intervals := [][]int{{1, 3}, {2, 6}, {8, 10}, {15, 18}}
	fmt.Println("Merged:", merge(intervals))
	// [[1,6],[8,10],[15,18]]
}
```

---

## Example 2: Insert and Merge

```go
package main

import "fmt"

func insert(intervals [][]int, newInterval []int) [][]int {
	result := [][]int{}
	i := 0
	n := len(intervals)

	// Add all intervals ending before newInterval starts
	for i < n && intervals[i][1] < newInterval[0] {
		result = append(result, intervals[i])
		i++
	}

	// Merge overlapping intervals with newInterval
	for i < n && intervals[i][0] <= newInterval[1] {
		if intervals[i][0] < newInterval[0] {
			newInterval[0] = intervals[i][0]
		}
		if intervals[i][1] > newInterval[1] {
			newInterval[1] = intervals[i][1]
		}
		i++
	}
	result = append(result, newInterval)

	// Add remaining intervals
	for i < n {
		result = append(result, intervals[i])
		i++
	}
	return result
}

func main() {
	intervals := [][]int{{1, 3}, {6, 9}}
	fmt.Println(insert(intervals, []int{2, 5}))
	// [[1,5],[6,9]]

	intervals2 := [][]int{{1, 2}, {3, 5}, {6, 7}, {8, 10}, {12, 16}}
	fmt.Println(insert(intervals2, []int{4, 8}))
	// [[1,2],[3,10],[12,16]]
}
```

---

## Example 3: Count Merged Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func countMerged(intervals [][]int) int {
	if len(intervals) == 0 {
		return 0
	}
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	count := 1
	end := intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] > end {
			count++
			end = intervals[i][1]
		} else if intervals[i][1] > end {
			end = intervals[i][1]
		}
	}
	return count
}

func main() {
	intervals := [][]int{{1, 4}, {2, 5}, {7, 9}, {8, 10}, {12, 15}}
	fmt.Println("Merged count:", countMerged(intervals)) // 3
}
```

---

## Example 4: Merge with Employee Schedules

```go
package main

import (
	"fmt"
	"sort"
)

func mergeSchedules(schedules [][][]int) [][]int {
	// Flatten all schedules
	all := [][]int{}
	for _, emp := range schedules {
		all = append(all, emp...)
	}

	sort.Slice(all, func(i, j int) bool {
		return all[i][0] < all[j][0]
	})

	merged := [][]int{all[0]}
	for i := 1; i < len(all); i++ {
		last := merged[len(merged)-1]
		if all[i][0] <= last[1] {
			if all[i][1] > last[1] {
				last[1] = all[i][1]
			}
		} else {
			merged = append(merged, all[i])
		}
	}
	return merged
}

func employeeFreeTime(schedules [][][]int) [][]int {
	merged := mergeSchedules(schedules)
	free := [][]int{}
	for i := 1; i < len(merged); i++ {
		free = append(free, []int{merged[i-1][1], merged[i][0]})
	}
	return free
}

func main() {
	schedules := [][][]int{
		{{1, 3}, {6, 7}},
		{{2, 4}},
		{{2, 5}, {9, 12}},
	}
	fmt.Println("Employee free time:", employeeFreeTime(schedules))
	// [[5,6],[7,9]]
}
```

---

## Example 5: Remove Covered Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func removeCoveredIntervals(intervals [][]int) int {
	// Sort by start asc, then by end desc
	sort.Slice(intervals, func(i, j int) bool {
		if intervals[i][0] == intervals[j][0] {
			return intervals[i][1] > intervals[j][1]
		}
		return intervals[i][0] < intervals[j][0]
	})

	count := 0
	maxEnd := 0

	for _, iv := range intervals {
		if iv[1] > maxEnd {
			count++
			maxEnd = iv[1]
		}
		// else it's covered by a previous interval
	}
	return count
}

func main() {
	intervals := [][]int{{1, 4}, {3, 6}, {2, 8}}
	fmt.Println("Remaining after removing covered:", removeCoveredIntervals(intervals))
	// 2: [1,4] is covered by [2,8]? No. [2,8] covers [3,6]? [3,6] not covered.
	// Sort: [1,4],[2,8],[3,6] → [2,8] covers [3,6] → 2 remaining
}
```

---

## Example 6: Merge Touching Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func mergeTouching(intervals [][]int) [][]int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	result := [][]int{intervals[0]}
	for i := 1; i < len(intervals); i++ {
		last := result[len(result)-1]
		// <= for overlapping, use < to NOT merge touching
		if intervals[i][0] <= last[1] {
			if intervals[i][1] > last[1] {
				last[1] = intervals[i][1]
			}
		} else {
			result = append(result, intervals[i])
		}
	}
	return result
}

func main() {
	// [1,5] and [5,8] touch at point 5
	intervals := [][]int{{1, 5}, {5, 8}, {10, 12}}
	fmt.Println("Merge touching:", mergeTouching(intervals))
	// [[1,8],[10,12]]
}
```

---

## Example 7: Interval Union Length

```go
package main

import (
	"fmt"
	"sort"
)

func unionLength(intervals [][]int) int {
	if len(intervals) == 0 {
		return 0
	}
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	total := 0
	start, end := intervals[0][0], intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] > end {
			total += end - start
			start = intervals[i][0]
			end = intervals[i][1]
		} else if intervals[i][1] > end {
			end = intervals[i][1]
		}
	}
	total += end - start
	return total
}

func main() {
	intervals := [][]int{{1, 3}, {2, 5}, {6, 8}}
	fmt.Println("Union length:", unionLength(intervals)) // (5-1) + (8-6) = 6
}
```

---

## Example 8: Minimum Number of Intervals to Remove

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
	end := intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] < end {
			count++ // remove this one (it overlaps)
		} else {
			end = intervals[i][1]
		}
	}
	return count
}

func main() {
	intervals := [][]int{{1, 2}, {2, 3}, {3, 4}, {1, 3}}
	fmt.Println("Remove:", eraseOverlapIntervals(intervals)) // 1
}
```

---

## Example 9: Merge with Weighted Intervals

```go
package main

import (
	"fmt"
	"sort"
)

type WeightedInterval struct {
	Start, End, Weight int
}

func mergeWeighted(intervals []WeightedInterval) []WeightedInterval {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i].Start < intervals[j].Start
	})

	result := []WeightedInterval{intervals[0]}
	for i := 1; i < len(intervals); i++ {
		last := &result[len(result)-1]
		curr := intervals[i]

		if curr.Start <= last.End {
			if curr.End > last.End {
				last.End = curr.End
			}
			last.Weight += curr.Weight // accumulate weight
		} else {
			result = append(result, curr)
		}
	}
	return result
}

func main() {
	intervals := []WeightedInterval{
		{1, 3, 10}, {2, 5, 20}, {7, 9, 15},
	}
	merged := mergeWeighted(intervals)
	for _, iv := range merged {
		fmt.Printf("[%d,%d] weight=%d\n", iv.Start, iv.End, iv.Weight)
	}
	// [1,5] weight=30, [7,9] weight=15
}
```

---

## Example 10: Interval Component Count (Graph-Based)

```go
package main

import "fmt"

type DSU struct {
	parent, rank []int
}

func NewDSU(n int) *DSU {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &DSU{p, make([]int, n)}
}

func (d *DSU) Find(x int) int {
	if d.parent[x] != x { d.parent[x] = d.Find(d.parent[x]) }
	return d.parent[x]
}

func (d *DSU) Union(x, y int) {
	px, py := d.Find(x), d.Find(y)
	if px == py { return }
	if d.rank[px] < d.rank[py] { px, py = py, px }
	d.parent[py] = px
	if d.rank[px] == d.rank[py] { d.rank[px]++ }
}

func countComponents(intervals [][]int) int {
	n := len(intervals)
	dsu := NewDSU(n)

	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			// Check if intervals overlap
			if intervals[i][0] <= intervals[j][1] && intervals[j][0] <= intervals[i][1] {
				dsu.Union(i, j)
			}
		}
	}

	roots := map[int]bool{}
	for i := 0; i < n; i++ {
		roots[dsu.Find(i)] = true
	}
	return len(roots)
}

func main() {
	intervals := [][]int{{1, 3}, {2, 5}, {7, 9}, {8, 10}, {12, 15}}
	fmt.Println("Connected components:", countComponents(intervals))
	// 3: {[1,3],[2,5]}, {[7,9],[8,10]}, {[12,15]}
}
```

---

## Key Takeaways

| Pattern | When to Use |
|---------|-------------|
| Sort by start | Default for merging |
| Sort by end | Greedy interval scheduling |
| Sort by start, then end desc | Remove covered intervals |
| Flatten + merge | Multiple schedules |
| Union-Find | Component counting |

1. Always sort intervals before merging — O(n log n) dominates
2. Two intervals [a,b] and [c,d] overlap iff a ≤ d AND c ≤ b
3. Merge = extend the end; no overlap = start new interval
4. Insert interval: three-pass (before, overlapping, after)
5. Many problems reduce to "merge intervals" as a subroutine

> **Next up:** Interval Intersection →
