# Phase 18: Intervals — Interval Intersection

## Overview

**Interval intersection** finds the overlap between two sets of intervals. The two-pointer technique processes both sorted lists simultaneously, advancing the pointer with the smaller endpoint.

| Overlap Condition | Formula |
|-------------------|---------|
| Two intervals overlap | `a.start ≤ b.end AND b.start ≤ a.end` |
| Intersection | `[max(a.start, b.start), min(a.end, b.end)]` |
| Advance rule | Move pointer for interval that ends first |

---

## Example 1: Interval List Intersections

```go
package main

import "fmt"

func intervalIntersection(A, B [][]int) [][]int {
	result := [][]int{}
	i, j := 0, 0

	for i < len(A) && j < len(B) {
		lo := A[i][0]
		if B[j][0] > lo { lo = B[j][0] }
		hi := A[i][1]
		if B[j][1] < hi { hi = B[j][1] }

		if lo <= hi {
			result = append(result, []int{lo, hi})
		}

		// Advance the one that ends first
		if A[i][1] < B[j][1] {
			i++
		} else {
			j++
		}
	}
	return result
}

func main() {
	A := [][]int{{0, 2}, {5, 10}, {13, 23}, {24, 25}}
	B := [][]int{{1, 5}, {8, 12}, {15, 24}, {25, 26}}
	fmt.Println(intervalIntersection(A, B))
	// [[1,2],[5,5],[8,10],[15,23],[24,24],[25,25]]
}
```

---

## Example 2: Check If Two Intervals Overlap

```go
package main

import "fmt"

func overlaps(a, b []int) bool {
	return a[0] <= b[1] && b[0] <= a[1]
}

func intersection(a, b []int) []int {
	if !overlaps(a, b) {
		return nil
	}
	lo := a[0]
	if b[0] > lo { lo = b[0] }
	hi := a[1]
	if b[1] < hi { hi = b[1] }
	return []int{lo, hi}
}

func main() {
	fmt.Println(overlaps([]int{1, 5}, []int{3, 7}))  // true
	fmt.Println(overlaps([]int{1, 5}, []int{6, 8}))  // false
	fmt.Println(overlaps([]int{1, 5}, []int{5, 8}))  // true (touching)

	fmt.Println(intersection([]int{1, 5}, []int{3, 7})) // [3,5]
	fmt.Println(intersection([]int{1, 5}, []int{6, 8})) // []
}
```

---

## Example 3: Intersection of Multiple Interval Lists

```go
package main

import (
	"fmt"
	"sort"
)

func intersectMultiple(lists [][][]int) [][]int {
	if len(lists) == 0 {
		return nil
	}
	result := lists[0]
	for i := 1; i < len(lists); i++ {
		result = pairIntersect(result, lists[i])
	}
	return result
}

func pairIntersect(A, B [][]int) [][]int {
	res := [][]int{}
	i, j := 0, 0
	for i < len(A) && j < len(B) {
		lo := A[i][0]
		if B[j][0] > lo { lo = B[j][0] }
		hi := A[i][1]
		if B[j][1] < hi { hi = B[j][1] }
		if lo <= hi {
			res = append(res, []int{lo, hi})
		}
		if A[i][1] < B[j][1] { i++ } else { j++ }
	}
	return res
}

func main() {
	lists := [][][]int{
		{{0, 5}, {10, 15}},
		{{2, 8}, {12, 20}},
		{{3, 7}, {11, 18}},
	}
	// Use sort to make sure each list is sorted
	for _, l := range lists {
		sort.Slice(l, func(i, j int) bool { return l[i][0] < l[j][0] })
	}
	fmt.Println(intersectMultiple(lists))
	// [[3,5],[12,15]]
}
```

---

## Example 4: Total Intersection Length

```go
package main

import "fmt"

func intersectionLength(A, B [][]int) int {
	total := 0
	i, j := 0, 0

	for i < len(A) && j < len(B) {
		lo := A[i][0]
		if B[j][0] > lo { lo = B[j][0] }
		hi := A[i][1]
		if B[j][1] < hi { hi = B[j][1] }

		if lo <= hi {
			total += hi - lo
		}

		if A[i][1] < B[j][1] { i++ } else { j++ }
	}
	return total
}

func main() {
	A := [][]int{{1, 5}, {10, 15}}
	B := [][]int{{3, 12}}
	fmt.Println("Intersection length:", intersectionLength(A, B))
	// [3,5] length 2, [10,12] length 2 → total 4
}
```

---

## Example 5: Find Common Free Time

```go
package main

import (
	"fmt"
	"sort"
)

func commonFreeTime(schedules [][][]int) [][]int {
	// Step 1: Find busy time (union of all)
	all := [][]int{}
	for _, s := range schedules {
		all = append(all, s...)
	}
	sort.Slice(all, func(i, j int) bool {
		return all[i][0] < all[j][0]
	})

	// Merge
	merged := [][]int{all[0]}
	for i := 1; i < len(all); i++ {
		last := merged[len(merged)-1]
		if all[i][0] <= last[1] {
			if all[i][1] > last[1] { last[1] = all[i][1] }
		} else {
			merged = append(merged, all[i])
		}
	}

	// Gaps between merged = free time
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
	fmt.Println("Common free time:", commonFreeTime(schedules))
	// [[5,6],[7,9]]
}
```

---

## Example 6: Range Coverage Check

```go
package main

import (
	"fmt"
	"sort"
)

func isCovered(ranges [][]int, left, right int) bool {
	sort.Slice(ranges, func(i, j int) bool {
		return ranges[i][0] < ranges[j][0]
	})

	// Merge ranges
	merged := [][]int{ranges[0]}
	for i := 1; i < len(ranges); i++ {
		last := merged[len(merged)-1]
		if ranges[i][0] <= last[1]+1 {
			if ranges[i][1] > last[1] { last[1] = ranges[i][1] }
		} else {
			merged = append(merged, ranges[i])
		}
	}

	// Check if [left, right] is fully within any merged interval
	for _, m := range merged {
		if m[0] <= left && m[1] >= right {
			return true
		}
	}
	return false
}

func main() {
	ranges := [][]int{{1, 2}, {3, 4}, {5, 6}}
	fmt.Println(isCovered(ranges, 2, 5)) // true

	ranges2 := [][]int{{1, 2}, {4, 6}}
	fmt.Println(isCovered(ranges2, 2, 5)) // false (3 not covered)
}
```

---

## Example 7: Interval Containment

```go
package main

import (
	"fmt"
	"sort"
)

func countContained(intervals [][]int) int {
	// Sort by start asc, end desc
	sort.Slice(intervals, func(i, j int) bool {
		if intervals[i][0] == intervals[j][0] {
			return intervals[i][1] > intervals[j][1]
		}
		return intervals[i][0] < intervals[j][0]
	})

	count := 0
	maxEnd := -1

	for _, iv := range intervals {
		if iv[1] <= maxEnd {
			count++ // contained in previous
		}
		if iv[1] > maxEnd {
			maxEnd = iv[1]
		}
	}
	return count
}

func main() {
	intervals := [][]int{{1, 10}, {2, 5}, {3, 8}, {6, 12}}
	fmt.Println("Contained intervals:", countContained(intervals))
	// [2,5] contained in [1,10], [3,8] contained in [1,10] → 2
}
```

---

## Example 8: My Calendar (No Double Booking)

```go
package main

import "fmt"

type MyCalendar struct {
	bookings [][]int
}

func (c *MyCalendar) Book(start, end int) bool {
	for _, b := range c.bookings {
		if start < b[1] && end > b[0] {
			return false // overlap
		}
	}
	c.bookings = append(c.bookings, []int{start, end})
	return true
}

func main() {
	cal := &MyCalendar{}
	fmt.Println(cal.Book(10, 20)) // true
	fmt.Println(cal.Book(15, 25)) // false (overlaps [10,20))
	fmt.Println(cal.Book(20, 30)) // true ([20,30) doesn't overlap [10,20))
}
```

---

## Example 9: Intersection with Point Queries

```go
package main

import (
	"fmt"
	"sort"
)

func pointInIntervals(intervals [][]int, points []int) []int {
	// Sort intervals by start
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	result := make([]int, len(points))
	for pi, p := range points {
		count := 0
		for _, iv := range intervals {
			if iv[0] > p { break } // no more can contain p
			if p <= iv[1] {
				count++
			}
		}
		result[pi] = count
	}
	return result
}

func main() {
	intervals := [][]int{{1, 4}, {2, 6}, {5, 8}, {7, 10}}
	points := []int{3, 5, 9}
	fmt.Println("Points covered:", pointInIntervals(intervals, points))
	// 3 → in [1,4],[2,6] = 2; 5 → in [2,6],[5,8] = 2; 9 → in [7,10] = 1
}
```

---

## Example 10: Intersection Size at Least Two

```go
package main

import (
	"fmt"
	"sort"
)

func intersectionSizeTwo(intervals [][]int) int {
	// Sort by end asc, then start desc
	sort.Slice(intervals, func(i, j int) bool {
		if intervals[i][1] == intervals[j][1] {
			return intervals[i][0] > intervals[j][0]
		}
		return intervals[i][1] < intervals[j][1]
	})

	// Start with two points from first interval
	p1 := intervals[0][1] - 1
	p2 := intervals[0][1]
	count := 2

	for i := 1; i < len(intervals); i++ {
		s := intervals[i][0]
		if s > p2 {
			// No overlap — need 2 new points
			p1 = intervals[i][1] - 1
			p2 = intervals[i][1]
			count += 2
		} else if s > p1 {
			// Overlaps with p2 only — need 1 new point
			p1 = p2
			p2 = intervals[i][1]
			count++
		}
		// else: both p1 and p2 are in this interval
	}
	return count
}

func main() {
	intervals := [][]int{{1, 3}, {3, 7}, {8, 9}}
	fmt.Println("Min points:", intersectionSizeTwo(intervals))
	// Need at least 2 points in each interval's intersection with chosen set
}
```

---

## Key Takeaways

1. Two-pointer intersection: advance the pointer with smaller endpoint
2. Intersection exists iff `max(starts) ≤ min(ends)`
3. Intersection = `[max(a.start, b.start), min(a.end, b.end)]`
4. For open/closed intervals: watch boundary conditions (< vs ≤)
5. Many intersection problems can be chained: intersect pairwise

> **Next up:** Sweep Line Algorithm →
