# Phase 18: Intervals — Sweep Line Algorithm

## Overview

The **sweep line** (or line sweep) algorithm processes events along a sorted axis. Instead of comparing all pairs of intervals O(n²), we process start/end events in order, maintaining active state as we sweep.

| Event Type | Action |
|-----------|--------|
| Start (+1) | Add to active set |
| End (−1) | Remove from active set |
| Query | Check active set |

**Time:** O(n log n) for sorting events.

---

## Example 1: Maximum Overlapping Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func maxOverlap(intervals [][]int) int {
	events := [][]int{}
	for _, iv := range intervals {
		events = append(events, []int{iv[0], 1})  // start
		events = append(events, []int{iv[1], -1}) // end
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i][0] == events[j][0] {
			return events[i][1] < events[j][1] // end before start at same point
		}
		return events[i][0] < events[j][0]
	})

	maxActive, active := 0, 0
	for _, e := range events {
		active += e[1]
		if active > maxActive {
			maxActive = active
		}
	}
	return maxActive
}

func main() {
	intervals := [][]int{{1, 5}, {2, 6}, {3, 7}, {4, 8}}
	fmt.Println("Max overlap:", maxOverlap(intervals))
	// At time 4: [1,5],[2,6],[3,7],[4,8] all active → 4
}
```

---

## Example 2: Minimum Platforms (Meeting Rooms II)

```go
package main

import (
	"fmt"
	"sort"
)

func minMeetingRooms(intervals [][]int) int {
	starts := make([]int, len(intervals))
	ends := make([]int, len(intervals))
	for i, iv := range intervals {
		starts[i] = iv[0]
		ends[i] = iv[1]
	}
	sort.Ints(starts)
	sort.Ints(ends)

	rooms, maxRooms := 0, 0
	si, ei := 0, 0
	for si < len(starts) {
		if starts[si] < ends[ei] {
			rooms++
			si++
		} else {
			rooms--
			ei++
		}
		if rooms > maxRooms {
			maxRooms = rooms
		}
	}
	return maxRooms
}

func main() {
	meetings := [][]int{{0, 30}, {5, 10}, {15, 20}}
	fmt.Println("Min rooms:", minMeetingRooms(meetings)) // 2
}
```

---

## Example 3: Sky Line Problem (Simplified)

```go
package main

import (
	"fmt"
	"sort"
)

type Event struct {
	X, Height int
	IsStart   bool
}

func getSkyline(buildings [][]int) [][]int {
	events := []Event{}
	for _, b := range buildings {
		events = append(events, Event{b[0], b[2], true})
		events = append(events, Event{b[1], b[2], false})
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i].X == events[j].X {
			if events[i].IsStart && events[j].IsStart {
				return events[i].Height > events[j].Height // taller start first
			}
			if !events[i].IsStart && !events[j].IsStart {
				return events[i].Height < events[j].Height // shorter end first
			}
			return events[i].IsStart // start before end
		}
		return events[i].X < events[j].X
	})

	// Using a simple approach: track active heights
	active := map[int]int{0: 1} // height → count, 0 always present
	result := [][]int{}
	prevMax := 0

	getMax := func() int {
		mx := 0
		for h := range active { if h > mx { mx = h } }
		return mx
	}

	for _, e := range events {
		if e.IsStart {
			active[e.Height]++
		} else {
			active[e.Height]--
			if active[e.Height] == 0 {
				delete(active, e.Height)
			}
		}
		currMax := getMax()
		if currMax != prevMax {
			result = append(result, []int{e.X, currMax})
			prevMax = currMax
		}
	}
	return result
}

func main() {
	buildings := [][]int{{2, 9, 10}, {3, 7, 15}, {5, 12, 12}, {15, 20, 10}, {19, 24, 8}}
	fmt.Println(getSkyline(buildings))
}
```

---

## Example 4: Count Points Inside Intervals

```go
package main

import (
	"fmt"
	"sort"
)

func countPointsInIntervals(intervals [][]int, points []int) []int {
	type Event struct {
		pos, typ, idx int
		// typ: 0=start, 1=point, 2=end
	}

	events := []Event{}
	for _, iv := range intervals {
		events = append(events, Event{iv[0], 0, -1})
		events = append(events, Event{iv[1], 2, -1})
	}
	for i, p := range points {
		events = append(events, Event{p, 1, i})
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i].pos == events[j].pos {
			return events[i].typ < events[j].typ
		}
		return events[i].pos < events[j].pos
	})

	result := make([]int, len(points))
	active := 0

	for _, e := range events {
		switch e.typ {
		case 0:
			active++
		case 1:
			result[e.idx] = active
		case 2:
			active--
		}
	}
	return result
}

func main() {
	intervals := [][]int{{1, 5}, {2, 6}, {3, 8}}
	points := []int{4, 7, 1}
	fmt.Println(countPointsInIntervals(intervals, points))
	// 4 → in all 3; 7 → in [3,8] = 1; 1 → in [1,5] = 1
}
```

---

## Example 5: Minimum Number of Arrows

```go
package main

import (
	"fmt"
	"sort"
)

func findMinArrowShots(points [][]int) int {
	if len(points) == 0 {
		return 0
	}
	sort.Slice(points, func(i, j int) bool {
		return points[i][1] < points[j][1]
	})

	arrows := 1
	end := points[0][1]

	for i := 1; i < len(points); i++ {
		if points[i][0] > end {
			arrows++
			end = points[i][1]
		}
	}
	return arrows
}

func main() {
	balloons := [][]int{{10, 16}, {2, 8}, {1, 6}, {7, 12}}
	fmt.Println("Min arrows:", findMinArrowShots(balloons)) // 2
}
```

---

## Example 6: Rectangle Area Union (Coordinate Compression + Sweep)

```go
package main

import (
	"fmt"
	"sort"
)

func rectangleArea(rects [][]int) int {
	// Collect all Y coordinates
	ys := []int{}
	type Event struct {
		x, y1, y2 int
		typ        int // 1 = open, -1 = close
	}
	events := []Event{}

	for _, r := range rects {
		ys = append(ys, r[1], r[3])
		events = append(events, Event{r[0], r[1], r[3], 1})
		events = append(events, Event{r[2], r[1], r[3], -1})
	}

	sort.Ints(ys)
	ys = unique(ys)

	sort.Slice(events, func(i, j int) bool {
		return events[i].x < events[j].x
	})

	count := make([]int, len(ys)) // count of active rectangles covering each y segment
	area := 0
	prevX := events[0].x

	for _, e := range events {
		if e.x != prevX {
			// Calculate active Y length
			activeY := 0
			for i := 0; i < len(ys)-1; i++ {
				if count[i] > 0 {
					activeY += ys[i+1] - ys[i]
				}
			}
			area += activeY * (e.x - prevX)
			prevX = e.x
		}

		// Update counts
		for i := 0; i < len(ys)-1; i++ {
			if ys[i] >= e.y1 && ys[i+1] <= e.y2 {
				count[i] += e.typ
			}
		}
	}
	return area
}

func unique(a []int) []int {
	sort.Ints(a)
	r := []int{a[0]}
	for i := 1; i < len(a); i++ {
		if a[i] != a[i-1] {
			r = append(r, a[i])
		}
	}
	return r
}

func main() {
	rects := [][]int{{0, 0, 2, 2}, {1, 0, 2, 3}, {1, 0, 3, 1}}
	fmt.Println("Total area:", rectangleArea(rects)) // 6
}
```

---

## Example 7: Right Interval (Binary Search + Sorting)

```go
package main

import (
	"fmt"
	"sort"
)

func findRightInterval(intervals [][]int) []int {
	n := len(intervals)
	type indexedInterval struct {
		start, end, idx int
	}

	sorted := make([]indexedInterval, n)
	for i, iv := range intervals {
		sorted[i] = indexedInterval{iv[0], iv[1], i}
	}
	sort.Slice(sorted, func(i, j int) bool {
		return sorted[i].start < sorted[j].start
	})

	starts := make([]int, n)
	for i, s := range sorted {
		starts[i] = s.start
	}

	result := make([]int, n)
	for _, iv := range sorted {
		pos := sort.SearchInts(starts, iv.end)
		if pos < n {
			result[iv.idx] = sorted[pos].idx
		} else {
			result[iv.idx] = -1
		}
	}
	return result
}

func main() {
	intervals := [][]int{{3, 4}, {2, 3}, {1, 2}}
	fmt.Println(findRightInterval(intervals)) // [-1,0,1]
}
```

---

## Example 8: Car Pooling

```go
package main

import (
	"fmt"
	"sort"
)

func carPooling(trips [][]int, capacity int) bool {
	events := [][]int{}
	for _, t := range trips {
		events = append(events, []int{t[1], t[0]})  // pickup
		events = append(events, []int{t[2], -t[0]}) // dropoff
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i][0] == events[j][0] {
			return events[i][1] < events[j][1] // dropoff before pickup at same point
		}
		return events[i][0] < events[j][0]
	})

	curr := 0
	for _, e := range events {
		curr += e[1]
		if curr > capacity {
			return false
		}
	}
	return true
}

func main() {
	trips := [][]int{{2, 1, 5}, {3, 3, 7}}
	fmt.Println(carPooling(trips, 4))  // false
	fmt.Println(carPooling(trips, 5))  // true
}
```

---

## Example 9: Range Addition (Difference Array)

```go
package main

import "fmt"

func rangeAddition(length int, updates [][]int) []int {
	// Difference array approach: sweep line on array indices
	diff := make([]int, length+1)

	for _, u := range updates {
		start, end, val := u[0], u[1], u[2]
		diff[start] += val
		if end+1 < len(diff) {
			diff[end+1] -= val
		}
	}

	result := make([]int, length)
	sum := 0
	for i := 0; i < length; i++ {
		sum += diff[i]
		result[i] = sum
	}
	return result
}

func main() {
	// length=5, updates: [1,3,2], [2,4,3], [0,2,-2]
	updates := [][]int{{1, 3, 2}, {2, 4, 3}, {0, 2, -2}}
	fmt.Println(rangeAddition(5, updates))
	// [-2, 0, 3, 5, 3]
}
```

---

## Example 10: Sweep Line Pattern Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Sweep Line Patterns ===")
	fmt.Println()

	patterns := []struct{ name, description, events, state string }{
		{
			"Max Overlap",
			"Count maximum simultaneous intervals",
			"+1 at start, -1 at end",
			"Running sum (counter)",
		},
		{
			"Meeting Rooms",
			"Minimum resources needed",
			"Sort starts/ends separately",
			"Two-pointer on sorted arrays",
		},
		{
			"Skyline",
			"Outline of merged rectangles",
			"Building start/end events",
			"Max-heap of active heights",
		},
		{
			"Rectangle Union",
			"Area of overlapping rectangles",
			"Vertical edge events + Y compress",
			"Segment counts or segment tree",
		},
		{
			"Difference Array",
			"Range updates then query",
			"+val at start, -val at end+1",
			"Prefix sum to reconstruct",
		},
	}

	for _, p := range patterns {
		fmt.Printf("Pattern: %s\n", p.name)
		fmt.Printf("  Use: %s\n", p.description)
		fmt.Printf("  Events: %s\n", p.events)
		fmt.Printf("  State: %s\n\n", p.state)
	}
}
```

---

## Key Takeaways

| Technique | Time | Space | Use Case |
|-----------|------|-------|----------|
| Event sort | O(n log n) | O(n) | Generic sweep line |
| Two arrays sort | O(n log n) | O(n) | Meeting rooms |
| Difference array | O(n + q) | O(n) | Range updates |
| Coordinate compression | O(n log n) | O(n) | Rectangle area |

1. Convert intervals to events (start/end), sort, sweep
2. At same position: process ends before starts (typically)
3. Maintain running count, max-heap, or segment tree as active state
4. Difference array is the discrete version of sweep line
5. Coordinate compression handles continuous domains

> **Next up:** Event-Based Processing →
