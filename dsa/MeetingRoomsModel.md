# Phase 18: Intervals — Meeting Rooms Model

## Overview

The **meeting rooms model** is a classic interval problem family: given a set of time intervals (meetings), determine room requirements, scheduling feasibility, and optimization. It unifies many interval techniques.

| Problem Variant | Core Question |
|----------------|---------------|
| Meeting Rooms I | Can one person attend all meetings? |
| Meeting Rooms II | Minimum rooms needed? |
| Meeting Rooms III | Assign rooms with rules? |
| Free time | When are all rooms free? |
| Room allocation | Which room for which meeting? |

---

## Example 1: Meeting Rooms I — Can Attend All

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
	fmt.Println(canAttendAll([][]int{{0, 30}, {5, 10}, {15, 20}})) // false
	fmt.Println(canAttendAll([][]int{{7, 10}, {2, 4}}))            // true
}
```

---

## Example 2: Meeting Rooms II — Min Rooms (Sorting)

```go
package main

import (
	"fmt"
	"sort"
)

func minRooms(intervals [][]int) int {
	starts := make([]int, len(intervals))
	ends := make([]int, len(intervals))
	for i, iv := range intervals {
		starts[i] = iv[0]
		ends[i] = iv[1]
	}
	sort.Ints(starts)
	sort.Ints(ends)

	rooms, maxRooms := 0, 0
	s, e := 0, 0

	for s < len(starts) {
		if starts[s] < ends[e] {
			rooms++
			s++
		} else {
			rooms--
			e++
		}
		if rooms > maxRooms {
			maxRooms = rooms
		}
	}
	return maxRooms
}

func main() {
	meetings := [][]int{{0, 30}, {5, 10}, {15, 20}}
	fmt.Println("Min rooms:", minRooms(meetings)) // 2

	meetings2 := [][]int{{1, 5}, {2, 3}, {4, 6}, {5, 7}}
	fmt.Println("Min rooms:", minRooms(meetings2)) // 2
}
```

---

## Example 3: Meeting Rooms II — Min Rooms (Heap)

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

func minRoomsHeap(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})

	h := &MinHeap{}
	heap.Init(h)

	for _, iv := range intervals {
		if h.Len() > 0 && (*h)[0] <= iv[0] {
			heap.Pop(h) // reuse room
		}
		heap.Push(h, iv[1]) // allocate room (end time)
	}
	return h.Len()
}

func main() {
	meetings := [][]int{{0, 30}, {5, 10}, {15, 20}}
	fmt.Println("Min rooms (heap):", minRoomsHeap(meetings)) // 2
}
```

---

## Example 4: Meeting Rooms III — Room Assignment

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

// Room that becomes free next
type Room struct {
	id      int
	freeAt  int
}
type RoomHeap []Room
func (h RoomHeap) Len() int { return len(h) }
func (h RoomHeap) Less(i, j int) bool {
	if h[i].freeAt == h[j].freeAt {
		return h[i].id < h[j].id
	}
	return h[i].freeAt < h[j].freeAt
}
func (h RoomHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *RoomHeap) Push(x interface{}) { *h = append(*h, x.(Room)) }
func (h *RoomHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func mostBooked(n int, meetings [][]int) int {
	sort.Slice(meetings, func(i, j int) bool {
		return meetings[i][0] < meetings[j][0]
	})

	count := make([]int, n)
	h := &RoomHeap{}

	// All rooms initially free
	for i := 0; i < n; i++ {
		heap.Push(h, Room{i, 0})
	}

	for _, m := range meetings {
		start, end := m[0], m[1]
		duration := end - start

		// Wait for earliest room if all busy
		room := heap.Pop(h).(Room)
		actualStart := start
		if room.freeAt > start {
			actualStart = room.freeAt
		}

		count[room.id]++
		heap.Push(h, Room{room.id, actualStart + duration})
	}

	// Find room with most bookings
	maxRoom := 0
	for i := 1; i < n; i++ {
		if count[i] > count[maxRoom] {
			maxRoom = i
		}
	}
	return maxRoom
}

func main() {
	fmt.Println(mostBooked(2, [][]int{{0, 10}, {1, 5}, {2, 7}, {3, 4}}))
	// Room 0: [0,10], [10,17]; Room 1: [1,5], [5,8] → Room 0 = 2 bookings
}
```

---

## Example 5: Employee Free Time

```go
package main

import (
	"container/heap"
	"fmt"
)

type Entry struct {
	start, end int
	empIdx     int
	ivIdx      int
}
type EntryHeap []Entry
func (h EntryHeap) Len() int           { return len(h) }
func (h EntryHeap) Less(i, j int) bool { return h[i].start < h[j].start }
func (h EntryHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *EntryHeap) Push(x interface{}) { *h = append(*h, x.(Entry)) }
func (h *EntryHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func employeeFreeTime(schedule [][][]int) [][]int {
	h := &EntryHeap{}
	for i, emp := range schedule {
		if len(emp) > 0 {
			heap.Push(h, Entry{emp[0][0], emp[0][1], i, 0})
		}
	}

	result := [][]int{}
	prevEnd := -1

	for h.Len() > 0 {
		e := heap.Pop(h).(Entry)

		if prevEnd != -1 && e.start > prevEnd {
			result = append(result, []int{prevEnd, e.start})
		}
		if e.end > prevEnd {
			prevEnd = e.end
		}

		// Push next interval from same employee
		nextIdx := e.ivIdx + 1
		if nextIdx < len(schedule[e.empIdx]) {
			next := schedule[e.empIdx][nextIdx]
			heap.Push(h, Entry{next[0], next[1], e.empIdx, nextIdx})
		}
	}
	return result
}

func main() {
	schedule := [][][]int{
		{{1, 2}, {5, 6}},
		{{1, 3}},
		{{4, 10}},
	}
	fmt.Println("Free time:", employeeFreeTime(schedule))
	// [[3,4]]
}
```

---

## Example 6: Maximum Meetings in One Room

```go
package main

import (
	"fmt"
	"sort"
)

func maxMeetingsOneRoom(meetings [][]int) int {
	// Greedy: pick by earliest end time
	sort.Slice(meetings, func(i, j int) bool {
		return meetings[i][1] < meetings[j][1]
	})

	count := 1
	end := meetings[0][1]

	for i := 1; i < len(meetings); i++ {
		if meetings[i][0] >= end {
			count++
			end = meetings[i][1]
		}
	}
	return count
}

func main() {
	meetings := [][]int{{1, 2}, {3, 4}, {0, 6}, {5, 7}, {8, 9}, {5, 9}}
	fmt.Println("Max meetings in one room:", maxMeetingsOneRoom(meetings))
	// [1,2],[3,4],[5,7],[8,9] → 4
}
```

---

## Example 7: Room Utilization Rate

```go
package main

import (
	"fmt"
	"sort"
)

func roomUtilization(meetings [][]int, totalTime int) float64 {
	// Merge overlapping meetings first
	sort.Slice(meetings, func(i, j int) bool {
		return meetings[i][0] < meetings[j][0]
	})

	merged := [][]int{meetings[0]}
	for i := 1; i < len(meetings); i++ {
		last := merged[len(merged)-1]
		if meetings[i][0] <= last[1] {
			if meetings[i][1] > last[1] {
				last[1] = meetings[i][1]
			}
		} else {
			merged = append(merged, meetings[i])
		}
	}

	busyTime := 0
	for _, m := range merged {
		busyTime += m[1] - m[0]
	}

	return float64(busyTime) / float64(totalTime) * 100
}

func main() {
	meetings := [][]int{{0, 5}, {3, 8}, {10, 15}}
	fmt.Printf("Utilization: %.1f%%\n", roomUtilization(meetings, 20))
	// Merged: [0,8],[10,15] → 8+5 = 13/20 = 65.0%
}
```

---

## Example 8: Interval Partitioning (Coloring)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type MH []int
func (h MH) Len() int           { return len(h) }
func (h MH) Less(i, j int) bool { return h[i] < h[j] }
func (h MH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func intervalPartition(intervals [][]int) (int, []int) {
	type Indexed struct {
		start, end, idx int
	}
	sorted := make([]Indexed, len(intervals))
	for i, iv := range intervals {
		sorted[i] = Indexed{iv[0], iv[1], i}
	}
	sort.Slice(sorted, func(i, j int) bool {
		return sorted[i].start < sorted[j].start
	})

	assignment := make([]int, len(intervals))
	h := &MH{}            // min-heap of available room IDs
	endHeap := &MH{}      // min-heap of (end time) — simplified
	roomEnds := map[int]int{} // room → end time
	nextRoom := 0

	for _, iv := range sorted {
		// Check if any room is free
		assigned := -1
		if endHeap.Len() > 0 && (*endHeap)[0] <= iv.start {
			freeEnd := heap.Pop(endHeap).(int)
			// Find room with this end time
			for r, e := range roomEnds {
				if e == freeEnd {
					assigned = r
					delete(roomEnds, r)
					break
				}
			}
		}

		if assigned == -1 {
			assigned = nextRoom
			nextRoom++
		}

		assignment[iv.idx] = assigned
		roomEnds[assigned] = iv.end
		heap.Push(endHeap, iv.end)
	}
	return nextRoom, assignment
}

func main() {
	intervals := [][]int{{0, 5}, {1, 3}, {4, 7}, {6, 9}}
	rooms, assign := intervalPartition(intervals)
	fmt.Println("Rooms needed:", rooms)
	fmt.Println("Assignment:", assign)
}
```

---

## Example 9: Minimum Platforms at a Station

```go
package main

import (
	"fmt"
	"sort"
)

func minPlatforms(arrivals, departures []int) int {
	sort.Ints(arrivals)
	sort.Ints(departures)

	plat, maxPlat := 0, 0
	i, j := 0, 0

	for i < len(arrivals) {
		if arrivals[i] <= departures[j] {
			plat++
			i++
		} else {
			plat--
			j++
		}
		if plat > maxPlat {
			maxPlat = plat
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

## Example 10: Meeting Room Model Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Meeting Rooms Model — Complete Guide ===")
	fmt.Println()

	problems := []struct{ variant, approach, complexity string }{
		{
			"Can attend all? (I)",
			"Sort by start, check adjacent overlaps",
			"O(n log n)",
		},
		{
			"Min rooms needed (II)",
			"Sort starts/ends separately, two pointers OR min-heap of end times",
			"O(n log n)",
		},
		{
			"Room assignment (III)",
			"Min-heap of (freeAt, roomId), sort meetings by start",
			"O(n log n)",
		},
		{
			"Max meetings in one room",
			"Greedy by earliest end time (activity selection)",
			"O(n log n)",
		},
		{
			"Employee free time",
			"Merge all schedules, find gaps",
			"O(n log n)",
		},
		{
			"Min platforms",
			"Sort arrivals/departures, sweep",
			"O(n log n)",
		},
		{
			"Room utilization",
			"Merge intervals, sum busy / total time",
			"O(n log n)",
		},
		{
			"Interval partitioning",
			"Graph coloring = min rooms (same as II)",
			"O(n log n)",
		},
	}

	for _, p := range problems {
		fmt.Printf("  %s\n", p.variant)
		fmt.Printf("    Approach: %s\n", p.approach)
		fmt.Printf("    Time: %s\n\n", p.complexity)
	}

	fmt.Println("Key insight: min rooms = max concurrent meetings")
	fmt.Println("All variants reduce to sweep line or heap processing")
}
```

---

## Key Takeaways

1. Meeting Rooms I: sort, check adjacent overlap — O(n log n)
2. Meeting Rooms II: min rooms = max concurrent = sweep line or min-heap
3. Room assignment: min-heap of (free_time, room_id)
4. Employee free time: merge all intervals, gaps = free time
5. All meeting room problems are O(n log n) — dominated by sorting

> **Phase 18 Complete!** Next up: Phase 19 — Trie →
