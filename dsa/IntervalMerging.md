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

**Textual Figure:**

```
Basic Merge Intervals: [[1,3],[2,6],[8,10],[15,18]]

Step 1: Sort by start time (already sorted)

Number line:
  0    2    4    6    8   10   12   14   16   18
  ├────┼────┼────┼────┼────┼────┼────┼────┼────┤

  [1,3]   ├────────┤
  [2,6]      ├──────────┤
  [8,10]                    ├────┤
  [15,18]                              ├──────┤

Step 2: Merge pass
  Start with result = [[1,3]]
  [2,6]:  2 ≤ 3 (overlaps) → extend end to 6 → result = [[1,6]]
  [8,10]: 8 > 6 (no overlap) → append → result = [[1,6],[8,10]]
  [15,18]: 15 > 10 (no overlap) → append → result = [[1,6],[8,10],[15,18]]

Result:
  [1,6]   ├────────────────┤
  [8,10]                    ├────┤
  [15,18]                              ├──────┤

  └────────── 3 merged intervals ──────────────┘
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

**Textual Figure:**

```
Insert and Merge — Case 1: intervals=[[1,3],[6,9]], new=[2,5]

  0    2    4    6    8   10
  ├────┼────┼────┼────┼────┤
  [1,3]  ├──────┤
  new    ·  ├────────┤  [2,5]
  [6,9]              ├──────┤

  Phase 1 — before new: [1,3] ends at 3 ≥ 2 → overlaps, skip
  Phase 2 — merge overlapping:
    [1,3]: start 1 < 2 → new.start=1; end 3 < 5 → keep 5
    [6,9]: start 6 > 5 → stop merging
    → merged new = [1,5]
  Phase 3 — after: append [6,9]
  Result: [[1,5],[6,9]]

Insert and Merge — Case 2: intervals=[[1,2],[3,5],[6,7],[8,10],[12,16]], new=[4,8]

  0    2    4    6    8   10   12   14   16
  ├────┼────┼────┼────┼────┼────┼────┼────┤
  [1,2] ├──┤
  [3,5]      ├────┤
  new          ├──────────┤  [4,8]
  [6,7]            ├──┤
  [8,10]                ├────┤
  [12,16]                        ├────────┤

  Phase 1 — before: [1,2] ends 2 < 4 → add [1,2]
  Phase 2 — merge [3,5],[6,7],[8,10] with new [4,8] → [3,10]
  Phase 3 — after: append [12,16]
  Result: [[1,2],[3,10],[12,16]]
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

**Textual Figure:**

```
Count Merged Intervals: [[1,4],[2,5],[7,9],[8,10],[12,15]]

Sorted (already sorted by start):

  0    2    4    6    8   10   12   14   16
  ├────┼────┼────┼────┼────┼────┼────┼────┤
  [1,4]  ├──────────┤
  [2,5]     ├──────────┤         ← overlaps [1,4] → merge group 1
  [7,9]                  ├────┤
  [8,10]                   ├────┤  ← overlaps [7,9] → merge group 2
  [12,15]                        ├──────┤  ← no overlap → group 3

  Tracking end pointer:
    end=4 → [2,5]: 2≤4 overlap, update end=5
           → [7,9]: 7>5 new group, count++, end=9
           → [8,10]: 8≤9 overlap, update end=10
           → [12,15]: 12>10 new group, count++, end=15

  Groups:  ┌─────────┐  ┌──────┐  ┌──────┐
           │ [1,5]   │  │[7,10]│  │[12,15]│
           └─────────┘  └──────┘  └──────┘
  Answer: 3 merged intervals
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

**Textual Figure:**

```
Merge Employee Schedules → Find Free Time

Employee 1: [[1,3],[6,7]]
Employee 2: [[2,4]]
Employee 3: [[2,5],[9,12]]

  0    2    4    6    8   10   12
  ├────┼────┼────┼────┼────┼────┤
  E1: ├────┤         ├──┤
  E2:    ├────┤
  E3:    ├──────┤              ├──────┤

Step 1: Flatten + sort: [[1,3],[2,4],[2,5],[6,7],[9,12]]

Step 2: Merge overlapping:
  [1,3] → [2,4]: 2≤3 → [1,4]
        → [2,5]: 2≤4 → [1,5]
        → [6,7]: 6>5 → new: [6,7]
        → [9,12]: 9>7 → new: [9,12]
  Merged busy: [[1,5],[6,7],[9,12]]

Step 3: Gaps = free time:
  [1,5]──┤  gap [5,6]  ├──[6,7]──┤  gap [7,9]  ├──[9,12]

  Free time: [[5,6],[7,9]]
           ═══════  ═══════
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

**Textual Figure:**

```
Remove Covered Intervals: [[1,4],[3,6],[2,8]]

Sort by start asc, end desc: [[1,4],[2,8],[3,6]]

  0    2    4    6    8
  ├────┼────┼────┼────┤
  [1,4] ├──────────┤
  [2,8]    ├────────────────┤
  [3,6]       ├────────┤  ← covered by [2,8]!

  Scan with maxEnd tracking:
    [1,4]: end=4 > maxEnd=0 → count=1, maxEnd=4
    [2,8]: end=8 > maxEnd=4 → count=2, maxEnd=8
    [3,6]: end=6 ≤ maxEnd=8 → COVERED (skip)

  Remaining: 2 intervals (after removing covered ones)
             [1,4] and [2,8] survive
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

**Textual Figure:**

```
Merge Touching Intervals: [[1,5],[5,8],[10,12]]

  0    2    4    6    8   10   12
  ├────┼────┼────┼────┼────┼────┤
  [1,5]  ├────────────┤
  [5,8]              ├────────┤  ← touches at point 5
  [10,12]                     ├────┤
                      │
                  touch point

  Merge pass (using ≤ for touching):
    result = [[1,5]]
    [5,8]: 5 ≤ 5 → overlaps/touches → extend to [1,8]
    [10,12]: 10 > 8 → new interval

  Result: [[1,8],[10,12]]
  [1,8]  ├─────────────────────┤
  [10,12]                     ├────┤
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

**Textual Figure:**

```
Interval Union Length: [[1,3],[2,5],[6,8]]

  0    2    4    6    8
  ├────┼────┼────┼────┤
  [1,3] ├──────┤
  [2,5]    ├────────┤
  [6,8]              ├────┤

  Merge tracking (start, end):
    init: start=1, end=3
    [2,5]: 2 ≤ 3 → overlap, end=5
    [6,8]: 6 > 5 → flush length (5-1)=4, start=6, end=8
    Final flush: length (8-6)=2

  Union on number line:
  ├────████████████┤    ├████┤
  1              5    6    8
    length = 4         len = 2

  Total union length: 4 + 2 = 6
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

**Textual Figure:**

```
Minimum Intervals to Remove: [[1,2],[2,3],[3,4],[1,3]]

Sort by end time:
  [[1,2],[2,3],[1,3],[3,4]]

  0    1    2    3    4
  ├────┼────┼────┼────┤
  [1,2] ├────┤         ← keep (end=2)
  [2,3]      ├────┤    ← 2 ≥ 2, keep (end=3)
  [1,3] ├─────────┤    ← 1 < 3, OVERLAPS → remove!
  [3,4]           ├────┤ ← 3 ≥ 3, keep (end=4)

  Greedy: keep earliest-ending non-overlapping intervals
    Kept: [1,2], [2,3], [3,4]  (3 kept)
    Removed: [1,3]             (1 removed)

  Answer: 1 removal needed
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

**Textual Figure:**

```
Merge with Weighted Intervals:
  Input: [{1,3,w=10}, {2,5,w=20}, {7,9,w=15}]

  0    2    4    6    8   10
  ├────┼────┼────┼────┼────┤
  {1,3}w=10  ├──────┤
  {2,5}w=20     ├────────┤
  {7,9}w=15                  ├────┤

  Merge pass:
    result = [{1,3,w=10}]
    {2,5,w=20}: 2 ≤ 3 → overlap
      extend end: 3 → 5
      accumulate weight: 10+20 = 30
      → result = [{1,5,w=30}]
    {7,9,w=15}: 7 > 5 → no overlap
      → append result = [{1,5,w=30}, {7,9,w=15}]

  Result:
  {1,5}w=30  ├────────────┤  weight=30
  {7,9}w=15                  ├────┤  weight=15
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

**Textual Figure:**

```
Interval Component Count: [[1,3],[2,5],[7,9],[8,10],[12,15]]

  0    2    4    6    8   10   12   14   16
  ├────┼────┼────┼────┼────┼────┼────┼────┤
  [1,3]  ├──────┤
  [2,5]     ├────────┤
            overlap! → union(0,1)

  [7,9]                  ├────┤
  [8,10]                   ├────┤
                 overlap! → union(2,3)

  [12,15]                       ├──────┤
                 no overlap with others

  DSU state after all unions:
    Component 1: {[1,3], [2,5]}   ← indices 0,1
    Component 2: {[7,9], [8,10]}  ← indices 2,3
    Component 3: {[12,15]}        ← index 4

    ┌────────────┐  ┌─────────────┐  ┌────────┐
    │ [1,3]─[2,5] │  │ [7,9]─[8,10] │  │ [12,15] │
    └────────────┘  └─────────────┘  └────────┘

  Answer: 3 connected components
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
