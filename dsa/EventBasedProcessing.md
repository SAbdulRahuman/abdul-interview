# Phase 18: Intervals — Event-Based Processing

## Overview

**Event-based processing** models problems as a timeline of discrete events (arrivals, departures, starts, ends) processed in chronological order. It generalizes the sweep line pattern by supporting multiple event types and complex state transitions.

| Component | Role |
|-----------|------|
| Event | Timestamped action (start, end, query) |
| Priority | Tie-breaking between simultaneous events |
| State | Running aggregation (count, sum, set) |
| Observer | Logic triggered on state changes |

---

## Example 1: Corporate Flight Bookings

```go
package main

import "fmt"

func corpFlightBookings(bookings [][]int, n int) []int {
	// Each booking [first, last, seats] → event-based
	diff := make([]int, n+2) // 1-indexed

	for _, b := range bookings {
		diff[b[0]] += b[2]
		diff[b[1]+1] -= b[2]
	}

	result := make([]int, n)
	sum := 0
	for i := 1; i <= n; i++ {
		sum += diff[i]
		result[i-1] = sum
	}
	return result
}

func main() {
	bookings := [][]int{{1, 2, 10}, {2, 3, 20}, {2, 5, 25}}
	fmt.Println(corpFlightBookings(bookings, 5))
	// [10, 55, 45, 25, 25]
}
```

---

## Example 2: Process Timeline Events

```go
package main

import (
	"fmt"
	"sort"
)

type Event struct {
	Time int
	Type string // "arrive", "depart", "query"
	ID   int
}

func processTimeline(events []Event) {
	sort.Slice(events, func(i, j int) bool {
		if events[i].Time == events[j].Time {
			// Process departures before arrivals before queries
			priority := map[string]int{"depart": 0, "arrive": 1, "query": 2}
			return priority[events[i].Type] < priority[events[j].Type]
		}
		return events[i].Time < events[j].Time
	})

	active := map[int]bool{}
	for _, e := range events {
		switch e.Type {
		case "arrive":
			active[e.ID] = true
			fmt.Printf("t=%d: Person %d arrives (active=%d)\n", e.Time, e.ID, len(active))
		case "depart":
			delete(active, e.ID)
			fmt.Printf("t=%d: Person %d departs (active=%d)\n", e.Time, e.ID, len(active))
		case "query":
			fmt.Printf("t=%d: Query → %d people active\n", e.Time, len(active))
		}
	}
}

func main() {
	events := []Event{
		{1, "arrive", 1}, {3, "arrive", 2}, {5, "depart", 1},
		{4, "query", 0}, {7, "depart", 2}, {6, "query", 0},
	}
	processTimeline(events)
}
```

---

## Example 3: Maximum Population Year

```go
package main

import "fmt"

func maximumPopulation(logs [][]int) int {
	diff := make([]int, 2051) // years 1950-2050

	for _, log := range logs {
		diff[log[0]]++
		diff[log[1]]--
	}

	maxPop, maxYear := 0, 0
	pop := 0
	for year := 1950; year <= 2050; year++ {
		pop += diff[year]
		if pop > maxPop {
			maxPop = pop
			maxYear = year
		}
	}
	return maxYear
}

func main() {
	logs := [][]int{{1993, 1999}, {2000, 2010}, {1950, 2050}}
	fmt.Println("Max population year:", maximumPopulation(logs)) // 2000 or other based on data

	logs2 := [][]int{{1950, 1961}, {1960, 1971}, {1970, 1981}}
	fmt.Println("Max population year:", maximumPopulation(logs2)) // 1960
}
```

---

## Example 4: Task Scheduler Events

```go
package main

import (
	"fmt"
	"sort"
)

type Task struct {
	ID       int
	Start    int
	End      int
	Priority int
}

func scheduleByPriority(tasks []Task) []string {
	type Event struct {
		time     int
		isStart  bool
		task     Task
	}

	events := []Event{}
	for _, t := range tasks {
		events = append(events, Event{t.Start, true, t})
		events = append(events, Event{t.End, false, t})
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i].time == events[j].time {
			return !events[i].isStart // ends before starts
		}
		return events[i].time < events[j].time
	})

	active := map[int]Task{}
	log := []string{}

	getHighest := func() (Task, bool) {
		best := Task{Priority: -1}
		for _, t := range active {
			if t.Priority > best.Priority {
				best = t
			}
		}
		return best, best.Priority >= 0
	}

	for _, e := range events {
		if e.isStart {
			active[e.task.ID] = e.task
		} else {
			delete(active, e.task.ID)
		}
		if t, ok := getHighest(); ok {
			log = append(log, fmt.Sprintf("t=%d: Task %d (pri=%d) running", e.time, t.ID, t.Priority))
		} else {
			log = append(log, fmt.Sprintf("t=%d: idle", e.time))
		}
	}
	return log
}

func main() {
	tasks := []Task{
		{1, 0, 5, 1}, {2, 2, 7, 3}, {3, 4, 6, 2},
	}
	for _, msg := range scheduleByPriority(tasks) {
		fmt.Println(msg)
	}
}
```

---

## Example 5: Online vs Offline Event Processing

```go
package main

import (
	"fmt"
	"sort"
)

// Offline: process all events sorted (we know all events upfront)
func offlineProcess(intervals [][]int, queries []int) []int {
	type Event struct {
		pos, typ, idx int
	}

	events := []Event{}
	for _, iv := range intervals {
		events = append(events, Event{iv[0], 0, -1})  // start
		events = append(events, Event{iv[1], 2, -1})  // end
	}
	for i, q := range queries {
		events = append(events, Event{q, 1, i}) // query
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i].pos == events[j].pos {
			return events[i].typ < events[j].typ
		}
		return events[i].pos < events[j].pos
	})

	result := make([]int, len(queries))
	active := 0
	for _, e := range events {
		switch e.typ {
		case 0: active++
		case 1: result[e.idx] = active
		case 2: active--
		}
	}
	return result
}

// Online: process events as they come (simulated with incremental updates)
type OnlineCounter struct {
	events [][]int // sorted events
	pos    int
	active int
}

func NewOnlineCounter(intervals [][]int) *OnlineCounter {
	events := [][]int{}
	for _, iv := range intervals {
		events = append(events, []int{iv[0], 1})
		events = append(events, []int{iv[1], -1})
	}
	sort.Slice(events, func(i, j int) bool {
		return events[i][0] < events[j][0]
	})
	return &OnlineCounter{events, 0, 0}
}

func (c *OnlineCounter) QueryAt(t int) int {
	for c.pos < len(c.events) && c.events[c.pos][0] <= t {
		c.active += c.events[c.pos][1]
		c.pos++
	}
	return c.active
}

func main() {
	intervals := [][]int{{1, 5}, {2, 6}, {4, 8}}
	queries := []int{3, 5, 7}

	fmt.Println("Offline:", offlineProcess(intervals, queries))

	oc := NewOnlineCounter(intervals)
	for _, q := range queries {
		fmt.Printf("Online at t=%d: %d\n", q, oc.QueryAt(q))
	}
}
```

---

## Example 6: Server Load Analysis

```go
package main

import (
	"fmt"
	"sort"
)

type Request struct {
	Start, End, Load int
}

func analyzeLoad(requests []Request) (int, int) {
	type Event struct {
		time, delta int
	}

	events := []Event{}
	for _, r := range requests {
		events = append(events, Event{r.Start, r.Load})
		events = append(events, Event{r.End, -r.Load})
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i].time == events[j].time {
			return events[i].delta < events[j].delta
		}
		return events[i].time < events[j].time
	})

	maxLoad, peakTime := 0, 0
	currentLoad := 0

	for _, e := range events {
		currentLoad += e.delta
		if currentLoad > maxLoad {
			maxLoad = currentLoad
			peakTime = e.time
		}
	}
	return maxLoad, peakTime
}

func main() {
	requests := []Request{
		{1, 5, 100}, {2, 4, 200}, {3, 6, 150},
	}
	load, time := analyzeLoad(requests)
	fmt.Printf("Peak load: %d at time %d\n", load, time)
	// At time 3: 100+200+150 = 450
}
```

---

## Example 7: Brightness of Lamps on a Street

```go
package main

import (
	"fmt"
	"sort"
)

func brightestPosition(lights [][]int) int {
	type Event struct {
		pos, delta int
	}

	events := []Event{}
	for _, l := range lights {
		pos, range_ := l[0], l[1]
		events = append(events, Event{pos - range_, 1})
		events = append(events, Event{pos + range_ + 1, -1})
	}

	sort.Slice(events, func(i, j int) bool {
		if events[i].pos == events[j].pos {
			return events[i].delta > events[j].delta // start before end
		}
		return events[i].pos < events[j].pos
	})

	maxBright, bestPos := 0, 0
	bright := 0

	for _, e := range events {
		bright += e.delta
		if bright > maxBright {
			maxBright = bright
			bestPos = e.pos
		}
	}
	return bestPos
}

func main() {
	lights := [][]int{{-3, 2}, {1, 2}, {3, 3}}
	fmt.Println("Brightest position:", brightestPosition(lights))
}
```

---

## Example 8: Process Logs to Find Busiest Period

```go
package main

import (
	"fmt"
	"sort"
)

type LogEntry struct {
	Time    int
	IsEnter bool
	Count   int
}

func busiestPeriod(logs []LogEntry) [2]int {
	sort.Slice(logs, func(i, j int) bool {
		if logs[i].Time == logs[j].Time {
			return !logs[i].IsEnter // exits first
		}
		return logs[i].Time < logs[j].Time
	})

	maxCount, curr := 0, 0
	busyStart := 0
	result := [2]int{}

	for i := 0; i < len(logs); i++ {
		if logs[i].IsEnter {
			curr += logs[i].Count
		} else {
			curr -= logs[i].Count
		}

		// Check at transitions (process all events at same time)
		if i == len(logs)-1 || logs[i].Time != logs[i+1].Time {
			if curr > maxCount {
				maxCount = curr
				busyStart = logs[i].Time
				if i+1 < len(logs) {
					result = [2]int{busyStart, logs[i+1].Time}
				} else {
					result = [2]int{busyStart, busyStart}
				}
			}
		}
	}
	return result
}

func main() {
	logs := []LogEntry{
		{1, true, 3}, {2, true, 5}, {4, false, 2},
		{5, true, 4}, {6, false, 3}, {7, false, 7},
	}
	period := busiestPeriod(logs)
	fmt.Printf("Busiest period: [%d, %d)\n", period[0], period[1])
}
```

---

## Example 9: Falling Squares (Height Tracking)

```go
package main

import "fmt"

func fallingSquares(positions [][]int) []int {
	type Interval struct {
		left, right, height int
	}

	active := []Interval{}
	result := make([]int, len(positions))
	maxH := 0

	for i, p := range positions {
		left, side := p[0], p[1]
		right := left + side

		// Find max height of any overlapping interval
		baseH := 0
		for _, a := range active {
			if a.left < right && left < a.right {
				if a.height > baseH {
					baseH = a.height
				}
			}
		}

		newH := baseH + side
		active = append(active, Interval{left, right, newH})

		if newH > maxH {
			maxH = newH
		}
		result[i] = maxH
	}
	return result
}

func main() {
	positions := [][]int{{1, 2}, {2, 3}, {6, 1}}
	fmt.Println(fallingSquares(positions)) // [2, 5, 5]
}
```

---

## Example 10: Event Processing Pattern Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Event-Based Processing Patterns ===")
	fmt.Println()

	patterns := []struct{ pattern, events, state, examples string }{
		{
			"Difference Array",
			"+val at start, -val at end+1",
			"Prefix sum",
			"Flight bookings, range additions",
		},
		{
			"Timeline Sweep",
			"Sorted start/end with type tag",
			"Active count / active set",
			"Meeting rooms, max overlap",
		},
		{
			"Multi-Type Events",
			"Different event types with priorities",
			"State machine per event type",
			"Task scheduling, server load",
		},
		{
			"Offline Queries",
			"Mix data events with query events",
			"Answer queries during sweep",
			"Count at point, range queries",
		},
		{
			"Online Processing",
			"Events arrive incrementally",
			"Monotonic pointer into sorted events",
			"Stream processing, sliding analysis",
		},
	}

	for _, p := range patterns {
		fmt.Printf("  %s\n", p.pattern)
		fmt.Printf("    Events: %s\n", p.events)
		fmt.Printf("    State: %s\n", p.state)
		fmt.Printf("    Examples: %s\n\n", p.examples)
	}
}
```

---

## Key Takeaways

1. Model problems as discrete events with timestamps and types
2. Sort events by time; use type priority for tie-breaking
3. Difference arrays handle range updates efficiently — O(n + q)
4. Offline processing sorts queries with data events; online processes incrementally
5. Event ordering matters: typically ends before starts at same time

> **Next up:** Line Sweep with Heap →
